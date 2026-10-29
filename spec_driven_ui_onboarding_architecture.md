# Technical Design Document: Spec-Driven UI Architecture for BYOA Multi-Cloud Database Onboarding

**Author:** Architecture & Platform Engineering  
**Status:** Approved  
**Version:** 1.1.0  
**Target Standard:** OpenSpec / JSON Schema (Draft 2020-12)

---

## 1. Executive Summary

This document specifies the architecture and implementation design for a **Spec-Driven User Interface (UI)** powering a customer onboarding wizard for a multi-cloud, multi-database provisioning platform.

The system implements a **Bring-Your-Own-Account (BYOA)** operational model across major hyperscalers (AWS, Azure, GCP) and heterogeneous database engines:
- Document Stores (e.g., **MongoDB Enterprise**)
- Distributed Transactional Key-Value Stores (e.g., **Apple FoundationDB**)
- Real-Time Analytical Columnar Stores (e.g., **ClickHouse Cloud/Enterprise**)

Instead of maintaining rigid, hardcoded frontend routing and form trees, the entire onboarding lifecycle is governed by machine-readable specifications (OpenSpec/JSON Schema) parsed dynamically by a decoupled rendering engine.

---

## 2. Core Architectural Principles

```
┌────────────────────────────────────────────────────────┐
│                   Step 1: Global Spec                  │
│       Select Cloud Provider & Database Engine          │
└──────────────────────────┬─────────────────────────────┘
                           │ Resolves
                           ▼
┌────────────────────────────────────────────────────────┐
│             Step 2 & 3: Dynamic Domain Spec            │
│   Merged Provider Spec (AWS/GCP/Azure) + Engine Spec  │
│  (MongoDB / FoundationDB / ClickHouse Topology & Tier) │
└────────────────────────────────────────────────────────┘
```

1. **Decoupled Contracts & Interpreters:** The user interface layout, validation boundaries, cross-field dependencies, and step transitions are declared purely as data schemas. The client acts as a stateless, component-registered interpreter.
2. **Two-Tier Schema Resolution:**
   * **Tier 1 (Global Spec):** Orchestrates multi-step progression, layout shells, and top-level selectors (Cloud Provider $\times$ Database Engine).
   * **Tier 2 (Dynamic Domain Spec):** Composable schema fragments resolved at runtime based on Tier 1 selections, dynamically loading cloud-specific identity fields and engine topology forms (e.g., ClickHouse Shard/Replica layouts and S3/GCS tiered storage paths).
3. **Asynchronous Pre-Flight Verification:** High-latency BYOA actions (cross-account IAM role assumption, VPC quota validation, regional subnet discovery, S3/GCS bucket write tests) are decoupled into declared asynchronous hooks executed before step transitions.
4. **Zero-Trust BYOA Boundaries:** The client interface strictly enforces ephemeral permission exchange (e.g., AWS IAM AssumeRole with external IDs, GCP Workload Identity Federation) rather than storing long-lived, high-privilege credentials.

---

## 3. Recommended Technology Stack

| Layer | Technology | Selection Justification |
| :--- | :--- | :--- |
| **Frontend Framework** | **React / Next.js (TypeScript)** | Provides server/client component boundaries, strong typing for spec parsing, and robust ecosystem tooling. |
| **Form & State Engine** | **React Hook Form + Zod** | Uncontrolled input performance; dynamically compiles JSON Schema constraints into runtime Zod validators. |
| **Workflow State Machine** | **XState (v5)** | Deterministic multi-step wizard state machine handling async validation guards and provisioning states. |
| **Component System** | **Tailwind CSS + Radix UI / Shadcn UI** | Headless accessible UI primitives bound seamlessly to spec identifier tokens. |
| **Spec Definition** | **OpenSpec / JSON Schema (2020-12)** | Open, extensible standard supporting conditional branches (`if/then/else`, `discriminator`, `anyOf`). |
| **Backend Orchestration** | **Temporal.io (Go or Node.js)** | Durable execution engine handling long-running, multi-step cloud infrastructure provisioning workflows. |

---

## 4. End-to-End Onboarding Workflow

The wizard progresses through four deterministic phases:

```
┌─────────────────────────┐     ┌─────────────────────────┐
│ 1. Provider & Engine    │ ──> │ 2. Cloud Credentials    │
│    - Select Cloud       │     │    - Cross-Account Role │
│    - Select DB Engine   │     │    - Async Quota Check  │
└─────────────────────────┘     └────────────┬────────────┘
                                             │
┌─────────────────────────┐     ┌────────────▼────────────┐
│ 4. Review & Launch      │ <── │ 3. Database Topology    │
│    - Aggregate Preview  │     │    - Node Configurations│
│    - Submit to Temporal │     │    - Storage & Backups  │
└─────────────────────────┘     └─────────────────────────┘
```

1. **Step 1: Provider & Engine Selection:** Capture target cloud (AWS, Azure, GCP) and database type (MongoDB, FoundationDB, ClickHouse).
2. **Step 2: BYOA Account & Networking:** Renders provider-specific auth schemas (e.g., AWS Role ARN). Dispatches async pre-flight probes to discover customer VPCs, subnets, and cloud object storage endpoints.
3. **Step 3: Database Topology & Sizing:** Renders engine-specific configuration fields:
   * *MongoDB:* Replica sets vs. sharded clusters, compute classes.
   * *FoundationDB:* Redundancy modes (triple, double), coordinator allocations, Redwood storage engine.
   * *ClickHouse:* Shard/replica topology matrix, ClickHouse Keeper quorum count, tiered S3/GCS cold disk storage, and vector/vector-search cache allocation.
4. **Step 4: Review & Provisioning:** Aggregates validated payloads, performs an end-to-end dry-run, and submits the execution payload to the backend control plane.

---

## 5. Formal OpenSpec Specification

Below is the production-grade schema definition for the onboarding wizard, featuring explicit support for **ClickHouse** alongside MongoDB and FoundationDB.

```json
{
  "$schema": "https://openspec.dev/v1/spec.json",
  "version": "1.1.0",
  "metadata": {
    "title": "Database BYOA Onboarding Wizard",
    "description": "Dynamic provisioning spec for multi-cloud database deployment including ClickHouse"
  },
  "workflow": {
    "initialStep": "provider_and_engine",
    "steps": [
      {
        "id": "provider_and_engine",
        "title": "Select Infrastructure & Engine",
        "layout": "single_column",
        "fields": [
          {
            "id": "cloud_provider",
            "type": "select",
            "label": "Cloud Provider",
            "required": true,
            "options": [
              { "label": "Amazon Web Services (AWS)", "value": "aws" },
              { "label": "Google Cloud Platform (GCP)", "value": "gcp" },
              { "label": "Microsoft Azure", "value": "azure" }
            ]
          },
          {
            "id": "database_engine",
            "type": "select",
            "label": "Database Engine",
            "required": true,
            "options": [
              { "label": "ClickHouse Analytical Database", "value": "clickhouse" },
              { "label": "MongoDB Enterprise", "value": "mongodb" },
              { "label": "Apple FoundationDB", "value": "foundationdb" }
            ]
          }
        ],
        "next": "cloud_account_credentials"
      },
      {
        "id": "cloud_account_credentials",
        "title": "BYOA Cloud Credentials",
        "description": "Grant our control plane permission to provision compute, networking, and object storage in your account.",
        "conditionalForm": {
          "discriminateBy": "cloud_provider",
          "variants": {
            "aws": {
              "fields": [
                {
                  "id": "aws_account_id",
                  "type": "text",
                  "label": "AWS Account ID",
                  "placeholder": "123456789012",
                  "validation": {
                    "pattern": "^[0-9]{12}$",
                    "message": "Must be a 12-digit AWS Account ID"
                  },
                  "required": true
                },
                {
                  "id": "aws_role_arn",
                  "type": "text",
                  "label": "Cross-Account IAM Role ARN",
                  "placeholder": "arn:aws:iam::123456789012:role/DBaaSOrchestratorRole",
                  "validation": {
                    "pattern": "^arn:aws:iam::[0-9]{12}:role/.+$",
                    "message": "Must be a valid IAM Role ARN"
                  },
                  "required": true
                },
                {
                  "id": "aws_region",
                  "type": "select",
                  "label": "Target Region",
                  "required": true,
                  "options": [
                    { "label": "US East (N. Virginia) [us-east-1]", "value": "us-east-1" },
                    { "label": "US West (Oregon) [us-west-2]", "value": "us-west-2" },
                    { "label": "Europe (Frankfurt) [eu-central-1]", "value": "eu-central-1" }
                  ]
                }
              ],
              "asyncActions": [
                {
                  "trigger": "onBlur",
                  "targetField": "aws_role_arn",
                  "endpoint": "/api/v1/cloud/validate-role",
                  "method": "POST",
                  "payload": {
                    "provider": "aws",
                    "accountId": "{{aws_account_id}}",
                    "roleArn": "{{aws_role_arn}}",
                    "region": "{{aws_region}}"
                  },
                  "onSuccess": {
                    "populateFieldOptions": {
                      "targetField": "selected_vpc_id",
                      "sourceKey": "vpcs"
                    }
                  }
                }
              ]
            },
            "gcp": {
              "fields": [
                {
                  "id": "gcp_project_id",
                  "type": "text",
                  "label": "GCP Project ID",
                  "required": true
                },
                {
                  "id": "gcp_service_account_key",
                  "type": "file_upload",
                  "label": "Service Account JSON Key",
                  "accept": ".json",
                  "required": true
                }
              ]
            }
          }
        },
        "next": "engine_configuration"
      },
      {
        "id": "engine_configuration",
        "title": "Database Topology & Storage Sizing",
        "conditionalForm": {
          "discriminateBy": "database_engine",
          "variants": {
            "clickhouse": {
              "fields": [
                {
                  "id": "clickhouse_version",
                  "type": "select",
                  "label": "ClickHouse Release Version",
                  "defaultValue": "24.3-lts",
                  "options": [
                    { "label": "24.3 LTS (Production Stable)", "value": "24.3-lts" },
                    { "label": "23.8 LTS (Previous Stable)", "value": "23.8-lts" },
                    { "label": "24.8 Current", "value": "24.8" }
                  ]
                },
                {
                  "id": "clickhouse_cluster_topology",
                  "type": "radio_group",
                  "label": "Cluster Architecture",
                  "defaultValue": "replicated_shards",
                  "options": [
                    { "label": "Single Node (Non-HA / Analytics Lab)", "value": "single_node" },
                    { "label": "Replicated Cluster (High Availability)", "value": "replicated_shards" }
                  ]
                },
                {
                  "id": "clickhouse_shard_count",
                  "type": "number",
                  "label": "Number of Shards (Horizontal Scale)",
                  "defaultValue": 2,
                  "validation": { "min": 1, "max": 64 }
                },
                {
                  "id": "clickhouse_replicas_per_shard",
                  "type": "number",
                  "label": "Replicas per Shard",
                  "defaultValue": 2,
                  "validation": { "min": 1, "max": 4 }
                },
                {
                  "id": "clickhouse_keeper_nodes",
                  "type": "select",
                  "label": "ClickHouse Keeper Quorum",
                  "defaultValue": "3",
                  "options": [
                    { "label": "Colocated (Shared with Data Nodes)", "value": "0" },
                    { "label": "Dedicated 3-Node Quorum (Production HA)", "value": "3" },
                    { "label": "Dedicated 5-Node Quorum (Cross-AZ HA)", "value": "5" }
                  ]
                },
                {
                  "id": "clickhouse_storage_tiering",
                  "type": "select",
                  "label": "Storage Engine & Tiering",
                  "defaultValue": "tiered_s3",
                  "options": [
                    { "label": "Local NVMe SSD Only", "value": "local_nvme" },
                    { "label": "Tiered: Hot NVMe + Cloud Object Storage (S3/GCS Cold)", "value": "tiered_s3" },
                    { "label": "Zero-Disk: Direct S3/GCS Object Storage Cache", "value": "zero_disk" }
                  ]
                },
                {
                  "id": "clickhouse_cold_bucket_name",
                  "type": "text",
                  "label": "Target Object Storage Bucket Name (BYOA)",
                  "placeholder": "my-corp-clickhouse-cold-data",
                  "required": true
                }
              ]
            },
            "mongodb": {
              "fields": [
                {
                  "id": "mongo_topology",
                  "type": "radio_group",
                  "label": "Topology Mode",
                  "defaultValue": "replica_set",
                  "options": [
                    { "label": "3-Node Replica Set", "value": "replica_set" },
                    { "label": "Sharded Cluster", "value": "sharded" }
                  ]
                },
                {
                  "id": "mongo_version",
                  "type": "select",
                  "label": "MongoDB Version",
                  "defaultValue": "7.0",
                  "options": [
                    { "label": "7.0 (Current Stable)", "value": "7.0" },
                    { "label": "6.0", "value": "6.0" }
                  ]
                },
                {
                  "id": "mongo_instance_size",
                  "type": "select",
                  "label": "Compute Size",
                  "options": [
                    { "label": "db.m6g.large (2 vCPU, 8 GB RAM)", "value": "db.m6g.large" },
                    { "label": "db.m6g.xlarge (4 vCPU, 16 GB RAM)", "value": "db.m6g.xlarge" }
                  ]
                }
              ]
            },
            "foundationdb": {
              "fields": [
                {
                  "id": "fdb_redundancy_mode",
                  "type": "select",
                  "label": "Redundancy Mode",
                  "defaultValue": "triple",
                  "options": [
                    { "label": "Single (Development Only)", "value": "single" },
                    { "label": "Double", "value": "double" },
                    { "label": "Triple (Production Standard)", "value": "triple" }
                  ]
                },
                {
                  "id": "fdb_coordinator_count",
                  "type": "number",
                  "label": "Coordinators Count",
                  "defaultValue": 3,
                  "validation": { "min": 1, "max": 9 }
                },
                {
                  "id": "fdb_storage_engine",
                  "type": "select",
                  "label": "Storage Engine",
                  "defaultValue": "ssd-redwood-1",
                  "options": [
                    { "label": "Redwood (B-tree SSD)", "value": "ssd-redwood-1" },
                    { "label": "Memory", "value": "memory" }
                  ]
                }
              ]
            }
          }
        },
        "next": "review_and_launch"
      },
      {
        "id": "review_and_launch",
        "title": "Review Configuration & Provision",
        "layout": "summary_card",
        "actions": [
          {
            "id": "launch_cluster",
            "type": "submit",
            "label": "Provision Cluster",
            "endpoint": "/api/v1/clusters/provision",
            "method": "POST"
          }
        ]
      }
    ]
  }
}
```

---

## 6. Client Implementation Blueprint

### 6.1 Component Registry (`registry/componentRegistry.tsx`)

The component registry decouples schema field types from the concrete rendering layer, including numeric steppers for ClickHouse shard/replica counts.

```tsx
import React from "react";
import { Input } from "@/components/ui/input";
import { Select } from "@/components/ui/select";
import { RadioGroup } from "@/components/ui/radio-group";

export interface FieldProps {
  field: {
    id: string;
    label: string;
    placeholder?: string;
    defaultValue?: any;
    validation?: { min?: number; max?: number; pattern?: string };
    options?: Array<{ label: string; value: string | number }>;
  };
  value: any;
  onChange: (val: any) => void;
  error?: string;
}

export const ComponentRegistry: Record<string, React.FC<FieldProps>> = {
  text: ({ field, value, onChange }) => (
    <Input
      id={field.id}
      placeholder={field.placeholder}
      value={value || ""}
      onChange={(e) => onChange(e.target.value)}
    />
  ),
  number: ({ field, value, onChange }) => (
    <Input
      id={field.id}
      type="number"
      min={field.validation?.min}
      max={field.validation?.max}
      value={value ?? field.defaultValue ?? ""}
      onChange={(e) => onChange(Number(e.target.value))}
    />
  ),
  select: ({ field, value, onChange }) => (
    <Select
      value={String(value ?? field.defaultValue ?? "")}
      options={field.options || []}
      onValueChange={(val) => onChange(val)}
    />
  ),
  radio_group: ({ field, value, onChange }) => (
    <RadioGroup
      value={String(value ?? field.defaultValue ?? "")}
      options={field.options || []}
      onValueChange={(val) => onChange(val)}
    />
  )
};
```

### 6.2 Dynamic Step Interpreter (`engine/StepRenderer.tsx`)

Evaluates active step metadata, dynamically calculates derived topologies (e.g., ClickHouse total nodes = $\text{shards} \times \text{replicas} + \text{keeper nodes}$), and manages input changes.

```tsx
import React from "react";
import { ComponentRegistry } from "@/registry/componentRegistry";

interface StepRendererProps {
  stepSpec: any;
  formData: Record<string, any>;
  onFieldChange: (fieldId: string, value: any) => void;
}

export const StepRenderer: React.FC<StepRendererProps> = ({
  stepSpec,
  formData,
  onFieldChange,
}) => {
  let fieldsToRender = [...(stepSpec.fields || [])];

  // Resolve conditional variants (e.g., AWS vs GCP, ClickHouse vs Mongo vs FoundationDB)
  if (stepSpec.conditionalForm) {
    const selectorKey = stepSpec.conditionalForm.discriminateBy;
    const selectedVariantKey = formData[selectorKey];

    if (selectedVariantKey && stepSpec.conditionalForm.variants[selectedVariantKey]) {
      const variant = stepSpec.conditionalForm.variants[selectedVariantKey];
      fieldsToRender = [...fieldsToRender, ...(variant.fields || [])];
    }
  }

  // Derived ClickHouse node count indicator
  const isClickhouse = formData["database_engine"] === "clickhouse";
  const totalClickhouseComputeNodes = isClickhouse
    ? (Number(formData["clickhouse_shard_count"] || 2) * Number(formData["clickhouse_replicas_per_shard"] || 2)) +
      Number(formData["clickhouse_keeper_nodes"] || 3)
    : 0;

  return (
    <div className="space-y-6">
      <div className="border-b pb-3">
        <h2 className="text-xl font-semibold tracking-tight">{stepSpec.title}</h2>
        {stepSpec.description && (
          <p className="text-sm text-muted-foreground mt-1">{stepSpec.description}</p>
        )}
      </div>

      {isClickhouse && totalClickhouseComputeNodes > 0 && (
        <div className="p-3 bg-muted/50 rounded-md border text-sm text-muted-foreground">
          Calculated Total Infrastructure Footprint:{" "}
          <span className="font-semibold text-foreground">
            {totalClickhouseComputeNodes} VMs / Instances
          </span>{" "}
          ({formData["clickhouse_shard_count"] || 2} shards × {formData["clickhouse_replicas_per_shard"] || 2} replicas + {formData["clickhouse_keeper_nodes"] || 3} ClickHouse Keepers)
        </div>
      )}

      <div className="space-y-4">
        {fieldsToRender.map((field) => {
          const Component = ComponentRegistry[field.type];
          if (!Component) {
            console.warn(`No registered component for type: ${field.type}`);
            return null;
          }

          return (
            <div key={field.id} className="flex flex-col space-y-1.5">
              <label htmlFor={field.id} className="text-sm font-medium">
                {field.label} {field.required && <span className="text-red-500">*</span>}
              </label>
              <Component
                field={field}
                value={formData[field.id] ?? field.defaultValue}
                onChange={(val: any) => onFieldChange(field.id, val)}
              />
            </div>
          );
        })}
      </div>
    </div>
  );
};
```

---

## 7. Security, Isolation & Validation Framework

### 7.1 Cross-Account Trust Policy & Permissions

To run BYOA deployments, customer accounts must delegate scoped access. For **ClickHouse** deployments utilizing S3-backed storage disks:

1. **AWS Identity & S3 Storage Delegation:**
   - In addition to standard compute (`ec2:*`) permissions, the cross-account role requires read/write delegation to the customer's cold data S3 bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowControlPlaneAssumeRole",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::CONTROL_PLANE_ACCOUNT_ID:root" },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": { "sts:ExternalId": "TENANT_SPECIFIC_EXTERNAL_ID" }
      }
    },
    {
      "Sid": "AllowClickHouseS3Tiering",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::CUSTOMER_CLICKHOUSE_BUCKET",
        "arn:aws:s3:::CUSTOMER_CLICKHOUSE_BUCKET/*"
      ]
    }
  ]
}
```

2. **GCP Identity:** Use Workload Identity Federation mapped to the control plane's OIDC issuer, granting `roles/storage.objectAdmin` on the designated Google Cloud Storage cold bucket.

### 7.2 Pre-Flight Async Probes for ClickHouse

Before proceeding past the credentials & topology steps, the UI triggers async background validation endpoints:
- **Object Storage Bucket Accessibility:** Tests whether the provided cold storage bucket exists and allows read/write/multipart uploads via the assumed role.
- **Compute Quota Validation:** Verifies the customer's cloud account has sufficient instance capacity for the combined shard count ($\text{shards} \times \text{replicas}$) plus dedicated ClickHouse Keeper nodes.
- **VPC Subnet CIDR Availability:** Checks whether subnets have at least 16 free private IP addresses to prevent IP exhaustion during rolling cluster updates.

---

## 8. Rollout & Extensibility Roadmap

1. **Phase 1: Spec Engine Stabilization:** Validate registry parsing, JSON Schema dynamic conditional variants, and basic AWS/MongoDB workflows.
2. **Phase 2: ClickHouse & FoundationDB General Availability:** Deploy production-ready specs for ClickHouse (with automated Keeper quorum clustering and S3 cold storage mapping) and FoundationDB across AWS and GCP.
3. **Phase 3: Azure & Zero-Disk Topologies:** Enable Azure Blob Storage tiering for ClickHouse and connect all workflow completions to Temporal workflows driving OpenTofu and Kubernetes ClickHouse Operator deployments.