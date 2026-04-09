# Data Mesh Platform — Multi-Region Resiliency Proposal

**Version:** 1.0  
**Date:** April 9, 2026  
**Status:** Draft — For Architecture Review  
**Primary Region:** us-east-1 (N. Virginia)  
**Secondary Region:** us-west-2 (Oregon)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current Architecture Overview](#2-current-architecture-overview)
3. [Multi-Region Resiliency Goals & Constraints](#3-multi-region-resiliency-goals--constraints)
4. [Component-by-Component Deep Dive](#4-component-by-component-deep-dive)
   - 4.1 [Producer S3 Data Layer](#41-producer-s3-data-layer)
   - 4.2 [Iceberg Tables — The Hard Problem](#42-iceberg-tables--the-hard-problem)
   - 4.3 [S3 VPC Endpoints & Network Path](#43-s3-vpc-endpoints--network-path)
   - 4.4 [AWS Glue Catalog](#44-aws-glue-catalog)
   - 4.5 [Data Mesh Platform APIs (ECS, Lambda, DynamoDB, Route53)](#45-data-mesh-platform-apis)
   - 4.6 [Snowflake Compute Platform](#46-snowflake-compute-platform)
   - 4.7 [Starburst Compute Platform](#47-starburst-compute-platform)
   - 4.8 [Databricks Compute Platform](#48-databricks-compute-platform)
   - 4.9 [Immuta Governance Layer](#49-immuta-governance-layer)
   - 4.10 [Consumer Notification Layer (SNS-SQS, Kafka, API)](#410-consumer-notification-layer)
5. [Cross-Account & IAM Considerations](#5-cross-account--iam-considerations)
6. [Failover Scenarios & Impact Analysis](#6-failover-scenarios--impact-analysis)
   - 6.1 [Scenario A — Full Failover (All Parties Flip to us-west-2)](#61-scenario-a--full-failover-all-parties-flip-to-us-west-2)
   - 6.2 [Scenario B — Critical Producers Flip, Non-Critical Stay](#62-scenario-b--critical-producers-flip-non-critical-stay)
   - 6.3 [Scenario C — Critical Producer Refuses to Flip During Resilience Test](#63-scenario-c--critical-producer-refuses-to-flip-during-resilience-test)
   - 6.4 [Scenario D — Platform Flips, Producers Stay](#64-scenario-d--platform-flips-producers-stay)
   - 6.5 [Scenario E — Immuta Region Failure](#65-scenario-e--immuta-region-failure)
   - 6.6 [Scenario F — DX/VPN to On-Prem Failure](#66-scenario-f--dxvpn-to-on-prem-failure)
   - 6.7 [Consumer Impact Matrix](#67-consumer-impact-matrix)
7. [Recommended Architecture — Target State](#7-recommended-architecture--target-state)
8. [Action Plan by Team](#8-action-plan-by-team)
   - 8.1 [Producer Teams](#81-producer-teams)
   - 8.2 [Data Mesh Platform Team](#82-data-mesh-platform-team)
   - 8.3 [Governance (Immuta) Team](#83-governance-immuta-team)
   - 8.4 [Consumer Teams](#84-consumer-teams)
9. [Implementation Phases & Timeline](#9-implementation-phases--timeline)
10. [Blockers, Risks & Mitigations](#10-blockers-risks--mitigations)
11. [RPO / RTO Summary](#11-rpo--rto-summary)
12. [Decision Log](#12-decision-log)
13. [Appendix](#13-appendix)

---

## 1. Executive Summary

This proposal defines the architecture, action plan, and failover strategy to make the Data Mesh platform resilient to a complete AWS region failure in **us-east-1**, with **us-west-2** as the secondary region.

### Why This Matters

A full us-east-1 outage would today render the entire Data Mesh platform — registration APIs, compute platforms, governance enforcement, and all consumer queries — completely unavailable. Historical precedent (us-east-1 outages in 2017, 2020, 2021, 2023) shows this is not a theoretical risk.

### Key Challenges Identified

| # | Challenge | Severity |
|---|-----------|----------|
| 1 | **Iceberg metadata contains absolute S3 paths** — S3 CRR alone does not make Iceberg tables queryable in another region | **Critical** |
| 2 | **Snowflake cannot replicate external/unmanaged Iceberg tables** — these are silently skipped during replication | **Critical** |
| 3 | **S3 Gateway VPC endpoints are same-region only** — compute platforms in us-west-2 cannot reach us-east-1 S3 buckets via gateway endpoints | **Critical** |
| 4 | **Glue Catalog is regional** — table definitions and partition metadata do not replicate automatically | **High** |
| 5 | **Immuta has no native multi-instance policy sync** — DR requires PostgreSQL PITR or Policy-as-Code automation | **High** |
| 6 | **S3 CRR has no ordering guarantee** — Iceberg metadata may replicate before or after data files, causing transient read failures | **High** |
| 7 | **Each producer is a separate AWS account** — multi-region setup must be coordinated across dozens of independent teams | **Medium** |
| 8 | **DynamoDB Global Tables use last-writer-wins** — silent data loss possible during dual-region writes | **Medium** |
| 9 | **SNS topics are regional** — consumer SQS subscriptions in us-east-1 become unreachable during outage; consumers need SQS in us-west-2 | **High** |
| 10 | **On-prem Kafka connectivity** — us-west-2 Lambda/ECS must have network path to on-prem Kafka brokers (Direct Connect / VPN) | **High** |

### Recommended Approach

An **active-passive (warm standby)** architecture where:

- The secondary region (us-west-2) infrastructure is pre-provisioned and kept warm
- Data replication runs continuously via S3 CRR with Replication Time Control (RTC)
- Failover is triggered manually by the platform team after confirming a sustained regional outage
- Target **RPO: 15 minutes** | Target **RTO: 30–60 minutes**

---

## 2. Current Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CURRENT STATE (us-east-1 only)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                      │
│  │ Producer A   │  │ Producer B   │  │ Producer C   │  ... (N producers)   │
│  │ (Own Account)│  │ (Own Account)│  │ (Own Account)│                      │
│  │              │  │              │  │              │                      │
│  │  S3 Bucket   │  │  S3 Bucket   │  │  S3 Bucket   │                      │
│  │  (Parquet)   │  │  (Iceberg)   │  │  (Parquet)   │                      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                      │
│         │                 │                 │                               │
│         │    Register Dataset APIs          │                               │
│         ▼                 ▼                 ▼                               │
│  ┌─────────────────────────────────────────────────────┐                    │
│  │         Data Mesh Platform (Platform Account)       │                    │
│  │                                                     │                    │
│  │   Route53 ──► ALB ──► ECS (Registration APIs)       │                    │
│  │                       Lambda (Notification APIs)     │                    │
│  │                       DynamoDB (Metadata Store)      │                    │
│  └───────────┬─────────────┬──────────────┬────────────┘                    │
│              │             │              │                                  │
│              ▼             ▼              ▼                                  │
│  ┌───────────────┐ ┌──────────────┐ ┌──────────────┐                       │
│  │  Snowflake    │ │  Starburst   │ │  Databricks  │                       │
│  │               │ │              │ │              │                       │
│  │ External Tbls │ │ Glue Catalog │ │ External Tbls│                       │
│  │ Unmanaged     │ │ S3 Access    │ │              │                       │
│  │ Iceberg Tbls  │ │ Point Alias  │ │              │                       │
│  │ Storage Integ.│ │              │ │              │                       │
│  └───────┬───────┘ └──────┬───────┘ └──────┬───────┘                       │
│          │                │                │                                │
│          │   S3 VPC Endpoint (Gateway)     │                                │
│          ▼                ▼                ▼                                │
│       Producer S3 Buckets (us-east-1)                                      │
│                                                                             │
│  ┌────────────────────────────────────────┐                                │
│  │    Immuta (Governance Team Account)    │                                │
│  │                                        │                                │
│  │  Starburst: Runtime API calls          │                                │
│  │  Snowflake: Native policy sync         │                                │
│  │  Databricks: Native policy sync        │                                │
│  └────────────────────────────────────────┘                                │
│                                                                             │
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │    Consumer Notification Layer (Platform Account)             │         │
│  │                                                               │         │
│  │  SNS (us-east-1) ──► Consumer SQS (consumer accounts)        │         │
│  │  On-Prem Kafka  ◄── Lambda/ECS push messages                 │         │
│  │  Notification API ──► Consumer polls for new data             │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                                                                             │
│  ┌────────────────────────────────────────┐                                │
│  │           Consumers                     │                                │
│  │  Query via Snowflake / Starburst /     │                                │
│  │  Databricks using entitled datasets    │                                │
│  │  Notified via SNS-SQS / Kafka / API   │                                │
│  └────────────────────────────────────────┘                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Architectural Facts

| Component | Account | Region | Technology |
|-----------|---------|--------|------------|
| Producer Data | Each producer's own AWS account | us-east-1 | S3 (Parquet or Iceberg) |
| Data Mesh APIs | Platform team's AWS account | us-east-1 | ECS, Lambda, DynamoDB, Route53 |
| Snowflake | Snowflake (managed service) | us-east-1 deployment | Storage Integration, External Tables, Unmanaged Iceberg Tables |
| Starburst | Platform team's AWS account (or dedicated) | us-east-1 | Glue Catalog, S3 Access Point Aliases |
| Databricks | Databricks (managed service) | us-east-1 workspace | External Tables |
| Immuta | Governance team's AWS account | us-east-1 | Policy Engine, Native sync to SF/DBX, Runtime API for Starburst |
| Consumer Notifications (SNS) | Platform team's AWS account | us-east-1 | SNS topics → Consumer SQS queues (cross-account) |
| Consumer Notifications (Kafka) | On-premises | N/A (on-prem) | On-prem Kafka cluster, messages pushed by platform Lambda/ECS |
| Consumer Notifications (API) | Platform team's AWS account | us-east-1 | Notification polling API (ECS/Lambda + DynamoDB) |
| Network | Each account with VPC | us-east-1 | S3 Gateway VPC Endpoints |

---

## 3. Multi-Region Resiliency Goals & Constraints

### Goals

| Goal | Target |
|------|--------|
| **Recovery Point Objective (RPO)** | ≤ 15 minutes (data loss tolerance) |
| **Recovery Time Objective (RTO)** | ≤ 60 minutes (time to restore service) |
| **Failover Trigger** | Manual, after confirmed sustained us-east-1 outage |
| **Failback** | Manual, after us-east-1 is fully restored and verified |
| **Consumer Experience** | Consumers reconnect to compute platforms with minimal/no connection string changes |
| **Data Consistency** | Eventual consistency acceptable during replication; strong consistency not required |

### Constraints

1. **Each producer is an independent AWS account** — the platform team cannot enforce changes; must provide guidance and tooling
2. **Producers may adopt multi-region at different paces** — the solution must handle partial adoption
3. **Budget sensitivity** — warm standby preferred over hot standby to manage costs
4. **Snowflake limitation** — external/unmanaged Iceberg tables cannot be natively replicated
5. **Iceberg metadata absolute paths** — requires explicit handling during failover
6. **Immuta is managed by a separate team** — coordination required but limited control

### Non-Goals

- Active-active (dual-write) architecture — too complex, not needed for DR
- Zero RPO — would require synchronous replication, cost-prohibitive
- Automatic failover without human decision — risk of split-brain, premature failover

---

## 4. Component-by-Component Deep Dive

### 4.1 Producer S3 Data Layer

#### Current State
- Each producer stores data in S3 buckets within their own AWS account in us-east-1
- Data formats: Parquet (flat files with partitioned directory structure) or Iceberg (data files + metadata layer)
- Compute platforms access these buckets via cross-account IAM roles and S3 VPC endpoints

#### What Needs to Change

**For Parquet Producers:**

| Step | Action | Owner |
|------|--------|-------|
| 1 | Create a mirror S3 bucket in us-west-2 in the same account | Producer |
| 2 | Enable S3 Cross-Region Replication (CRR) from us-east-1 → us-west-2 | Producer |
| 3 | Enable S3 Replication Time Control (RTC) for 15-minute SLA | Producer |
| 4 | Enable S3 Replication Metrics and Notifications | Producer |
| 5 | Grant cross-account access on the us-west-2 bucket to compute platform roles | Producer |
| 6 | Create S3 Access Points on the us-west-2 bucket (for Starburst) | Producer |

**For Iceberg Producers (additional steps beyond Parquet):**

| Step | Action | Owner |
|------|--------|-------|
| 7 | Enable CRR for both data files AND metadata files (same bucket replication covers both) | Producer |
| 8 | Register Iceberg table in us-west-2 Glue Catalog (pointing to replicated metadata location) | Platform Team (via API) |
| 9 | **Decision Required**: Adopt S3 Multi-Region Access Points (MRAP) for new tables so metadata contains region-agnostic paths | Producer + Platform |
| 10 | For existing tables: build automation to update metadata paths during failover using `register_table` procedure | Platform Team |

> **CRITICAL FINDING — Iceberg Metadata Absolute Paths**  
> Iceberg metadata files (metadata.json, manifest lists, manifest files) contain **absolute S3 paths** (e.g., `s3://producer-bucket-east/warehouse/table/data/file.parquet`). When data is replicated via CRR to `s3://producer-bucket-west/...`, the metadata in the west bucket still references the east bucket. The table is **unreadable** in us-west-2 without additional intervention.  
> See [Section 4.2](#42-iceberg-tables--the-hard-problem) for detailed solutions.

#### S3 CRR Configuration Requirements

```
Source Bucket: s3://producer-data-us-east-1/
Destination Bucket: s3://producer-data-us-west-2/

Replication Configuration:
  - Scope: Entire bucket (or prefix-based for selective replication)
  - Replication Time Control: ENABLED (15-minute SLA, 99.99%)
  - Replica Modifications Sync: ENABLED
  - Delete Marker Replication: ENABLED
  - Metrics: ENABLED (monitor via CloudWatch)
  - S3 Object Lock: Match source bucket settings

IAM Role Requirements:
  - Source bucket: s3:GetReplicationConfiguration, s3:ListBucket
  - Source objects: s3:GetObjectVersionForReplication, s3:GetObjectVersionAcl
  - Destination bucket: s3:ReplicateObject, s3:ReplicateDelete, s3:ReplicateTags
```

#### Cost Impact

| Item | Estimated Monthly Cost per Producer |
|------|-------------------------------------|
| S3 CRR Data Transfer (us-east-1 → us-west-2) | $0.02/GB |
| S3 Storage (us-west-2 replica) | Same as source (~$0.023/GB-month for Standard) |
| S3 RTC | $0.015/GB + $0.01 per 10,000 RTC replicated objects |
| S3 Replication Metrics | Included with RTC |

---

### 4.2 Iceberg Tables — The Hard Problem

This is the most technically challenging aspect of the multi-region strategy.

#### The Problem in Detail

Iceberg tables consist of:
```
table/
├── metadata/
│   ├── v1.metadata.json          ← Contains absolute S3 paths
│   ├── v2.metadata.json          ← Contains absolute S3 paths
│   ├── snap-1234-manifest-list.avro  ← Contains absolute S3 paths
│   └── 5678-manifest.avro        ← Contains absolute S3 paths to data files
└── data/
    ├── partition=2024-01/
    │   ├── file1.parquet
    │   └── file2.parquet
    └── partition=2024-02/
        └── file3.parquet
```

Every metadata file contains fully qualified S3 URIs:
```json
{
  "format-version": 2,
  "table-uuid": "abc-123",
  "location": "s3://producer-bucket-us-east-1/warehouse/my_table",
  "snapshots": [{
    "manifest-list": "s3://producer-bucket-us-east-1/warehouse/my_table/metadata/snap-1234.avro"
  }]
}
```

After CRR to us-west-2, the replicated metadata.json in `s3://producer-bucket-us-west-2/...` still references `s3://producer-bucket-us-east-1/...`. If us-east-1 is down, these paths are unresolvable.

#### Solution Options (Ranked by Recommendation)

**Option 1: S3 Multi-Region Access Points (MRAP) — Recommended for New Tables**

| Aspect | Detail |
|--------|--------|
| **How it works** | Create an MRAP spanning buckets in both regions. Write Iceberg tables using MRAP endpoint as location (e.g., `s3://mfzwi23gnjvgw.mrap/warehouse/table/`). Metadata contains MRAP paths, not physical bucket paths. During failover, MRAP routes to surviving region. |
| **Pros** | Region-agnostic paths in metadata; automatic routing during failover; no metadata rewriting needed |
| **Cons** | Must be adopted from table creation time; existing tables need migration; MRAP + Iceberg currently has best support with Spark only — **Snowflake and Starburst MRAP compatibility must be validated** |
| **RPO** | ~15 minutes (CRR + RTC) |
| **Effort** | Medium (for new tables), High (for migrating existing tables) |

**Option 2: Failover-Time Metadata Re-registration — Recommended for Existing Tables**

| Aspect | Detail |
|--------|--------|
| **How it works** | During failover: (1) Locate the latest `metadata.json` in the replicated us-west-2 bucket. (2) Use Iceberg's `register_table` procedure to register the table in us-west-2 Glue Catalog, pointing to the west bucket. (3) Compute platforms read from the registered west catalog. |
| **Pros** | Works with existing tables without migration; no changes to current write path |
| **Cons** | Requires automation to execute during failover; **race condition risk** — metadata may reference data files not yet replicated; adds to RTO |
| **RPO** | ~15 minutes (CRR + RTC) + validation time |
| **Effort** | Medium (build automation, test thoroughly) |

**How `register_table` works during failover:**
```sql
-- In Spark connected to us-west-2 Glue Catalog:
CALL system.register_table(
  table => 'glue_catalog.database_name.table_name',
  metadata_file => 's3://producer-bucket-us-west-2/warehouse/table/metadata/v42.metadata.json'
);
```

> **Important:** Even with `register_table`, the metadata.json still references east bucket paths internally. This works ONLY if:
> - The west bucket has the same prefix structure as the east bucket, AND
> - The compute engine is configured to do path substitution (replace `s3://producer-bucket-us-east-1/` with `s3://producer-bucket-us-west-2/`), OR
> - You run a metadata rewrite job that updates all internal paths

**Option 3: S3 Tables (Managed Iceberg Service) — Future State**

| Aspect | Detail |
|--------|--------|
| **How it works** | Amazon S3 Tables provides managed Iceberg table service with native replication that **automatically rewrites metadata paths** and enforces **strict snapshot ordering**. |
| **Pros** | Fully managed; path rewriting handled automatically; snapshot ordering guaranteed |
| **Cons** | Only works with S3 Tables service (not arbitrary S3 buckets); producers would need to migrate to S3 Tables |
| **RPO** | Minutes (AWS-managed) |
| **Effort** | High (migration to new service) |

**Option 4: Dual-Write Pipeline — Highest Fidelity**

| Aspect | Detail |
|--------|--------|
| **How it works** | Producer writes to both regions via a Spark/Flink pipeline. Each region has native, independently written Iceberg tables with correct local paths. |
| **Pros** | No metadata path issues; each region is fully independent; no CRR dependency |
| **Cons** | Doubles write compute cost; complex to maintain consistency; pipeline must handle failures |
| **RPO** | Near-zero (if pipeline is healthy) |
| **Effort** | Very High |

#### Recommendation

| Table Type | Strategy |
|------------|----------|
| **New Iceberg tables** (going forward) | Adopt MRAP-based paths at creation time (Option 1) |
| **Existing Iceberg tables** | Build `register_table` + path substitution automation for failover (Option 2) |
| **Long term** | Evaluate migration to S3 Tables as the service matures (Option 3) |

#### CRR Ordering Risk for Iceberg

S3 CRR provides **no ordering guarantee** across objects. This creates a dangerous race condition:

```
Timeline:
  T1: Producer commits snapshot S5 → metadata.json updated → references new data files
  T2: CRR replicates metadata.json to us-west-2 (fast, small file)
  T3: CRR has NOT yet replicated referenced data files (large parquet files)
  T4: Consumer reads from us-west-2 → metadata says data file exists → FileNotFoundException
```

**Mitigations:**
1. After failover, run a **validation job** that checks the latest snapshot's manifest — verify all referenced data files exist in the west bucket before allowing consumer access
2. Implement a **staleness buffer** — use the second-to-last snapshot (which is more likely fully replicated) rather than the latest
3. Monitor S3 `ReplicationLatency` metric — do not declare failover ready until replication lag is below threshold

---

### 4.3 S3 VPC Endpoints & Network Path

#### Current State
- Compute platforms (Snowflake, Starburst, Databricks) access producer S3 buckets via **S3 Gateway VPC Endpoints**
- Starburst specifically uses **S3 Access Point Aliases** to route to producer buckets
- All access is same-region (us-east-1 VPC endpoint → us-east-1 S3 buckets)

#### Critical Constraint: Gateway Endpoints Are Same-Region Only

> **S3 Gateway VPC Endpoints can ONLY access S3 buckets in the same region as the endpoint.**  
> A gateway endpoint in us-west-2 cannot reach a bucket in us-east-1.  
> S3 Access Points are also regional — bound to one bucket in one region.

This means: if a producer's data is only in us-east-1, a compute platform running in us-west-2 **cannot access it via VPC endpoints at all**.

#### What Needs to Change

**For Compute Platforms in us-west-2:**

| Component | us-east-1 (Current) | us-west-2 (New) |
|-----------|---------------------|------------------|
| S3 Gateway VPC Endpoint | Existing | Must create new in us-west-2 VPC |
| S3 Access Points (Starburst) | Existing on us-east-1 buckets | Must create on us-west-2 replicated buckets |
| VPC | Existing | Must create new VPC in us-west-2 (or extend existing) |
| Subnets, Route Tables | Existing | Must mirror in us-west-2 |
| Security Groups | Existing | Must recreate in us-west-2 |

**For Cross-Region Access During Partial Failover (some producers in east, compute in west):**

This is the **hardest networking problem** in partial failover scenarios.

| Option | How | Pros | Cons |
|--------|-----|------|------|
| **S3 Interface Endpoint (PrivateLink)** | Create interface endpoint in us-west-2 VPC for S3; access us-east-1 buckets cross-region via PrivateLink | Works cross-region; private network | Per-hour + per-GB cost; higher latency; complex DNS configuration |
| **S3 Multi-Region Access Point + PrivateLink** | Create MRAP spanning both regions; use `com.amazonaws.s3-global.accesspoint` endpoint type in us-west-2 VPC | Region-agnostic access; automatic routing | Requires MRAP setup by producers; must validate compute platform compatibility |
| **Transit Gateway Inter-Region Peering** | Peer VPCs across regions via Transit Gateway; route S3 traffic through east TGW | Full network connectivity | Expensive; complex; adds latency; may not work with managed services |
| **Accept the limitation** | If producer is only in us-east-1 and region is down, their data is unavailable from us-west-2 | Simple; no engineering | Data unavailable during outage |

**Recommendation:** For producers who implement multi-region (CRR to us-west-2), create local S3 Access Points on the replicated bucket. For producers who do NOT implement multi-region, their data is accepted as unavailable during us-east-1 outage.

#### Starburst-Specific: Access Point Aliases

Starburst currently uses S3 Access Point Aliases (e.g., `prodA-ap-east-s3alias`) to access producer buckets. In us-west-2:

1. Producer creates new S3 Access Point on their us-west-2 replicated bucket
2. New alias is generated (e.g., `prodA-ap-west-s3alias`)
3. Starburst catalog configuration in us-west-2 cluster must reference the new alias
4. Platform team's Glue Catalog entries in us-west-2 must reference the new access point

#### Snowflake-Specific: Storage Integration

Snowflake uses Storage Integrations to access S3. For us-west-2:

1. Create a new Storage Integration in the us-west-2 Snowflake account pointing to the replicated S3 bucket
2. Create a new External Volume (for Iceberg tables) pointing to us-west-2 storage
3. Producer must grant the Snowflake IAM role access to the us-west-2 bucket
4. Trust relationship must be established between Snowflake's us-west-2 account IAM role and the producer's bucket policy

---

### 4.4 AWS Glue Catalog

#### Current State
- Starburst uses Glue Catalog in us-east-1 as its metastore
- Data Mesh Notification API adds partitions to Glue Catalog tables
- Iceberg tables registered in Glue Catalog point to us-east-1 S3 locations

#### Challenge
- **Glue Catalog is regional** — table definitions in us-east-1 Glue do not exist in us-west-2 Glue
- Partition information accumulated over time via Notification APIs must be available in us-west-2

#### What Needs to Change

| Step | Action | Owner |
|------|--------|-------|
| 1 | Set up **AWS Glue Catalog replication** from us-east-1 → us-west-2 using the [Lake Formation cross-region table access](https://docs.aws.amazon.com/lake-formation/latest/dg/cross-region-access.html) or the [Glue Catalog replication utility](https://github.com/aws-samples/lake-formation-pemissions-sync) | Platform Team |
| 2 | Alternatively, use **Lake Formation resource links** in us-west-2 that point to us-east-1 tables (works during normal operations but NOT during us-east-1 outage) | Platform Team |
| 3 | **Recommended:** Modify Notification API to write partition information to **both** us-east-1 and us-west-2 Glue Catalogs simultaneously (dual-write) | Platform Team |
| 4 | For Iceberg tables: register tables in us-west-2 Glue Catalog pointing to replicated S3 locations | Platform Team |
| 5 | Apply Lake Formation permissions in us-west-2 catalog (permissions do not replicate automatically) | Platform Team |

#### Recommended Approach: Dual-Write to Both Glue Catalogs

```
Producer calls Notification API
        │
        ▼
  Notification API (Lambda)
        │
        ├──► Add partition to us-east-1 Glue Catalog  (current)
        │
        └──► Add partition to us-west-2 Glue Catalog  (NEW - dual write)
             (with S3 location pointing to us-west-2 replicated bucket)
```

This ensures us-west-2 Glue Catalog is always up to date without relying on a separate replication mechanism. The us-west-2 entries point to the replicated bucket paths.

**Trade-off:** Dual-write adds latency to the Notification API call (~50-100ms for cross-region Glue API call). If this is unacceptable, use async dual-write (SQS queue in us-west-2 that a Lambda drains).

---

### 4.5 Data Mesh Platform APIs

#### Current State
- **Registration APIs**: ECS services behind ALB, exposed via Route53
- **Notification APIs**: Lambda functions (add partitions, refresh tables)
- **Metadata Store**: DynamoDB (stores dataset registrations, configurations)
- **DNS**: Route53 for API domain names
- All deployed in the platform team's AWS account in us-east-1

#### What Needs to Change

**A. DynamoDB — Enable Global Tables**

| Step | Action | Detail |
|------|--------|--------|
| 1 | Convert DynamoDB tables to **Global Tables** | Add us-west-2 as a replica region |
| 2 | Choose consistency model | **MREC (Eventual Consistency)** recommended — supports us-east-1 + us-west-2 without restrictions |
| 3 | Monitor replication | Use `ReplicationLatency` CloudWatch metric (typically 1-2 seconds) |
| 4 | Handle conflict resolution | Design for region-partitioned writes to minimize LWW conflicts |

**DynamoDB Global Tables Considerations:**
- Replication lag: typically **< 2 seconds**
- Conflict resolution: **Last Writer Wins (LWW)** based on timestamp — no visibility into lost writes
- Mitigation: During normal operations, only us-east-1 writes. us-west-2 replica is read-only until failover
- After failover: writes go to us-west-2. After failback: reconcile any conflicting writes
- Cost: Every write is replicated, so you pay for writes in both regions

**B. ECS Services — Deploy in us-west-2**

| Step | Action | Detail |
|------|--------|--------|
| 1 | Create VPC, subnets, ALB in us-west-2 | Mirror us-east-1 networking |
| 2 | Deploy ECS cluster in us-west-2 | Same task definitions, service configurations |
| 3 | **Warm standby**: Run with minimum desired count (e.g., 1 task) | Scale up during failover |
| 4 | Keep ECR images in sync | Use ECR cross-region replication |
| 5 | Configure ECS services to connect to us-west-2 DynamoDB, Glue, etc. | Environment-specific config via SSM Parameter Store |

**C. Lambda Functions — Deploy in us-west-2**

| Step | Action | Detail |
|------|--------|--------|
| 1 | Deploy all Lambda functions in us-west-2 | Use CloudFormation StackSets or Terraform workspaces |
| 2 | Configure Lambda to use us-west-2 regional services | DynamoDB, Glue, Snowflake, Starburst endpoints |
| 3 | **Dual-write Notification Lambdas** | See Section 4.4 — Lambdas write to both regional Glue Catalogs |

**D. Route53 — Failover Routing**

| Step | Action | Detail |
|------|--------|--------|
| 1 | Create health checks on us-east-1 ALB endpoints | HTTP health check, 10-second interval, 3 failure threshold |
| 2 | Configure **failover routing policy** | Primary: us-east-1 ALB, Secondary: us-west-2 ALB |
| 3 | Set TTL to **60 seconds** | Balance between failover speed and DNS caching |
| 4 | Consider **Route53 Application Recovery Controller (ARC)** | For coordinated failover of multiple endpoints |

**Route53 Failover Architecture:**
```
  api.datamesh.company.com
          │
          ▼
      Route53 (Failover Policy)
          │
    ┌─────┴─────┐
    │           │
    ▼           ▼
 us-east-1   us-west-2
   ALB         ALB        (Health check fails → traffic routes to west)
    │           │
    ▼           ▼
   ECS         ECS
 (Primary)   (Warm Standby)
```

**E. Infrastructure as Code**

All us-west-2 infrastructure should be deployed via the same IaC (Terraform/CloudFormation) that manages us-east-1, parameterized by region. This ensures:
- Configuration drift is minimized
- Changes to us-east-1 automatically propagate to us-west-2
- Failover infrastructure is always in sync

---

### 4.6 Snowflake Compute Platform

#### Current State
- Snowflake account deployed in us-east-1 (AWS)
- External tables for Parquet producers
- Unmanaged Iceberg tables for Iceberg producers
- Storage Integrations grant access to producer S3 buckets
- Immuta syncs policies natively to Snowflake

#### Critical Limitation

> **Snowflake CANNOT replicate external tables or externally managed (unmanaged) Iceberg tables.**  
> These objects are **silently skipped** during replication refresh operations (BCR-1528).  
> Catalog-linked databases do not support replication or cloning.

This means the entire external table layer — which is the core of the Data Mesh on Snowflake — **must be recreated from scratch** in the DR Snowflake account.

#### What Needs to Change

| Step | Action | Owner |
|------|--------|-------|
| 1 | **Provision a Snowflake account in us-west-2** (Business Critical edition for failover groups) | Platform Team |
| 2 | **Replicate internal objects** (databases, schemas, roles, warehouses, grants) via Snowflake Account Replication / Failover Groups | Platform Team |
| 3 | **Create Storage Integrations** in us-west-2 account pointing to producer us-west-2 S3 buckets | Platform Team |
| 4 | **Create External Volumes** in us-west-2 account pointing to us-west-2 storage locations | Platform Team |
| 5 | **Build automation to recreate all external tables and unmanaged Iceberg tables** in us-west-2 account | Platform Team |
| 6 | Source DDL from DynamoDB metadata store (which replicates via Global Tables) | Platform Team |
| 7 | **Run recreation automation periodically** (e.g., nightly) to keep us-west-2 Snowflake warm | Platform Team |
| 8 | Configure Immuta integration for us-west-2 Snowflake account | Governance Team |
| 9 | Producers must grant Snowflake us-west-2 IAM role access to their us-west-2 S3 buckets | Producer |

#### Automation Design for External Table Recreation

```
┌─────────────────────────────────────────────────────┐
│         External Table Recreation Pipeline           │
│                                                      │
│  1. Read dataset metadata from DynamoDB              │
│     (replicated via Global Tables)                   │
│                                                      │
│  2. For each dataset:                                │
│     a. Determine table type (External / Iceberg)     │
│     b. Generate DDL with us-west-2 S3 paths          │
│     c. Create/Replace table in us-west-2 Snowflake   │
│                                                      │
│  3. For Iceberg tables:                              │
│     a. Point to us-west-2 external volume            │
│     b. Set metadata location to us-west-2 bucket     │
│                                                      │
│  4. Validate: SHOW TABLES, check row counts          │
│                                                      │
│  Schedule: Nightly sync + on-demand during failover  │
└─────────────────────────────────────────────────────┘
```

#### Snowflake Consumer Connectivity

| Approach | How | Trade-off |
|----------|-----|-----------|
| **Connection Redirect** (Recommended) | Consumers use a CNAME (e.g., `datamesh-sf.company.com`) that resolves to either the us-east-1 or us-west-2 Snowflake account URL via Route53 failover | Requires consumers to use the CNAME, not direct Snowflake URLs |
| **Snowflake Client Redirect** | Use Snowflake's [Client Redirect](https://docs.snowflake.com/en/user-guide/client-redirect) feature to transparently redirect connections to the secondary account | Requires Business Critical edition; some client libraries may not support it |
| **Manual switch** | Notify consumers to change their connection string | Slowest RTO; error-prone |

---

### 4.7 Starburst Compute Platform

#### Current State
- Starburst cluster deployed in us-east-1
- Uses Glue Catalog as metastore
- Accesses producer S3 via Access Point Aliases through S3 Gateway VPC Endpoints
- Immuta enforces policies via runtime API calls

#### What Needs to Change

| Step | Action | Owner |
|------|--------|-------|
| 1 | **Deploy a Starburst cluster in us-west-2** | Platform Team |
| 2 | Create VPC, subnets, S3 Gateway VPC Endpoint in us-west-2 | Platform Team |
| 3 | Configure us-west-2 Starburst to use **us-west-2 Glue Catalog** | Platform Team |
| 4 | Configure us-west-2 Starburst with S3 Access Points for **us-west-2 replicated buckets** | Platform Team + Producers |
| 5 | Configure IAM roles for us-west-2 Starburst to access us-west-2 producer buckets | Platform Team + Producers |
| 6 | Configure Immuta integration for us-west-2 Starburst cluster | Governance Team |
| 7 | Set up **Starburst Gateway** or DNS-based routing for consumer failover | Platform Team |
| 8 | Run the us-west-2 cluster as **warm standby** (minimal worker nodes) | Platform Team |

#### Starburst Gateway for Failover

```
  Consumer query
       │
       ▼
  Starburst Gateway (or DNS CNAME with Route53 failover)
       │
  ┌────┴────┐
  │         │
  ▼         ▼
 East      West
Cluster    Cluster     ← Route53 health check triggers failover
  │         │
  ▼         ▼
East       West
Glue       Glue        ← Each cluster uses its regional Glue Catalog
  │         │
  ▼         ▼
East S3    West S3     ← Each cluster accesses its regional S3 via local VPC endpoints
```

#### Catalog Configuration in us-west-2

The us-west-2 Starburst cluster's Iceberg catalog configuration:
```properties
connector.name=iceberg
iceberg.catalog.type=glue
# Points to us-west-2 Glue Catalog (automatic — Glue SDK uses regional endpoint)
hive.metastore.glue.region=us-west-2
# S3 file system configuration
hive.s3.endpoint=s3.us-west-2.amazonaws.com
```

Each producer's S3 access in the catalog is configured with the us-west-2 Access Point Alias, so no cross-region S3 access is needed.

---

### 4.8 Databricks Compute Platform

#### Current State
- Databricks workspace deployed in us-east-1
- External tables for producer data
- Immuta syncs policies natively

#### What Needs to Change

| Step | Action | Owner |
|------|--------|-------|
| 1 | **Provision a Databricks workspace in us-west-2** | Platform Team |
| 2 | Configure Unity Catalog metastore in us-west-2 | Platform Team |
| 3 | Create external locations pointing to us-west-2 replicated S3 buckets | Platform Team |
| 4 | Build automation to recreate external tables in us-west-2 workspace | Platform Team |
| 5 | Configure IAM roles for us-west-2 Databricks to access us-west-2 producer buckets | Platform Team + Producers |
| 6 | Configure Immuta integration for us-west-2 Databricks workspace | Governance Team |
| 7 | Set up consumer connection routing (DNS CNAME or workspace URL redirect) | Platform Team |

Databricks supports [disaster recovery patterns](https://docs.databricks.com/en/admin/disaster-recovery.html) including workspace replication via Terraform and Unity Catalog metastore replication.

---

### 4.9 Immuta Governance Layer

#### Current State
- Immuta deployed in the governance team's AWS account in us-east-1
- **Starburst**: Immuta API called at runtime during query execution for policy enforcement
- **Snowflake & Databricks**: Immuta syncs policies natively to these platforms (tag-based masking, row filters, etc.)
- All policies, data source registrations, and configurations stored in Immuta's PostgreSQL database

#### Critical Dependencies

If Immuta is unavailable:
- **Starburst queries fail** (runtime dependency — no policy check = no query execution)
- **Snowflake/Databricks continue working** with last-synced policies (eventual consistency — policies already materialized in the platform)

#### What Needs to Change

| Step | Action | Owner |
|------|--------|-------|
| 1 | **Deploy Immuta instance in us-west-2** (SaaS: coordinate with Immuta; Self-managed: deploy via Helm) | Governance Team |
| 2 | **Implement Policy-as-Code sync** using Immuta CLI to keep both instances consistent | Governance Team |
| 3 | Set up PostgreSQL cross-region replication (for self-managed) or rely on Immuta SaaS DR (for SaaS) | Governance Team |
| 4 | Configure us-west-2 Immuta instance integrations: connect to us-west-2 Starburst, Snowflake, Databricks | Governance Team |
| 5 | Configure us-west-2 Starburst to call us-west-2 Immuta API | Platform Team + Governance Team |
| 6 | Configure us-west-2 Snowflake/Databricks for policy sync from us-west-2 Immuta | Governance Team |
| 7 | Set up **CI/CD pipeline** for Policy-as-Code: Git repo → apply to both Immuta instances | Governance Team |

#### Policy-as-Code Sync Architecture

```
  ┌─────────────────────────────┐
  │     Git Repository          │
  │  (Policy definitions YAML)  │
  └──────────┬──────────────────┘
             │
        CI/CD Pipeline
             │
     ┌───────┴───────┐
     │               │
     ▼               ▼
  Immuta           Immuta
  us-east-1        us-west-2
     │               │
     ├──► Starburst   ├──► Starburst
     │    (runtime)   │    (runtime)
     │               │
     ├──► Snowflake   ├──► Snowflake
     │    (sync)      │    (sync)
     │               │
     └──► Databricks  └──► Databricks
          (sync)           (sync)
```

#### Immuta Failover Considerations

| Scenario | Impact | Mitigation |
|----------|--------|------------|
| Immuta us-east-1 down, Snowflake/Databricks still in us-east-1 | Policies continue with last-synced state; new policy changes cannot be applied | Acceptable for short outages; long outages require failover to us-west-2 |
| Immuta us-east-1 down, Starburst still in us-east-1 | **Starburst queries fail** — runtime dependency | Starburst must fail over to us-west-2 cluster connected to us-west-2 Immuta, OR implement Immuta policy caching in Starburst |
| Full failover to us-west-2 | All compute platforms use us-west-2 Immuta | Ensure policy sync is current; validate integrations pre-failover |

---

### 4.10 Consumer Notification Layer

The Data Mesh platform delivers data-availability notifications to consumers through three channels. Each has distinct multi-region implications.

#### Current State

```
  Producer calls Notification API ("new data available")
          │
          ▼
  Platform Notification Service
          │
          ├──► 1. SNS (us-east-1) ──► Consumer SQS (consumer's AWS account, us-east-1)
          │         Consumer polls SQS, then queries compute platform for the data
          │
          ├──► 2. On-Prem Kafka ◄── Lambda/ECS pushes message to on-prem Kafka cluster
          │         Consumer's on-prem apps consume from Kafka topic
          │
          └──► 3. Notification API (DynamoDB-backed)
                    Consumer polls the API for new notifications, then queries data
```

#### Channel 1: SNS → Consumer SQS

**How it works today:**
- Platform owns an SNS topic in us-east-1 (Platform AWS account)
- Each consumer subscribes their SQS queue (in the consumer's own AWS account, us-east-1) to the SNS topic
- When new data is available, platform publishes to SNS → SNS fans out to all subscribed SQS queues
- Consumer applications poll their SQS queue and process the messages

**Multi-Region Challenges:**

| Challenge | Detail |
|-----------|--------|
| **SNS topics are regional** | The us-east-1 SNS topic is unavailable during a region outage. A new SNS topic is needed in us-west-2. |
| **Consumer SQS queues are regional** | If the consumer's SQS is in us-east-1 and that region is down, even a us-west-2 SNS cannot deliver to it. |
| **SNS-SQS subscriptions are regional** | Subscriptions tied to the us-east-1 SNS topic do not exist in us-west-2. Must be recreated. |
| **Cross-region SNS→SQS is supported** | SNS in us-west-2 CAN deliver to SQS in us-east-1 (and vice versa) — but only if both regions are healthy. |
| **Message ordering** | During failover, there may be a gap — messages published to us-east-1 SNS just before the outage may be lost if not yet delivered to SQS. |

**What Needs to Change:**

| Step | Action | Owner |
|------|--------|-------|
| 1 | **Create SNS topic in us-west-2** (mirror of us-east-1 topic) | Platform Team |
| 2 | **Consumers create SQS queue in us-west-2** in their own AWS account | Consumer |
| 3 | **Subscribe us-west-2 SQS to us-west-2 SNS topic** (cross-account subscription) | Consumer + Platform |
| 4 | **Modify platform Notification Lambda/ECS to publish to both SNS topics** (dual-publish) during normal operations, OR publish only to the active region's SNS | Platform Team |
| 5 | **Consumer applications must poll BOTH SQS queues** (east and west), or switch to the active region's SQS during failover | Consumer |
| 6 | Set up **SNS message deduplication** or idempotent consumer logic to handle potential duplicate messages during failover transitions | Consumer |

**Recommended Architecture:**

```
  NORMAL OPERATION:
  Platform Notification Service
          │
          ├──► SNS (us-east-1) ──► Consumer SQS (us-east-1)  ← Consumer polls this
          │
          └──► SNS (us-west-2) ──► Consumer SQS (us-west-2)  ← Dormant (warm standby)
               (dual-publish)

  DURING FAILOVER (us-east-1 down):
  Platform Notification Service (running in us-west-2)
          │
          └──► SNS (us-west-2) ──► Consumer SQS (us-west-2)  ← Consumer polls this
```

**Option A (Recommended): Dual-Publish + Consumer polls both**
- Platform always publishes to both SNS topics
- Consumer always has two SQS queues and their application polls both
- During failover, us-east-1 SQS stops receiving; us-west-2 SQS continues
- Consumer application transparently continues with us-west-2 messages
- **Pro:** Zero consumer action needed during failover
- **Con:** Double SNS publish cost during normal operations; consumer must handle deduplication

**Option B: Active-region-only publish + Consumer switches SQS**
- Platform publishes only to the active region's SNS (east normally, west during failover)
- Consumer must detect failover and switch their polling to us-west-2 SQS
- **Pro:** Simpler, lower cost
- **Con:** Consumer must detect and react to failover; messages during transition may be lost

> **RPO for SNS-SQS:** With dual-publish (Option A), RPO is ~0 (messages flow through both regions simultaneously). With active-only (Option B), RPO equals the time between last successful us-east-1 SNS publish and consumer switchover to us-west-2 SQS.

#### Channel 2: On-Prem Kafka

**How it works today:**
- Platform Lambda/ECS pushes data-availability messages to an on-premises Kafka cluster
- Connection is typically via AWS Direct Connect or VPN from the platform VPC to on-prem network
- Consumer's on-prem applications consume from Kafka topics

**Multi-Region Challenges:**

| Challenge | Detail |
|-----------|--------|
| **Network connectivity from us-west-2** | The us-west-2 VPC must have a network path to on-prem Kafka. If Direct Connect / VPN terminates only in us-east-1, us-west-2 Lambda/ECS cannot reach Kafka. |
| **Direct Connect resilience** | If Direct Connect is only provisioned in us-east-1, a region outage may also sever the on-prem link. |
| **Kafka is on-prem (not affected by AWS outage)** | Kafka itself stays available regardless of which AWS region is active — the problem is getting messages TO it. |
| **Message ordering** | Kafka preserves partition ordering; if the producer switches from east Lambda to west Lambda, partition assignment must be consistent. |

**What Needs to Change:**

| Step | Action | Owner |
|------|--------|-------|
| 1 | **Establish Direct Connect / VPN from us-west-2 VPC to on-prem network** | Platform Team + Network Team |
| 2 | Validate Kafka broker reachability from us-west-2 Lambda/ECS (security groups, NACLs, firewall rules) | Platform Team + Network Team |
| 3 | Configure us-west-2 Lambda/ECS with Kafka broker endpoints and TLS certificates | Platform Team |
| 4 | **Test end-to-end:** us-west-2 Lambda → on-prem Kafka → consumer receives message | Platform Team + Consumer |
| 5 | Ensure Kafka topic ACLs allow the us-west-2 producer identity | Platform Team + Kafka Admin |

**Network Architecture:**

```
  NORMAL OPERATION:
  us-east-1 Lambda/ECS ──► Direct Connect (us-east-1) ──► On-Prem Kafka
  
  DURING FAILOVER:
  us-west-2 Lambda/ECS ──► Direct Connect (us-west-2) ──► On-Prem Kafka
                            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                            MUST BE PRE-PROVISIONED!
                            
  Alternative (if DX is only in us-east-1):
  us-west-2 Lambda/ECS ──► Transit Gateway ──► us-east-1 DX ──► On-Prem Kafka
                            (cross-region peering)
  ⚠️ This only works if us-east-1 networking is still partially available
```

> **CRITICAL:** If Direct Connect is exclusively provisioned in us-east-1, and us-east-1 suffers a full network outage, the on-prem Kafka channel will be **completely unavailable** from us-west-2 until a separate DX/VPN is established. This is a **blocker** that must be resolved before the Kafka channel can be considered DR-ready.

> **RPO for Kafka:** Same as Notification API (~2 seconds via DynamoDB Global Tables). The Kafka message is published after the notification is processed, so the limiting factor is the notification service failover time.

#### Channel 3: Notification Polling API

**How it works today:**
- Consumer applications poll a REST API endpoint for new data-availability notifications
- API is backed by ECS/Lambda + DynamoDB (stores notification records)
- Consumer filters by dataset, timestamp, etc.

**Multi-Region Considerations:**

This channel has the **simplest DR story** because it already benefits from the platform API failover:

| Component | DR Mechanism | RPO |
|-----------|-------------|-----|
| API Endpoint | Route53 failover to us-west-2 ECS/Lambda | 5-10 min (DNS + scale-up) |
| Notification Data | DynamoDB Global Tables (already replicated) | ~2 seconds |

**What Needs to Change:**

| Step | Action | Owner |
|------|--------|-------|
| 1 | Ensure notification data is stored in DynamoDB (already a Global Table per Section 4.5) | Platform Team (already done) |
| 2 | Deploy notification polling API in us-west-2 (part of ECS/Lambda deployment) | Platform Team (already done) |
| 3 | **Consumers must use the platform CNAME** (e.g., `api.datamesh.company.com`) not a direct regional URL | Consumer |

> **RPO for Notification API:** ~2 seconds (DynamoDB Global Tables replication). This is the most resilient notification channel.

#### Summary: Consumer Notification DR Strategy

| Channel | Multi-Region Approach | RPO | Consumer Effort | Platform Effort |
|---------|----------------------|-----|-----------------|-----------------|
| **SNS-SQS** | Dual-publish to both regional SNS topics; consumer creates SQS in us-west-2 | ~0 (dual-publish) | **Medium** — create SQS, update app to poll both | **Medium** — create SNS, dual-publish logic |
| **On-Prem Kafka** | Establish Direct Connect / VPN from us-west-2 to on-prem; configure Kafka access | ~2 sec | **Low** — no change (Kafka is on-prem) | **High** — DX/VPN provisioning, network validation |
| **Notification API** | Already covered by platform API failover (Route53 + DynamoDB Global Tables) | ~2 sec | **Low** — use CNAME | **Low** — already covered |

**Recommendation by Channel:**

1. **Notification API** — Most resilient, lowest effort. **Encourage consumers to prefer this channel** where possible.
2. **SNS-SQS (Dual-Publish)** — Reliable with proper setup. Requires consumer-side work (SQS in us-west-2, deduplication).
3. **On-Prem Kafka** — Highest risk due to network dependency. **Resolve Direct Connect from us-west-2 as a prerequisite.**

---

## 5. Cross-Account & IAM Considerations

The multi-account nature of this architecture adds significant IAM complexity to multi-region.

### Current IAM Model (us-east-1)

```
Producer Account                    Platform/Compute Account
┌─────────────────────┐            ┌─────────────────────────┐
│ S3 Bucket Policy    │            │ Snowflake IAM Role      │
│ grants access to:   │◄───────────│ (arn:aws:iam::SNOWFLAKE │
│ - Snowflake role    │            │  :role/sf-access-east)  │
│ - Starburst role    │            │                         │
│ - Databricks role   │            │ Starburst IAM Role      │
│                     │◄───────────│ (arn:aws:iam::PLATFORM  │
│                     │            │  :role/starburst-east)  │
└─────────────────────┘            └─────────────────────────┘
```

### What Needs to Change for us-west-2

Each producer must grant access to **new IAM roles** used by the us-west-2 compute platforms:

| Compute Platform | us-east-1 Role (Existing) | us-west-2 Role (New) | Producer Action |
|-----------------|---------------------------|----------------------|-----------------|
| Snowflake | Snowflake's us-east-1 IAM role | Snowflake's us-west-2 IAM role (different account) | Add new role to S3 bucket policy on us-west-2 bucket |
| Starburst | Platform's us-east-1 Starburst role | Platform's us-west-2 Starburst role | Add new role to S3 bucket policy on us-west-2 bucket |
| Databricks | Databricks' us-east-1 IAM role | Databricks' us-west-2 IAM role | Add new role to S3 bucket policy on us-west-2 bucket |

**Important:** IAM roles are global, but instance profiles and trust relationships are region-specific for compute services. Each compute platform in us-west-2 will use different IAM roles than us-east-1.

### Automation Recommendation

Build a **self-service onboarding tool** that:
1. Generates the required IAM policy/bucket policy documents for us-west-2
2. Provides a CloudFormation/Terraform template producers can deploy
3. Validates access after deployment (test read from us-west-2 compute to us-west-2 bucket)

---

## 6. Failover Scenarios & Impact Analysis

### 6.1 Scenario A — Full Failover (All Parties Flip to us-west-2)

**Trigger:** Sustained us-east-1 outage. All teams execute failover runbook.

**Assumptions:**
- All critical producers have CRR enabled to us-west-2
- Data Mesh platform APIs are deployed in us-west-2 (warm standby)
- Compute platforms (Snowflake, Starburst, Databricks) have us-west-2 instances
- Immuta has us-west-2 instance with synced policies
- Glue Catalog in us-west-2 is up to date

```
                    FAILOVER STATE
                    
  Producers          Platform APIs        Compute            Governance
  ─────────          ─────────────        ───────            ──────────
  ✅ us-west-2       ✅ us-west-2         ✅ us-west-2       ✅ us-west-2
     S3 (replica)       ECS/Lambda           SF/SB/DBX          Immuta
     
  Data Flow:
  us-west-2 S3 ──► us-west-2 VPC EP ──► us-west-2 Compute ──► Consumer
                                              │
                                              ▼
                                         us-west-2 Immuta (policy check)
```

| Aspect | Status | Notes |
|--------|--------|-------|
| Data Availability | ✅ Available | Via replicated S3 buckets in us-west-2 |
| Data Freshness | ⚠️ Up to 15 min stale | CRR + RTC lag |
| Registration APIs | ✅ Available | Route53 fails over to us-west-2 ALB |
| Notification APIs | ✅ Available | Lambda in us-west-2 writes to us-west-2 Glue |
| Snowflake Queries | ✅ Available | Using recreated external tables in us-west-2 |
| Starburst Queries | ✅ Available | Using us-west-2 cluster + Glue + S3 |
| Databricks Queries | ✅ Available | Using us-west-2 workspace |
| Policy Enforcement | ✅ Available | Immuta us-west-2 instance |
| Consumer Impact | ⚠️ Minimal | May need to reconnect; 15-min data staleness |
| **RTO** | **~30-60 min** | Time to validate replication + scale up warm standby + validate |

**Failover Steps (Runbook):**

1. **Detect** — Confirm sustained us-east-1 outage (AWS Health Dashboard + internal monitoring)
2. **Decide** — Platform team lead authorizes failover
3. **Scale** — Scale up us-west-2 ECS services, Starburst workers
4. **Validate Data** — Run Iceberg metadata validation job (check data file existence)
5. **Activate Snowflake** — Run external table recreation automation against us-west-2 Snowflake
6. **Activate DNS** — Route53 ARC triggers failover for platform APIs
7. **Activate Notifications** — Verify Notification Service publishes to us-west-2 SNS; confirm on-prem Kafka messages flow via us-west-2 DX; validate consumer SQS queues in us-west-2
8. **Validate** — Run synthetic queries across all three compute platforms
9. **Notify** — Inform consumers that platform is operational in us-west-2
10. **Monitor** — Continuous monitoring for issues, replication lag

---

### 6.2 Scenario B — Critical Producers Flip, Non-Critical Stay

**Trigger:** us-east-1 outage. Critical producers have multi-region; non-critical producers do not.

```
  Critical Producers           Non-Critical Producers
  ──────────────────           ──────────────────────
  ✅ us-west-2 S3 available    ❌ us-east-1 S3 unavailable
     (CRR replicated)             (no replication)
     
  Platform APIs: ✅ us-west-2
  Compute:       ✅ us-west-2
  
  Result:
  ┌────────────────────────────────────────────────────┐
  │ Consumers CAN query datasets from critical         │
  │ producers (data up to 15 min stale)                │
  │                                                    │
  │ Consumers CANNOT query datasets from non-critical  │
  │ producers (S3 unreachable)                         │
  │                                                    │
  │ Queries joining critical + non-critical datasets   │
  │ will FAIL                                          │
  └────────────────────────────────────────────────────┘
```

| Aspect | Impact |
|--------|--------|
| Critical producer datasets | ✅ Available in us-west-2 |
| Non-critical producer datasets | ❌ Unavailable (no S3 replica) |
| Cross-dataset joins | ❌ Fail if they involve non-critical producer data |
| Consumer experience | ⚠️ **Degraded** — partial dataset availability |
| Platform behavior | Must gracefully handle missing datasets (return clear errors) |

**Required Platform Enhancements:**

1. **Dataset criticality tagging** — Metadata in DynamoDB must tag each dataset as Critical/Non-Critical
2. **Graceful degradation in compute** — External tables pointing to unavailable S3 should return meaningful errors, not cryptic S3 access denied
3. **Consumer notification** — Automated notification listing which datasets are available vs. unavailable
4. **Dashboard** — Real-time dashboard showing per-dataset availability status per region

**Recommendation:** Define a **tiered resiliency SLA** for producers:

| Tier | Definition | Multi-Region Requirement |
|------|-----------|-------------------------|
| **Tier 1 — Critical** | Revenue-impacting, regulatory, or SLA-bound datasets | **Mandatory** — CRR, IAM, Access Points in us-west-2 |
| **Tier 2 — Important** | Widely used but not revenue-critical | **Strongly Recommended** — CRR at minimum |
| **Tier 3 — Standard** | Low-usage, internal analytics | **Optional** — accepted unavailability during outage |

---

### 6.3 Scenario C — Critical Producer Refuses to Flip During Resilience Test

**Situation:** During a planned resilience test, most critical producers simulate failover to us-west-2, but **Producer X (critical) decides to stay on us-east-1**.

```
  Producer X: "We're staying on us-east-1"
  
  Test Scenario:
  - Platform APIs: Pointed to us-west-2
  - Compute Platforms: Running in us-west-2
  - Producer X S3: Only in us-east-1
  
  Can us-west-2 compute reach Producer X's us-east-1 S3?
```

**Analysis:**

| Network Path | Works? | Why |
|--------------|--------|-----|
| us-west-2 Starburst → S3 Gateway VPC Endpoint → us-east-1 S3 | ❌ **NO** | Gateway endpoints are same-region only |
| us-west-2 Starburst → S3 Interface Endpoint → us-east-1 S3 | ⚠️ Possible | Requires Interface Endpoint setup + IAM permissions; adds latency and cost |
| us-west-2 Snowflake → us-east-1 S3 | ⚠️ Possible | Snowflake can access cross-region S3 if Storage Integration allows it; but VPC endpoint won't work |
| us-west-2 Databricks → us-east-1 S3 | ⚠️ Possible | Similar to Snowflake — can access cross-region S3 over public internet if allowed |

**Impact:**

- **Starburst:** Producer X's datasets are **UNAVAILABLE** unless S3 Interface Endpoints are configured for cross-region access (expensive, adds latency, complex setup)
- **Snowflake:** May work if Storage Integration is configured for cross-region access, but this bypasses VPC endpoint (traffic goes over Snowflake's managed network, not private VPC)
- **Databricks:** Similar to Snowflake

**Recommendations:**

1. **Establish a contractual obligation** — Critical producers MUST participate in resilience tests. This should be a platform SLA requirement.
2. **If a producer cannot participate:** Accept degraded availability for their datasets during the test and document the risk.
3. **Do NOT engineer complex cross-region access paths** just to accommodate non-participating producers. The cost and complexity outweigh the benefit.
4. **Pre-test validation:** 2 weeks before any resilience test, all critical producers must confirm participation. If any decline, escalate to senior management.

---

### 6.4 Scenario D — Platform Flips, Producers Stay

**Situation:** Platform APIs and compute flip to us-west-2, but us-east-1 is NOT actually down. Producers continue operating in us-east-1.

This can happen during:
- A planned resilience test
- A false alarm that triggers failover
- Platform-specific issues (not regional outage)

```
  Producers: us-east-1 (active, writing new data)
  Platform:  us-west-2 (APIs, Compute)
  
  Data Flow:
  Producer writes to us-east-1 S3
    → CRR replicates to us-west-2 S3 (15 min lag)
      → us-west-2 Compute reads from us-west-2 S3 (stale)
  
  Producer calls Notification API
    → Route53 routes to us-west-2 Lambda
      → Lambda adds partition to us-west-2 Glue ✅
      → Lambda adds partition to us-east-1 Glue ✅ (if dual-write enabled)
```

| Aspect | Status |
|--------|--------|
| Data writes | ✅ Continue in us-east-1 |
| Data availability in us-west-2 | ✅ Available (15 min stale via CRR) |
| Notification API | ✅ Works (Route53 routes to us-west-2 Lambda) |
| Registration API | ✅ Works (Route53 routes to us-west-2 ECS) |
| Consumer queries | ⚠️ **15-minute stale data** — consumers see delayed data |
| Iceberg metadata consistency | ⚠️ Risk — new snapshots in east may not be fully replicated |

**Key Insight:** This scenario is actually the **steady-state during any failover**. The platform must be designed to handle this gracefully. The 15-minute data staleness is the accepted RPO.

---

### 6.5 Scenario E — Immuta Region Failure

**Situation:** Immuta's us-east-1 deployment fails, but everything else is fine.

| Compute Platform | Impact | Mitigation |
|-----------------|--------|------------|
| **Starburst** | ❌ **Queries FAIL** — runtime dependency on Immuta API | Fail over Starburst to us-west-2 cluster connected to us-west-2 Immuta, OR implement local policy cache |
| **Snowflake** | ✅ Continues working with last-synced policies | New policy changes cannot be pushed until Immuta recovers |
| **Databricks** | ✅ Continues working with last-synced policies | Same as Snowflake |

**This is a significant risk for Starburst.** If Immuta's us-east-1 is down but AWS us-east-1 is otherwise healthy, you'd need to either:
1. Fail over ONLY Starburst to us-west-2 (complex partial failover)
2. Implement Immuta policy caching/local enforcement in Starburst (feature request to Immuta/Starburst)
3. Accept Starburst downtime until Immuta recovers

**Recommendation:** Work with Immuta and Starburst teams to implement a **policy cache with configurable staleness window** (e.g., serve cached policies for up to 1 hour if Immuta API is unreachable).

---

### 6.6 Scenario F — DX/VPN to On-Prem Failure

**Trigger:** Direct Connect or VPN link between AWS and on-premises data center goes down, while all AWS services remain healthy.

**Impact:** On-prem Kafka consumers stop receiving notifications. Compute platforms and SNS-SQS are unaffected (they operate entirely within AWS).

| Component | Impact | Notes |
|-----------|--------|-------|
| Snowflake | ✅ Unaffected | Operates entirely within AWS |
| Starburst | ✅ Unaffected | Operates entirely within AWS |
| Databricks | ✅ Unaffected | Operates entirely within AWS |
| SNS-SQS | ✅ Unaffected | AWS-to-AWS communication |
| On-Prem Kafka | ❌ **Messages cannot reach on-prem brokers** | No network path to on-prem |
| Notification API | ✅ Unaffected | AWS-hosted, consumers poll from cloud |

**Mitigation:**
- Kafka consumers should also subscribe to SNS-SQS as a backup notification channel
- Platform should buffer Kafka messages in a dead-letter queue; replay when DX/VPN restores
- Monitor DX/VPN health via CloudWatch; alert on link-down within 1 minute

---

### 6.7 Consumer Impact Matrix

#### Compute Platform Availability

| Scenario | Snowflake | Starburst | Databricks | Consumer Action Required |
|----------|-----------|-----------|------------|-------------------------|
| **A: Full failover** | ⚠️ Reconnect to west account | ⚠️ Reconnect to west cluster | ⚠️ Reconnect to west workspace | Update connection (or use CNAME) |
| **B: Critical only** | ⚠️ Partial datasets | ⚠️ Partial datasets | ⚠️ Partial datasets | Check dataset availability dashboard |
| **C: Producer stays east** | ⚠️ That producer's data unavailable | ❌ That producer's data unavailable | ⚠️ That producer's data unavailable | Adjust queries |
| **D: Platform flips, producers stay** | ✅ Works (15 min stale) | ✅ Works (15 min stale) | ✅ Works (15 min stale) | None (if using CNAME) |
| **E: Immuta down only** | ✅ Works (stale policies) | ❌ Queries fail | ✅ Works (stale policies) | Switch to Snowflake/Databricks |
| **F: DX/VPN down** | ✅ Unaffected | ✅ Unaffected | ✅ Unaffected | None (compute unaffected) |

#### Notification Channel Availability

| Scenario | SNS-SQS (Dual-Publish) | SNS-SQS (Active-Only) | On-Prem Kafka | Notification API |
|----------|------------------------|-----------------------|---------------|------------------|
| **A: Full failover** | ✅ us-west-2 SQS continues | ⚠️ Consumer switches to west SQS | ⚠️ Works if DX from us-west-2 exists | ✅ Route53 failover (CNAME) |
| **B: Critical only** | ✅ Notifications continue for all datasets | ⚠️ Switch SQS | ⚠️ DX dependency | ✅ Works |
| **C: Producer stays east** | ✅ Notifications flow (data may be stale) | ⚠️ Switch SQS | ⚠️ DX dependency | ✅ Works |
| **D: Platform flips, producers stay** | ✅ Seamless (both regions publish) | ⚠️ Switch SQS | ✅ Works if DX from us-west-2 | ✅ Works (CNAME) |
| **E: Immuta down only** | ✅ Unaffected | ✅ Unaffected | ✅ Unaffected | ✅ Unaffected |
| **F: DX/VPN to on-prem down** | ✅ Unaffected | ✅ Unaffected | ❌ **Kafka unreachable** | ✅ Unaffected |

**Consumer Preparation Checklist:**
- [ ] Use platform-provided CNAMEs instead of direct compute platform URLs
- [ ] Test connectivity to us-west-2 compute endpoints
- [ ] Identify queries that join across multiple producers — know which are critical
- [ ] Subscribe to platform status notifications
- [ ] Have runbook for switching connection strings if CNAMEs are not used
- [ ] **(SNS-SQS consumers)** Create SQS queue in us-west-2, subscribe to us-west-2 SNS topic
- [ ] **(SNS-SQS consumers)** Update application to poll both east and west SQS queues (or implement switchover logic)
- [ ] **(SNS-SQS consumers)** Implement idempotent message processing (to handle potential duplicates during failover)
- [ ] **(Notification API consumers)** Ensure polling uses the platform CNAME, not a regional URL
- [ ] **(Kafka consumers)** No consumer-side changes needed — validate that on-prem Kafka remains reachable

---

## 7. Recommended Architecture — Target State

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           TARGET STATE (Multi-Region)                            │
├──────────────────────────────────────┬──────────────────────────────────────────┤
│           us-east-1 (PRIMARY)        │          us-west-2 (SECONDARY)           │
├──────────────────────────────────────┼──────────────────────────────────────────┤
│                                      │                                          │
│  Producer S3 Buckets                 │  Producer S3 Buckets (Replicas)          │
│  (Parquet + Iceberg)                 │  (via S3 CRR + RTC)                     │
│         │                            │         │                                │
│         │ S3 Gateway VPC EP          │         │ S3 Gateway VPC EP             │
│         ▼                            │         ▼                                │
│  ┌─────────────┐                     │  ┌─────────────┐                        │
│  │ Starburst   │ ◄─Glue─►           │  │ Starburst   │ ◄─Glue─►              │
│  │ Cluster     │  Catalog            │  │ Cluster     │  Catalog               │
│  └─────────────┘  (east)             │  └─────────────┘  (west)                │
│                                      │                                          │
│  ┌─────────────┐                     │  ┌─────────────┐                        │
│  │ Snowflake   │ Storage             │  │ Snowflake   │ Storage                │
│  │ Account     │ Integration         │  │ Account     │ Integration            │
│  │ (east)      │ (east)              │  │ (west)      │ (west)                 │
│  └─────────────┘                     │  └─────────────┘                        │
│                                      │                                          │
│  ┌─────────────┐                     │  ┌─────────────┐                        │
│  │ Databricks  │                     │  │ Databricks  │                        │
│  │ Workspace   │                     │  │ Workspace   │                        │
│  │ (east)      │                     │  │ (west)      │                        │
│  └─────────────┘                     │  └─────────────┘                        │
│                                      │                                          │
│  ┌──────────────────────┐            │  ┌──────────────────────┐               │
│  │ Data Mesh APIs       │            │  │ Data Mesh APIs       │               │
│  │ ECS + Lambda         │            │  │ ECS + Lambda         │               │
│  │ DynamoDB ◄═══════════╪════════════╪══► DynamoDB             │               │
│  │ (Global Tables)      │            │  │ (Global Tables)      │               │
│  └──────────────────────┘            │  └──────────────────────┘               │
│                                      │                                          │
│  ┌──────────────────────┐            │  ┌──────────────────────┐               │
│  │ Immuta (east)        │            │  │ Immuta (west)        │               │
│  │ Policy Engine        │◄──Git──────┼──► Policy Engine       │               │
│  └──────────────────────┘  sync      │  └──────────────────────┘               │
│                                      │                                          │
│  Route53: api.datamesh.company.com ──┼──► Failover routing (health checks)     │
│           sf.datamesh.company.com  ──┼──► Client redirect                      │
│           sb.datamesh.company.com  ──┼──► DNS failover                         │
│                                      │                                          │
├──────────────────────────────────────┴──────────────────────────────────────────┤
│                              REPLICATION FLOWS                                  │
│                                                                                 │
│  S3 CRR + RTC ─────────────────────────────► (RPO: 15 min)                     │
│  DynamoDB Global Tables ────────────────────► (RPO: ~2 sec)                     │
│  Glue Catalog (dual-write by Notification API) ► (RPO: ~0 if sync; seconds if async) │
│  Immuta Policies (Git-based Policy-as-Code) ─► (RPO: last CI/CD run)           │
│  Snowflake Internal Objects (replication groups) ► (RPO: configurable)          │
│  Snowflake External Tables (automation recreation) ► (RTO: 10-15 min)          │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Action Plan by Team

### 8.1 Producer Teams

| # | Action | Priority | Complexity | Dependency | Deliverable |
|---|--------|----------|------------|------------|-------------|
| P1 | Create S3 bucket in us-west-2 with matching prefix structure | **P0** | Low | None | Bucket ARN |
| P2 | Enable S3 CRR + RTC from us-east-1 → us-west-2 bucket | **P0** | Low | P1 | Replication config |
| P3 | Configure bucket policy on us-west-2 bucket granting access to compute platform IAM roles | **P0** | Medium | P1, Platform provides role ARNs | Updated bucket policy |
| P4 | Create S3 Access Points on us-west-2 bucket (for Starburst) | **P1** | Low | P1 | Access Point ARNs and Aliases |
| P5 | **(Iceberg only)** Validate Iceberg metadata replication — confirm metadata.json files are replicated | **P0** | Low | P2 | Validation report |
| P6 | **(Iceberg only)** Evaluate MRAP adoption for new tables | **P2** | Medium | Platform provides guidance | Decision documented |
| P7 | Monitor CRR replication lag via CloudWatch | **P1** | Low | P2 | CloudWatch dashboard |
| P8 | Participate in failover drill (quarterly) | **P1** | Low | All above complete | Drill results |

**Template CloudFormation/Terraform provided by Platform Team:** A ready-to-deploy template that creates P1-P4 with parameterized inputs.

---

### 8.2 Data Mesh Platform Team

| # | Action | Priority | Complexity | Dependency | Deliverable |
|---|--------|----------|------------|------------|-------------|
| M1 | Convert DynamoDB tables to Global Tables (add us-west-2 replica) | **P0** | Medium | None | Global Table enabled |
| M2 | Deploy ECS services in us-west-2 (warm standby) | **P0** | High | VPC/networking in us-west-2 | ECS cluster running |
| M3 | Deploy Lambda functions in us-west-2 | **P0** | Medium | M2 (shared VPC) | Lambda functions deployed |
| M4 | Set up Route53 failover routing with health checks | **P0** | Medium | M2, M3 | Failover DNS configured |
| M5 | **Modify Notification API for Glue Catalog dual-write** (us-east-1 + us-west-2) | **P0** | High | M3 | Notification API updated |
| M6 | Set up us-west-2 Glue Catalog with initial table sync | **P0** | High | Producer CRR (P2) | Glue Catalog populated |
| M7 | Set up ECR cross-region replication for container images | **P1** | Low | None | ECR replication enabled |
| M8 | **Build Snowflake external table recreation automation** | **P0** | Very High | M1, Snowflake us-west-2 account | Automation pipeline |
| M9 | Provision Snowflake account in us-west-2 | **P0** | Medium | None | Snowflake account |
| M10 | Create Storage Integrations + External Volumes in us-west-2 Snowflake | **P0** | High | M9, Producer IAM (P3) | Storage configured |
| M11 | Configure Snowflake account replication for internal objects | **P1** | Medium | M9 | Replication group |
| M12 | Set up Snowflake Client Redirect or DNS CNAME | **P1** | Medium | M9 | Consumer routing |
| M13 | Deploy Starburst cluster in us-west-2 (warm standby) | **P0** | High | VPC/networking | Starburst cluster |
| M14 | Configure us-west-2 Starburst: Glue Catalog, S3 Access Points, IAM | **P0** | High | M6, M13, Producer (P4) | Starburst configured |
| M15 | Deploy Databricks workspace in us-west-2 | **P1** | Medium | None | Workspace provisioned |
| M16 | Build Databricks external table recreation automation | **P1** | High | M15 | Automation pipeline |
| M17 | **Build Iceberg failover automation** (register_table + path validation) | **P0** | Very High | M6 | Iceberg failover scripts |
| M18 | Create producer onboarding template (CloudFormation/Terraform) for us-west-2 | **P1** | Medium | None | Template published |
| M19 | Build dataset availability dashboard | **P1** | Medium | M1 | Dashboard |
| M20 | **Build comprehensive failover runbook** | **P0** | Medium | All above | Runbook document |
| M21 | Set up monitoring & alerting for all us-west-2 components | **P1** | Medium | All above | CloudWatch alarms |
| M22 | Conduct quarterly failover drills | **P1** | High | All above | Drill reports |
| M23 | Modify Registration API to create tables in both regions | **P1** | High | M6, M8, M14 | API updated |
| M24 | **Create SNS topic in us-west-2** and configure dual-publish from Notification Service | **P0** | Medium | M3 | SNS topic + dual-publish |
| M25 | **Establish Direct Connect / VPN from us-west-2 to on-prem** for Kafka connectivity | **P0** | Very High | Network Team dependency | DX/VPN active, Kafka reachable |
| M26 | Configure us-west-2 Lambda/ECS with on-prem Kafka broker endpoints and credentials | **P1** | Medium | M25 | Kafka producer config |
| M27 | Validate end-to-end Kafka message flow from us-west-2 to on-prem consumers | **P1** | Medium | M25, M26 | Validation report |
| M28 | Provide consumer onboarding guide for us-west-2 SQS creation and SNS subscription | **P1** | Low | M24 | Guide published |

---

### 8.3 Governance (Immuta) Team

| # | Action | Priority | Complexity | Dependency | Deliverable |
|---|--------|----------|------------|------------|-------------|
| G1 | Deploy Immuta instance in us-west-2 (or coordinate SaaS DR) | **P0** | High | None | Immuta instance running |
| G2 | Implement Policy-as-Code pipeline (Git → Immuta CLI → both regions) | **P0** | High | G1 | CI/CD pipeline |
| G3 | Configure us-west-2 Immuta integrations (Starburst, Snowflake, Databricks) | **P0** | Medium | G1, Platform M9/M13/M15 | Integrations configured |
| G4 | Validate policy consistency between us-east-1 and us-west-2 instances | **P1** | Medium | G2, G3 | Validation report |
| G5 | Investigate Starburst policy caching for Immuta API unavailability | **P2** | Medium | Starburst/Immuta vendor engagement | Feature request or workaround |
| G6 | Participate in failover drills | **P1** | Low | G1-G4 | Drill results |

---

### 8.4 Consumer Teams

| # | Action | Priority | Complexity | Dependency | Deliverable |
|---|--------|----------|------------|------------|-------------|
| C1 | Switch to platform-provided CNAMEs for compute connections | **P1** | Low | Platform M12 | Updated connection configs |
| C2 | Test connectivity to us-west-2 compute endpoints | **P1** | Low | Platform M9/M13/M15 | Validation results |
| C3 | Identify critical query dependencies (which producers, which datasets) | **P1** | Low | None | Dependency map |
| C4 | Subscribe to platform status notifications | **P1** | Low | Platform M19 | Subscribed |
| C5 | Prepare connection failover runbook (for manual switch if needed) | **P2** | Low | C1 | Runbook |
| C6 | Participate in failover drills | **P1** | Low | All above | Drill results |
| C7 | **(SNS-SQS consumers)** Create SQS queue in us-west-2; subscribe to us-west-2 SNS topic | **P0** | Medium | Platform M24 | SQS queue + subscription |
| C8 | **(SNS-SQS consumers)** Update application to poll both east and west SQS queues | **P1** | Medium | C7 | Updated application code |
| C9 | **(SNS-SQS consumers)** Implement idempotent message processing (deduplication) | **P1** | Medium | C8 | Deduplication logic |
| C10 | **(Notification API consumers)** Ensure polling uses platform CNAME, not regional URL | **P1** | Low | Platform M4 | Updated config |
| C11 | **(Kafka consumers)** Validate on-prem Kafka remains reachable — no consumer-side changes | **P2** | Low | Platform M25 | Validation |

---

## 9. Implementation Phases & Timeline

### Phase 1: Foundation (Weeks 1–6)

**Goal:** Core infrastructure in us-west-2, data replication flowing

| Week | Milestone | Key Actions |
|------|-----------|-------------|
| 1-2 | Platform Infrastructure | M1 (DynamoDB Global Tables), M7 (ECR replication), M9 (Snowflake us-west-2), VPC/networking in us-west-2 |
| 2-3 | Producer Onboarding Starts | M18 (Template published), Critical Tier 1 producers begin P1-P4 |
| 3-4 | APIs Deployed in us-west-2 | M2 (ECS), M3 (Lambda), M4 (Route53 failover) |
| 4-6 | Glue Catalog Dual-Write | M5 (Notification API), M6 (Glue Catalog sync) |
| 1-2 | **DX/VPN Provisioning Start** | **M25 (Direct Connect from us-west-2 to on-prem) — 4-8 week lead time, order immediately** |
| 3-4 | SNS Dual-Publish | M24 (SNS topic in us-west-2, dual-publish from Notification Service) |

**Exit Criteria:** DynamoDB replicating, APIs running in us-west-2, Tier 1 producers have CRR enabled, Glue Catalog dual-write active, DX provisioning ordered.

### Phase 2: Compute Platform DR (Weeks 5–12)

**Goal:** All three compute platforms operational in us-west-2

| Week | Milestone | Key Actions |
|------|-----------|-------------|
| 5-7 | Snowflake DR | M10 (Storage Integrations), M11 (Replication groups), M8 (External table automation) |
| 6-8 | Starburst DR | M13 (Cluster), M14 (Configuration) |
| 7-10 | Databricks DR | M15 (Workspace), M16 (External table automation) |
| 8-10 | Iceberg Automation | M17 (Iceberg failover automation — register_table + validation) |
| 10-12 | Consumer Routing | M12 (Snowflake redirect), DNS for Starburst/Databricks |
| 8-10 | Kafka DR Validation | M26 (Kafka config for us-west-2 DX), M27 (End-to-end Kafka validation) |
| 10-12 | Consumer Notification Onboarding | M28 (Consumer onboarding guide), C7-C9 (SQS setup), C10-C11 (API/Kafka validation) |

**Exit Criteria:** All compute platforms query-able in us-west-2 using replicated data. Automated table recreation tested. Notification channels validated.

### Phase 3: Governance & Validation (Weeks 10–16)

**Goal:** Immuta in us-west-2, full stack validated

| Week | Milestone | Key Actions |
|------|-----------|-------------|
| 10-12 | Immuta DR | G1 (Deploy), G2 (Policy-as-Code), G3 (Integrations) |
| 12-14 | End-to-End Validation | Synthetic queries across all platforms, policy enforcement verified |
| 14-16 | Failover Drill #1 | Full failover simulation with critical producers |

**Exit Criteria:** Immuta policies enforced in us-west-2. First successful failover drill completed.

### Phase 4: Hardening & Operational Readiness (Weeks 16–20)

**Goal:** Production-ready DR with ongoing operational processes

| Week | Milestone | Key Actions |
|------|-----------|-------------|
| 16-17 | Monitoring & Alerting | M21 (CloudWatch), dashboards, PagerDuty integration |
| 17-18 | Runbook Finalization | M20 (Runbook), consumer communication templates |
| 18-19 | Failover Drill #2 | Include Tier 2 producers, consumer participation |
| 19-20 | Sign-off | Architecture review, management approval, quarterly drill schedule |

**Exit Criteria:** Runbook approved, monitoring active, drill cadence established, all Tier 1 producers onboarded.

---

## 10. Blockers, Risks & Mitigations

### Blockers

| # | Blocker | Impact | Owner | Mitigation | Status |
|---|---------|--------|-------|------------|--------|
| B1 | **Snowflake cannot replicate unmanaged Iceberg tables** | Must build custom automation for table recreation | Platform Team | Build recreation pipeline sourced from DynamoDB metadata | 🔴 Open |
| B2 | **Iceberg metadata absolute paths** | Replicated tables unreadable without path resolution | Platform Team | register_table automation + path substitution; MRAP for new tables | 🔴 Open |
| B3 | **Producer adoption dependency** | Platform team cannot force producers to enable CRR | Platform Team + Management | Mandate for Tier 1; provide self-service templates; escalation path | 🔴 Open |
| B4 | **Immuta cross-region licensing/deployment** | May require additional license or SaaS contract change | Governance Team | Engage Immuta vendor early (Week 1) | 🔴 Open |
| B5 | **Starburst runtime dependency on Immuta** | No policy cache = query failure if Immuta unavailable | Governance + Platform | Vendor engagement for policy caching feature | 🔴 Open |
| B6 | **Snowflake Business Critical edition required** for failover groups | May require contract upgrade | Platform Team | Verify current edition; budget for upgrade if needed | 🟡 Investigating |
| B7 | **On-prem Kafka: no Direct Connect / VPN from us-west-2** | Kafka notification channel completely unavailable during us-east-1 outage | Platform + Network Team | Provision DX/VPN from us-west-2 to on-prem (long lead time: 4-8 weeks for DX) | 🔴 Open |
| B8 | **SNS-SQS: consumer SQS queues only in us-east-1** | Consumers cannot receive notifications during outage until they create us-west-2 SQS | Consumer Teams | Mandate SQS in us-west-2 for Tier 1 consumers; provide onboarding guide | 🔴 Open |

### Risks

| # | Risk | Probability | Impact | Mitigation |
|---|------|-------------|--------|------------|
| R1 | S3 CRR replication lag exceeds 15 min during high-volume periods | Medium | Data staleness beyond RPO target | Monitor `ReplicationLatency`; alert at 10 min; consider S3 batch replication for backfill |
| R2 | Iceberg metadata/data ordering race condition during failover | High | Transient query failures (FileNotFoundException) | Validation job before enabling consumer access; staleness buffer |
| R3 | DynamoDB LWW conflicts during failback (dual-region writes) | Low | Silent data loss in metadata store | Region-partitioned write pattern; reconciliation script for failback |
| R4 | Producer CRR misconfiguration (wrong prefix, missing permissions) | Medium | Incomplete data in us-west-2 | Automated validation checks; onboarding checklist |
| R5 | Snowflake external table recreation takes too long | Medium | Extended RTO beyond target | Pre-create tables nightly; parallelize DDL execution |
| R6 | Consumer connection strings hardcoded to us-east-1 endpoints | High | Extended consumer RTO | Mandate CNAME usage; publish migration guide early |
| R7 | Cost overrun from running warm standby in us-west-2 | Medium | Budget concerns | Right-size warm standby; use spot instances for Starburst workers; scale down aggressively |

### Risk Heat Map

```
            │ Low Impact  │ Medium Impact │ High Impact  │
────────────┼─────────────┼───────────────┼──────────────┤
High Prob.  │             │ R2, R6        │              │
────────────┼─────────────┼───────────────┼──────────────┤
Med. Prob.  │             │ R1, R4, R5, R7│              │
────────────┼─────────────┼───────────────┼──────────────┤
Low Prob.   │             │ R3            │              │
────────────┴─────────────┴───────────────┴──────────────┘
```

---

## 11. RPO / RTO Summary

### Per-Component RPO & RTO

| Component | RPO | RTO | Notes |
|-----------|-----|-----|-------|
| **S3 Data (Parquet)** | 15 min (CRR + RTC) | 0 min (pre-replicated) | Data available immediately if CRR is healthy |
| **S3 Data (Iceberg)** | 15 min (CRR + RTC) | 10–15 min (metadata registration) | Requires validation + register_table |
| **DynamoDB Metadata** | ~2 sec (Global Tables) | 0 min (active replica) | Immediately available |
| **Glue Catalog** | ~0 (dual-write) | 0 min (already populated) | Dual-write keeps west in sync |
| **Data Mesh APIs** | ~2 sec (matches DynamoDB) | 5–10 min (ECS scale-up + DNS) | Route53 TTL + ECS task start |
| **Snowflake (Internal)** | Configurable (replication) | 5 min (promote secondary) | Business Critical required |
| **Snowflake (External/Iceberg Tables)** | N/A (recreated, not replicated) | 10–20 min (automation run) | **Bottleneck** — depends on table count |
| **Starburst** | ~0 (Glue dual-write) | 5–10 min (cluster scale-up) | Already configured against west Glue |
| **Databricks** | N/A (recreated) | 10–15 min (automation run) | Similar to Snowflake |
| **Immuta** | Last CI/CD run (Policy-as-Code) | 5–10 min (activate integrations) | Policies pre-synced |
| **Notification: SNS-SQS (Dual-Publish)** | ~0 (dual-publish) | 0 min (already active in both regions) | Consumer must poll both SQS queues |
| **Notification: SNS-SQS (Active-Only)** | Time to last publish before outage | 5-10 min (DNS failover + consumer SQS switch) | Consumer must detect and switch |
| **Notification: On-Prem Kafka** | ~2 sec (matches Notification API) | 5-10 min (Lambda failover + DX path) | **Blocked** if no DX/VPN from us-west-2 |
| **Notification: Polling API** | ~2 sec (DynamoDB Global Tables) | 5-10 min (Route53 + ECS scale-up) | Already covered by platform API failover |

### Overall Platform

| Metric | Target | Expected Actual | Bottleneck |
|--------|--------|-----------------|------------|
| **RPO** | ≤ 15 min | ~15 min | S3 CRR replication lag |
| **RTO** | ≤ 60 min | 45–60 min (with parallelism) | Snowflake external table recreation + Iceberg validation |

---

## 12. Decision Log

Decisions requiring senior architecture / management approval:

| # | Decision | Options | Recommendation | Rationale | Status |
|---|----------|---------|----------------|-----------|--------|
| D1 | Failover strategy | Active-Active vs. Active-Passive | **Active-Passive (Warm Standby)** | Cost-effective; complexity manageable; RPO/RTO targets achievable | ⬜ Pending |
| D2 | Producer multi-region mandate | Mandatory for all vs. Tiered | **Tiered (Tier 1 mandatory, Tier 2 recommended, Tier 3 optional)** | Pragmatic; not all datasets warrant DR cost | ⬜ Pending |
| D3 | Iceberg path strategy (new tables) | MRAP vs. Keep absolute paths | **MRAP for new tables** (if compute platform compatibility confirmed) | Eliminates metadata path problem long-term | ⬜ Pending |
| D4 | Iceberg path strategy (existing tables) | Migrate to MRAP vs. Failover-time re-registration | **Failover-time re-registration** | Lower disruption; migration is high-effort with uncertain ROI | ⬜ Pending |
| D5 | Snowflake edition | Current vs. Business Critical | **Upgrade to Business Critical** (if not already) | Required for failover groups + Client Redirect | ⬜ Pending |
| D6 | Glue Catalog sync strategy | Replication utility vs. Dual-write | **Dual-write from Notification API** | Real-time; no additional infrastructure; simpler | ⬜ Pending |
| D7 | Immuta deployment model | SaaS DR vs. Self-managed in us-west-2 | **Depends on current deployment** — engage vendor | Need to confirm SaaS DR coverage for us-west-2 | ⬜ Pending |
| D8 | Failover trigger | Automatic vs. Manual | **Manual** (with automated detection + alerting) | Avoids split-brain; human judgment for major decision | ⬜ Pending |
| D9 | Consumer connection strategy | CNAME vs. Client Redirect vs. Manual | **CNAME (Route53)** for Starburst/Databricks; **Client Redirect** for Snowflake | Transparent to consumers; minimal reconfiguration | ⬜ Pending |
| D10 | Quarterly drill mandate | Optional vs. Mandatory | **Mandatory for Tier 1 producers and all platform teams** | Only way to validate DR readiness | ⬜ Pending |
| D11 | SNS-SQS notification DR strategy | Dual-Publish (both regions) vs. Active-only | **Dual-Publish** — platform publishes to both regional SNS topics; consumers poll both SQS | Zero consumer action during failover; slightly higher cost | ⬜ Pending |
| D12 | On-prem Kafka DR connectivity | Provision DX from us-west-2 vs. Cross-region TGW vs. Accept unavailability | **Provision Direct Connect from us-west-2** (if Kafka consumers are Tier 1) | Only reliable option; TGW depends on us-east-1 networking | ⬜ Pending |

---

## 13. Appendix

### A. Glossary

| Term | Definition |
|------|-----------|
| **CRR** | S3 Cross-Region Replication — asynchronous replication of S3 objects across regions |
| **RTC** | Replication Time Control — SLA guaranteeing 99.99% of objects replicate within 15 minutes |
| **MRAP** | S3 Multi-Region Access Point — region-agnostic S3 endpoint that routes to nearest bucket |
| **RPO** | Recovery Point Objective — maximum acceptable data loss measured in time |
| **RTO** | Recovery Time Objective — maximum acceptable downtime |
| **LWW** | Last Writer Wins — DynamoDB Global Tables conflict resolution strategy |
| **MREC** | Multi-Region Eventual Consistency (DynamoDB) |
| **MRSC** | Multi-Region Strong Consistency (DynamoDB) |
| **ARC** | Route53 Application Recovery Controller — coordinated failover service |

### B. Producer Onboarding Checklist for Multi-Region

```
Pre-requisites:
  □ AWS account has us-west-2 region enabled
  □ Received compute platform IAM role ARNs for us-west-2 from Platform Team

S3 Setup:
  □ Created S3 bucket in us-west-2: s3://<producer>-data-us-west-2/
  □ Enabled versioning on both source and destination buckets
  □ Configured S3 CRR rule: us-east-1 → us-west-2
  □ Enabled Replication Time Control (RTC)
  □ Enabled Replication Metrics
  □ Verified: first objects replicated successfully

IAM / Access:
  □ Updated us-west-2 bucket policy to grant access to:
    - Snowflake us-west-2 IAM role: arn:aws:iam::<SF_ACCOUNT>:role/<ROLE>
    - Starburst us-west-2 IAM role: arn:aws:iam::<PLATFORM_ACCOUNT>:role/<ROLE>
    - Databricks us-west-2 IAM role: arn:aws:iam::<DBX_ACCOUNT>:role/<ROLE>
  □ Created S3 Access Point on us-west-2 bucket (for Starburst)
  □ Shared Access Point ARN and Alias with Platform Team

Iceberg-Specific (if applicable):
  □ Verified Iceberg metadata files are replicating
  □ Confirmed metadata.json is accessible in us-west-2 bucket
  □ (Optional) Evaluated MRAP adoption for new tables

Validation:
  □ CRR replication lag < 15 minutes (verified via CloudWatch)
  □ Platform Team confirmed successful read from us-west-2 bucket
  □ Participated in failover drill

Monitoring:
  □ CloudWatch alarm on ReplicationLatency > 10 minutes
  □ CloudWatch alarm on CRR FailedReplication count > 0
```

### C. Failover Runbook (Summary)

```
FAILOVER RUNBOOK: us-east-1 → us-west-2
========================================

TRIGGER: Confirmed sustained us-east-1 outage (>15 minutes)
AUTHORITY: Platform Team Lead + VP Engineering approval

Step 1: ASSESS (5 min)
  - Confirm us-east-1 outage via AWS Health Dashboard
  - Check S3 CRR replication status (last successful replication time)
  - Identify which producers have replicated data in us-west-2
  - Get verbal approval from VP Engineering

Step 2: ACTIVATE PLATFORM APIs (5-10 min)
  - Route53 ARC: Activate us-west-2 readiness check
  - Verify DynamoDB Global Table is serving from us-west-2
  - Scale up ECS services in us-west-2 (desired count → production level)
  - Verify Lambda functions in us-west-2 are responding
  - Validate: Hit /health endpoint on us-west-2 ALB

Step 3: ACTIVATE ICEBERG TABLES (10-15 min)
  - Run Iceberg validation job:
    - For each Iceberg dataset, check latest metadata.json in us-west-2
    - Verify all referenced data files exist in us-west-2 bucket
    - If files missing: use previous snapshot (staleness buffer)
  - Run register_table for each Iceberg table in us-west-2 Glue Catalog

Step 4: ACTIVATE SNOWFLAKE (10-20 min)
  - Promote us-west-2 Snowflake account (if using failover groups)
  - Run external table recreation automation
  - Validate: SELECT COUNT(*) on sample tables
  - Activate Snowflake Client Redirect (if configured)

Step 5: ACTIVATE STARBURST (5-10 min)
  - Scale up us-west-2 Starburst worker nodes
  - Verify Glue Catalog connectivity
  - Verify Immuta API connectivity
  - Validate: Run sample queries
  - Update DNS to point to us-west-2 Starburst

Step 6: ACTIVATE DATABRICKS (10-15 min)
  - Run external table recreation automation
  - Validate: Run sample queries
  - Update DNS/notify consumers

Step 7: ACTIVATE NOTIFICATION CHANNELS (5 min) [parallel with Step 6]
  - Verify Notification Service is publishing to us-west-2 SNS topic
  - Confirm on-prem Kafka messages flowing via us-west-2 DX/VPN
  - Validate consumer SQS queues in us-west-2 are receiving messages
  - Verify Notification API is reachable via Route53 CNAME

Step 8: VALIDATE GOVERNANCE (5 min)
  - Confirm Immuta us-west-2 is reachable
  - Run sample query as restricted user — verify policy enforcement
  - Confirm Snowflake/Databricks policies are in effect

Step 9: NOTIFY CONSUMERS (5 min)
  - Send status notification: Platform operational in us-west-2
  - Include: list of available datasets, known limitations, data staleness
  - Include: list of unavailable datasets (non-replicated producers)
  - Publish dataset availability dashboard link

Step 10: MONITOR
  - Continuous monitoring of us-west-2 platform health
  - Monitor S3 replication metrics for data freshness
  - Watch for us-east-1 recovery signals

NOTE: Steps 3-6 run partially in parallel (Iceberg + Snowflake can overlap
with Starburst + Databricks). Step 7 runs in parallel with Step 6.

TOTAL ESTIMATED RTO: 45-60 minutes (with parallelism)
  - Best case:  ~45 min (Steps 3+4 parallel with 5+6, all succeed quickly)
  - Worst case: ~85 min (all sequential, max durations)
```

### D. Failback Runbook (Summary)

```
FAILBACK RUNBOOK: us-west-2 → us-east-1
========================================

TRIGGER: us-east-1 confirmed stable for >2 hours
AUTHORITY: Platform Team Lead + VP Engineering approval

WARNING: Failback is MORE dangerous than failover.
Data may have been written to us-west-2 during the outage that has
not been replicated back to us-east-1.

Step 1: VERIFY us-east-1 STABILITY (2+ hours)
  - AWS Health Dashboard shows all clear
  - Test S3, DynamoDB, Glue, ECS, Lambda in us-east-1

Step 2: REVERSE REPLICATION CHECK
  - Verify forward CRR resumed (us-east-1 → us-west-2 replication catching up)
  - Check if any data was written directly to us-west-2 during outage
  - DynamoDB: Check for write conflicts (Global Tables reconciliation)
  NOTE: S3 CRR is configured one-way (east → west). Any data written to
  us-west-2 during outage requires manual reverse sync (Step 3).

Step 3: SYNC DATA (if needed)
  - If producers wrote to us-west-2 during outage:
    run `aws s3 sync s3://bucket-us-west-2/ s3://bucket-us-east-1/` for affected buckets
    (S3 CRR does NOT replicate in the reverse direction automatically)
  - Verify Glue Catalog in us-east-1 is up to date
  - Verify DynamoDB Global Tables have reconciled all conflicts

Step 4: GRADUAL TRAFFIC SHIFT
  - Route 10% of traffic to us-east-1, monitor
  - Route 50% of traffic to us-east-1, monitor
  - Route 100% of traffic to us-east-1

Step 5: DEACTIVATE us-west-2
  - Scale down us-west-2 ECS to warm standby
  - Scale down us-west-2 Starburst to minimum
  - Demote us-west-2 Snowflake back to secondary
  - Verify all consumers are on us-east-1

Step 6: POST-INCIDENT REVIEW
  - Document timeline, issues encountered
  - Update runbook based on learnings
  - Schedule remediation items
```

### E. Cost Estimate (Monthly, Order of Magnitude)

| Component | Warm Standby Cost | Full Active Cost | Notes |
|-----------|-------------------|------------------|-------|
| S3 CRR Storage | $X per TB replicated | Same | Depends on data volume |
| S3 CRR Transfer | $0.02/GB × monthly change volume | Same | One-time backfill cost higher |
| DynamoDB Global Tables | ~2× write costs | Same | Replicated write units |
| ECS Warm Standby | ~20% of primary | 100% of primary | Minimum task count |
| Starburst Warm Standby | ~15-20% of primary | 100% of primary | Minimum worker nodes |
| Snowflake Account | Credit cost for automation runs | Full credit usage | Warehouses suspended when idle |
| Databricks Workspace | Minimal (control plane) | Full cluster costs | Clusters terminated when idle |
| Immuta | License cost (TBD with vendor) | Same | May require additional license |
| **Estimated Total** | **~25-35% of primary region cost** | **~100% of primary** | Warm standby is cost-efficient |

---

*This document was prepared for architecture review. All decisions marked "Pending" require sign-off before implementation begins.*

*Last updated: April 9, 2026*
