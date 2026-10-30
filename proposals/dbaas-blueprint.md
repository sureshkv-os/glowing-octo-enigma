# A Plugin-Based Database Management Service for Self-Hosted Fleets

*Design proposal.* An engine-agnostic control plane that gives Aerospike, FoundationDB, and later engines a managed-service experience on VMs we own. Each engine is added as a declarative pack instead of new platform code.

## Contents

1. [Summary](#1-summary)
2. [How hyperscalers run managed databases](#2-how-hyperscalers-run-managed-databases)
3. [Design principles](#3-design-principles)
4. [Foundational constructs](#4-foundational-constructs)
5. [Reference architecture](#5-reference-architecture)
6. [The engine pack](#6-the-engine-pack-the-plugin-unit)
7. [Aerospike and FoundationDB constraints](#7-aerospike-and-foundationdb-constraints)
8. [Patching, redesigned](#8-patching-redesigned)
9. [VMs or Kubernetes](#9-vms-or-kubernetes)
10. [Build order](#10-build-order)
11. [Risks and unknowns](#11-risks-and-unknowns)
12. [Sources](#12-sources)

---

## 1. Summary

Managed database services are not built from configuration management plus IaC. They are built from a **control plane** that holds the desired state of every database, **durable workflows** that carry out long operations, **health-gated rollouts** that respect each engine's failure rules, and **images and agents** on the nodes. Ansible and Pulumi are good at the first provisioning of a substrate. They are weak at running a fleet, because they hold no live state, have no notion of a cluster's health, and execute from a person's laptop or a CI job.

The proposal has four parts:

1. **A small, engine-agnostic core** with eleven constructs plus a substrate adapter and node agent (section 4).
2. **An engine pack** as the plugin unit. About 80% of a pack is declarative (schema, topology, probes, operation graphs, alert mappings, compatibility matrix). The remaining 20% is small, sandboxed adapter code behind a fixed contract.
3. **Immutable images and node agents** in place of in-place Ansible runs, with rack-by-rack or zone-by-zone waves instead of node-by-node serial patching.
4. **A build order** that proves the abstraction by shipping Aerospike first, FoundationDB second, and only then opening the pack contract to others.

> **The biggest single lever on the patching problem.** Serial patching of a 25-node cluster is 25 sequential units of work. If the cluster is rack-aware with enough racks, the same cluster can be patched in as many waves as it has racks, because each rack can go down together without losing a copy of the data. That is a policy change in the rollout engine, not new technology, and it is the first thing the platform should encode.

---

## 2. How hyperscalers run managed databases

Their internals are not public in detail, so each claim below is marked by evidence level: **Documented** (first-party docs), **Secondary** (third-party or conference material), or **Inference** (my reasoning from the pattern; not confirmed). Searches for first-party material on host-level mechanics turned up little.

| Question | Answer | Evidence |
|---|---|---|
| Is there a control plane and a data plane? | Yes. AWS defines the control plane as the part that handles changes to resources and propagates them, and the data plane as what resources do day to day. Control planes have more moving parts and fail more often, while data planes carry the traffic customers care about. | Documented |
| What happens when the control plane fails? | Data planes are designed to be *statically stable*: they keep working on last known state, and recovery should not depend on control plane calls. | Documented |
| How is blast radius limited? | Cells. Each cell is a self-contained slice of the service. The control plane provisions cells and tells a router where to send traffic. A bad deployment touches one cell first. | Documented |
| How are minor upgrades sequenced? | For RDS Multi-AZ DB clusters, AWS upgrades readers one at a time, switches one reader to become the writer, then upgrades the old writer, with downtime of about 35 seconds. RDS Blue/Green builds a full copy (green), keeps it in sync, and switches over, typically in under a minute. | Documented |
| How does Azure SQL structure it? | An API and logic layer is separated from the SQL instances. Each database lives in a replica set on Service Fabric, a primary with two secondaries on separate machines. Service Fabric rolls out upgrades in waves with health policies and rolls back when they fail. | Secondary |
| Do they run Ansible? | No public evidence either way. The pattern points elsewhere: proprietary orchestration, pre-built host images, and an agent on each host. Configuration drift is avoided by replacing hosts rather than mutating them. | Inference |
| Is fleet state held in Terraform or Pulumi? | Very unlikely. Terraform and Pulumi state files describe infrastructure someone owns and applies. CloudFormation and ARM or Bicep are customer-facing front ends that call the service API. The service's own state lives in its own metadata stores, with workflow state tracked durably per operation. | Inference |

### What to take from them

- **State belongs in the platform, not in a plan file.** The desired and observed state of each cluster lives in a store the control plane owns. IaC tools are used only for the layer beneath (accounts, networks, cell scaffolding).
- **Operations are workflows, not scripts.** A patch is a durable, resumable, observable state machine with checkpoints, not a playbook run that you hope finishes.
- **Hosts are cattle where the engine allows it.** Replace from an image, then let the engine rebalance. Where data lives on local NVMe, replacement has a data-movement cost, so the platform must know when in-place patching with a reboot is the cheaper path.
- **The data plane never needs the control plane to keep serving.** Node agents act on local state, and an outage of the control plane pauses change but not traffic.

---

## 3. Design principles

1. **Declare, don't script.** Users state what they want (version, size, racks, policy). The platform converges.
2. **The engine knows its own rules.** Quorum, replication, and migration rules belong to the engine pack. The core never hard-codes them.
3. **Every change is gated by health.** No step proceeds until the engine says it is safe, and no failure proceeds silently.
4. **Operations are durable and idempotent.** Any step can be retried, paused, or taken over by a human.
5. **The core stays small.** A new engine should need a pack, not a core change.
6. **Control, not lock-in.** Anything the platform sets can be overridden by an authorised user, with the override recorded.

---

## 4. Foundational constructs

These are engine-agnostic. They exist once, and every pack uses them.

| Construct | Responsibility | Notes |
|---|---|---|
| Resource model | Typed objects: `Cluster`, `NodeGroup`, `Node`, `Operation`, `MaintenancePolicy`, `BackupPolicy`, `Cell`. Each has `spec` (desired) and `status` (observed). | Versioned API with a schema per engine. The shape Kubernetes made familiar, whether or not Kubernetes is the host. |
| Spec and status store | Source of truth for desired and observed state, with history. | Transactional store with an audit-grade change log. |
| Reconciler | Compares spec and status and starts operations to close the gap. Handles drift, repair, and scale. | Level-triggered, so a missed event is corrected on the next pass. |
| Workflow engine | Runs multi-step operations durably: patch, upgrade, replace, scale, restore. | Needs retries, timers, signals (pause, approve), and resume after a crash. A durable-execution engine fits. |
| Rollout engine | Plans waves, enforces disruption budgets, applies canary-first ordering, and halts or rolls back on gate failure. | Takes topology and limits from the pack. Owns the "how fast, how many at once" decision. |
| Health signals | Normalises probes and metrics into a small set of states: *healthy, degraded, unsafe-to-disturb, down*. | Reads Prometheus and engine probes. The pack defines the queries. |
| Remediation engine | Maps detected conditions to safe actions, with a trust level per action: observe, suggest, auto-with-limits, auto. | Starts in suggest mode, graduates per rule. |
| Backup and DR service | Schedules backups, verifies them by restore, and orchestrates cross-cluster failover and failback. | The pack supplies the engine driver. The core owns schedule, retention, and test restores. |
| Fleet inventory | Version, patch level, image, owner, cost, and compliance state for every node and cluster. | What Security queries instead of asking an SRE. |
| Policy, identity, audit | Who may do what, maintenance windows, version allow-lists, approval rules, and an immutable record of every action. | Policy as code. Overrides require a reason. |
| Self-service API | One API behind a CLI, a UI, and a Git-based path. | Application teams request and change clusters within policy. |
| Substrate adapter | Creates and replaces compute, disks, and network on Azure, AWS, or Kubernetes. | Thin interface. Pulumi or Terraform can sit behind it for substrate resources only. |
| Node agent | Small daemon on each node: runs local steps, reports health, keeps working if the control plane is away. | Replaces most Ansible-over-SSH usage. |

---

## 5. Reference architecture

```text
+---------------------------------------------------------------+      +-------------------------+
| CONTROL PLANE (one per environment)                           |      | ENGINE PACKS (plugins)  |
|                                                               |      |                         |
|  [API + policy] [Spec/status store] [Reconciler] [Inventory]  | <--> |  - aerospike            |
|  [Workflow eng] [Rollout engine]    [Remediation] [Backup/DR] |      |  - foundationdb         |
|                                                               |      |  - next engine          |
|  Desired state in, operations out. Never on the data path.    |      |  schema, topology,      |
+---------------------------------------------------------------+      |  probes, operations     |
        |  operations, desired state        ^  status, health, events  +-------------------------+
        v                                   |
+------------------------------------------------+   +------------------------------------------+
| CELL: AWS region                               |   | CELL: Azure region (same shape)          |
|  [Substrate adapter + image registry]          |   |                                          |
|  [Aerospike cluster]   [FoundationDB cluster]  |   | Each cell keeps serving on last known    |
|   racks A B C           zones 1 2 3            |   | state if the control plane is            |
|   node agents           node agents            |   | unreachable. Change pauses, serving      |
|   Prometheus exporter   Prometheus exporter    |   | does not. New region or cloud = new      |
+------------------------------------------------+   | cell, same packs.                        |
                                                     +------------------------------------------+
```

The control plane handles intent and orchestration. Packs describe each engine. Cells hold the clusters and keep serving without the control plane.

---

## 6. The engine pack: the plugin unit

A pack is a versioned bundle that tells the core everything engine-specific. The goal is that adding an engine means writing a pack and reviewing it, not changing the core.

| Section | Declares | Form |
|---|---|---|
| Spec schema | The fields a user may set, their defaults, limits, and which changes are safe live. | Declarative (JSON Schema) |
| Image recipe | How to build the node image: OS, engine binaries, exporters, agent, hardening. | Declarative plus build script |
| Config templates | Engine config files rendered from the spec. | Declarative templates |
| Topology model | The failure-domain unit (rack, zone), how it maps to cloud zones, and the maximum units that may be down at once for a given replication setting. | Declarative rules |
| Observation | Probes and metric queries that define healthy, degraded, and unsafe-to-disturb. | Declarative queries |
| Operation graphs | Ordered steps for patch, upgrade, replace, scale, and restore, built from core primitives. | Declarative DAG |
| Compatibility matrix | Allowed upgrade paths, minimum versions, client-library requirements. | Declarative table |
| Alert mappings | Condition to remediation action, with trust level. | Declarative rules |
| Backup driver | How to take, catalogue, and restore a backup for this engine. | Adapter code |
| Engine adapter | Commands the engine needs that no primitive covers: quiesce, exclude, set roster, read a custom statistic. | Adapter code behind a fixed gRPC contract |

### Core primitives that operation graphs are built from

`preflight`, `wait-until` (a probe condition with a timeout), `drain`, `set-param`, `stop`, `start`, `apply-image` or `patch-in-place`, `reboot`, `replace-node`, `verify`, `checkpoint`, and `approve`. Because every engine's patch is a different ordering of the same primitives plus a few engine adapter calls, most of a new pack is data.

### What a pack looks like

An abbreviated Aerospike example. Field names are illustrative, not a final schema.

```yaml
kind: EnginePack
name: aerospike
version: 0.1.0
topology:
  unit: rack
  map: cloud.zone
  disruption:
    maxUnitsDown: "1 rack"
    require: "replication_factor <= rack_count"
observe:
  safe_to_disturb:
    - asinfo: cluster-stable == true
    - metric: cluster_size == spec.nodes
    - metric: migrate_partitions_remaining == 0
operations:
  patch-node:
    steps:
      - preflight
      - adapter: set-migrate-fill-delay
      - adapter: quiesce
      - stop
      - patch-in-place
      - reboot
      - start
      - wait-until: cluster_size == spec.nodes
      - wait-until: safe_to_disturb
      - verify
upgrade_matrix: ./upgrade-paths.yaml
alerts: ./alert-rules.yaml
adapter: grpc://aerospike-adapter
```

### Keeping packs safe

- Adapter code runs sandboxed (a separate process or WASM module) with only the permissions the pack declares.
- Packs are signed, versioned, and pinned per cluster, so upgrading a pack is itself a gated change.
- A pack test harness runs every operation against a throwaway cluster before a pack is accepted.

---

## 7. Aerospike and FoundationDB constraints

These live inside each pack, not in the core. The table shows the same concerns resolved differently, which is the reason for the pack boundary. Items marked **(Verify)** reflect general FoundationDB knowledge that I did not confirm against current documentation.

| Concern | Aerospike | FoundationDB | Lives in |
|---|---|---|---|
| Failure-domain unit | Rack (rack-aware clusters). With replication factor at most the rack count, each rack can hold a full copy, which is what allows a whole rack to be taken down together. | Fault domain (zone, machine) as configured in locality. Redundancy mode (for example `triple`) sets how many can fail. | Topology model |
| Before stopping a node | Quiesce the node so it hands off master ownership at the next rebalance. Set `migrate-fill-delay` on every node above the expected maintenance time to avoid needless data movement. | For permanent removal, exclude the process so data moves off first. For a restart, no exclusion is needed if redundancy allows. **(Verify)** | Adapter |
| What "safe for the next node" means | Cluster key uniform and cluster size as expected, or the cluster-stable check on newer versions. Migrations must complete first if any namespace is in-memory only, and always for replication factor 1. If every namespace is persisted, the docs say the next node can proceed without losing data. | Database available, no data movement in flight beyond a threshold, and replication healthy as reported by cluster status. **(Verify)** | Observation |
| Strong consistency mode | Needs a roster. Dead or unavailable partitions must be resolved by a person before restarts continue. SC cannot be switched on or off for a running cluster, and all racks must share identical SC namespaces. | Not applicable. Transactions are always serialisable. | Operation graph (gate), spec schema |
| Version upgrade compatibility | Rolling, node by node, within documented upgrade paths. The Kubernetes operator also supports batch upgrades within a rack. | Patch releases of the same minor version are protocol-compatible. Different minor versions are not, so all server processes must move together, and clients need a compatible library. | Compatibility matrix |
| Client coordination | Client library compatibility with server version. | Hard requirement. Clients need a matching library before the server cuts over. The platform should record which applications use which client versions. | Compatibility matrix, inventory |
| Special roles | Namespaces, storage engine per namespace (memory, device), rack assignment. | Process classes (storage, log, stateless), and coordinators, which have their own safe-change rules. **(Verify)** | Spec schema, operation graph |
| Local NVMe | Persisted data survives an in-place reboot. Replacing the VM loses the local disk and forces full re-replication, so patch-in-place plus reboot is usually cheaper. **(Verify per instance type)** | Same principle. Replacing a storage process means redistribution through the cluster. | Operation graph (choose in-place or replace) |
| Replication and DR | XDR is asynchronous replication between clusters, scoped by namespace, set, or bin. Restore into an active-active XDR setup needs XDR disabled first. | Native cross-region configuration and backup agent, using the engine's own tooling. **(Verify)** | Backup driver |
| Existing operator | Aerospike Kubernetes Operator, Kubernetes only. | FoundationDB Kubernetes operator, Kubernetes only, one FDB version per operator image. | Substrate choice (section 9) |

> **How confidence was marked.** Aerospike rows come from Aerospike's published upgrade, quiesce, strong-consistency, and operator documentation. FoundationDB protocol and client rules come from the FoundationDB upgrade wiki. The first engineering task for each pack is to turn every Verify item into a tested step.

---

## 8. Patching, redesigned

Units below are abstract: one unit is the time to patch one node and reach a safe state. Actual times depend on your migration speed, which still needs to be measured.

Example: a 25-node cluster across 5 racks of 5 nodes.

```text
Serial (today), 25 units:
[#][#][#][#][#][#][#][#][#][#][#][#][#][#][#][#][#][#][#][#][#][#][#][#][#]

Five rack waves, 5 units (illustrative):
[#####][#####][#####][#####][#####]
```

The saving depends on racks being balanced and replication factor fitting the rack count. Each wave still waits for its health gate before the next starts.

### The rollout, step by step

1. **Plan.** The rollout engine reads the cluster's topology and the pack's disruption rules, and builds waves. A cluster that cannot safely lose a whole rack falls back to smaller waves, down to one node.
2. **Canary.** A low-risk cluster of the same engine and version goes first. The fleet waits on its result.
3. **Per wave.** Preflight gate, engine-specific drain (for example quiesce), patch (in place or from image), reboot, rejoin, safe-to-disturb gate.
4. **Stop rules.** A failed gate pauses the rollout and pages a human with the full state. Nothing continues on its own after a failure.
5. **Record.** Inventory is updated per node. Security's question, "is every node patched?", becomes a query.

### In place or replace

Choose per engine and per storage type. With local NVMe, patching in place and rebooting keeps the data, so the engine avoids re-replicating everything. Replace-from-image gives clean hosts and no drift but costs a full data copy each time. The platform should support both, default to the cheaper one per pack, and use replace for base-image changes and for nodes that fail a health check after patching.

---

## 9. VMs or Kubernetes

Treat these as two separate decisions: where the *control plane* runs, and where the *data plane* runs.

| Option | What you get | Cost |
|---|---|---|
| Data on VMs, custom control plane | Full control, no change for the database teams, local NVMe used as today. Works the same on Azure and AWS through the substrate adapter. | You build orchestration, node agent, and the engine adapters. The largest build. |
| Data on VMs, Kubernetes API as control plane | Use custom resources for the spec and status model, and controllers for reconciliation, in a small management cluster. Data stays on VMs, as Cluster API does for machines. You get API machinery, RBAC, and tooling for free. | Operational weight of a management cluster. Long operations still need a workflow engine alongside controllers. |
| Data on Kubernetes, vendor operators | Aerospike and FoundationDB both have Kubernetes operators that handle rolling logic. Fastest path if the engines run well on your Kubernetes nodes with local NVMe. | Operators cover one engine each and have different models, which fights the goal of a single plugin ecosystem. Aerospike's management tooling cost may remain. |

**Recommendation.** Design the core so the substrate is an adapter, as shown in the architecture. Start with VMs, since that is where you are. Keep Kubernetes as a second substrate, and use the vendor operators as reference material for the packs, because they encode the engines' rules. Choose between a custom state store and a Kubernetes-API-based one early, because the resource model is the hardest piece to change later.

---

## 10. Build order

1. **Baseline and quick wins.** Measure migration time per node, patch time, and SRE hours. Put rack-wave and canary logic around the current Ansible steps. This gives patching relief while the platform is built.
2. **Core skeleton with Aerospike.** Resource model, store, workflow engine, rollout engine, node agent, and the first pack. Cover create, patch, upgrade, replace, and scale. Retire the 30-minute manual configuration step by generating it from a spec.
3. **Health, inventory, and remediation in suggest mode.** Gate everything on probes. Publish fleet patch compliance. Auto-remediation begins as suggestions, and each rule earns automation.
4. **FoundationDB pack.** This is the test of the abstraction. Every place the core needed a change is a defect in the contract. Fix the contract, then freeze it.
5. **Backup, DR, and self-service.** Verified restores, cross-cluster failover drills, and a request path for application teams.
6. **Open the plugin interface.** Pack authoring guide, test harness, and signing. Add a third engine through a pack only.

---

## 11. Risks and unknowns

- **Platform sprawl.** A custom control plane is a product. Keep the core small, adopt a workflow engine instead of writing one, and measure SRE hours against the build.
- **Pack contract too thin or too wide.** Two engines is a small sample. Expect to revise the contract once, after FoundationDB.
- **Automation can amplify mistakes.** Hard stop on failed gates, disruption budgets enforced centrally, and trust levels per remediation rule.
- **Data movement time.** It sets the floor for patch time. There is no figure yet for your migration speed.
- **Unverified engine facts.** Items marked Verify must be tested against your versions.
- **Aerospike licensing.** The management software cost is the motivation, but vendor support terms may constrain what you can automate. Check before committing.
- **Public hyperscaler detail is thin.** The Ansible and IaC answers in section 2 are inference. A conversation with someone who has worked on a managed service would test them.

---

## 12. Sources

- [AWS: Control plane and data plane (cell-based architecture)](https://docs.aws.amazon.com/wellarchitected/latest/reducing-scope-of-impact-with-cell-based-architecture/control-plane-and-data-plane.html)
- [AWS: Static stability, Fault Isolation Boundaries](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/static-stability.html)
- [AWS: Upgrades of the RDS for MySQL DB engine](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_UpgradeDBInstance.MySQL.html)
- [AWS: Overview of RDS Blue/Green Deployments](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-overview.html)
- [SQLBits: How We Build a SQL Server PaaS at Scale](https://sqlbits.com/sessions/event2026/Architecting_the_Cloud_How_We_Build_a_SQL_Server_PaaS_at_Scale) (session abstract)
- [Petri: Is Azure SQL Highly Available?](https://petri.com/?p=89864) (secondary source)
- [Microsoft: Service Fabric managed cluster upgrades](https://learn.microsoft.com/uk-ua/azure/service-fabric/how-to-managed-cluster-upgrades)
- [Aerospike: Upgrading Aerospike](https://aerospike.com/docs/server/operations/upgrade/aerospike)
- [Aerospike: Quiesce node](https://aerospike.com/docs/database/manage/cluster/quiesce-node/)
- [Aerospike Kubernetes Operator: Strong consistency](https://aerospike.com/docs/kubernetes/4.3.0/manage/configure/strong-consistency)
- [Aerospike Kubernetes Operator 2.4.0 release notes](https://aerospike.com/docs/cloud/kubernetes/operator/2.4.0/release-notes)
- [Aerospike: Cross-Datacenter Replication (XDR) architecture](https://aerospike.com/docs/architecture/xdr)
- [FoundationDB: Upgrading FoundationDB](https://GitHub.com/apple/foundationdb/wiki/Upgrading-FoundationDB)
- [FoundationDB Kubernetes operator: compatibility (v1.38.0)](https://github.com/FoundationDB/fdb-kubernetes-operator/blob/v1.38.0/docs/compatibility.md)
