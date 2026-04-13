# Data Mesh Platform — Resiliency Strategy: Ideal & Hybrid Architecture

**Version:** 2.0  
**Date:** April 14, 2026  
**Status:** Draft — For Architecture Review  
**Target Primary Region:** us-west-2 (Oregon)  
**Secondary / Failback Region:** us-east-1 (N. Virginia)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current Architecture Recap](#2-current-architecture-recap)
3. [Ideal State Architecture](#3-ideal-state-architecture)
4. [Hybrid State Architecture](#4-hybrid-state-architecture)
5. [Tasks for Ideal State](#5-tasks-for-ideal-state)
6. [Tasks for Hybrid State](#6-tasks-for-hybrid-state)
7. [Comparison: Ideal vs Hybrid](#7-comparison-ideal-vs-hybrid)
8. [Snowflake Private Listing / SnowGrid Deep Dive](#8-snowflake-private-listing--snowgrid-deep-dive)
9. [Migration Sequencing](#9-migration-sequencing)
10. [Risks & Mitigations](#10-risks--mitigations)

---

## 1. Executive Summary

This document presents the **updated resiliency strategy** for the Data Mesh platform. The approach has shifted from a disaster-recovery (warm standby) model to a **full regional flip** where **us-west-2 becomes the new primary region**.

Two target states are defined:

### Ideal State

All producers, the Data Mesh platform, and all AWS consumers **flip to us-west-2 simultaneously**.

- **Producers** reverse their S3 replication direction (east1→west2 becomes west2→east1) and begin writing directly to us-west-2
- **Data Mesh Platform** deploys all infrastructure in us-west-2: Producer Notification API, Consumer Notification API, SNS, Registration API, Lambda functions, DynamoDB
- **Consumers** deploy infrastructure in us-west-2 (SQS queues, compute platform connections) and operate entirely from the new primary region

### Hybrid State

Most producers flip to us-west-2, but **a few cannot flip**. For those non-flipping producers:

- Their data remains in us-east-1 S3 buckets
- **Snowflake Private Listing (or SnowGrid)** is used to replicate unmanaged Iceberg tables from the us-east-1 Snowflake account to the us-west-2 Snowflake account
- Data Mesh platform and consumers operate in us-west-2
- Consumers access non-flipped producer data through the us-west-2 Snowflake account (via Private Listing)

### Why This Change?

The original DR approach (active-passive warm standby with failover) had significant complexity:
- Iceberg metadata absolute path problem required failover-time re-registration (eliminated in Ideal State; sidestepped by Private Listing in Hybrid State for non-flipped producers)
- Snowflake external/Iceberg tables could not be replicated
- Partial producer adoption created complex cross-region access scenarios
- Failover was reactive — triggered only during outages

The new approach **proactively moves to us-west-2**, eliminating the Iceberg path problem for flipped producers and simplifying the architecture by making us-west-2 the steady-state primary.

---

## 2. Current Architecture Recap

All components operate exclusively in **us-east-1**:

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         CURRENT STATE (us-east-1 Only)                           │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                           │
│  │ Producer A   │  │ Producer B   │  │ Producer C   │  ... (N producers)        │
│  │ (Own Account)│  │ (Own Account)│  │ (Own Account)│                           │
│  │  S3 Bucket   │  │  S3 Bucket   │  │  S3 Bucket   │                           │
│  │  (Parquet)   │  │  (Iceberg)   │  │  (Parquet)   │                           │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                           │
│         │                 │                 │                                    │
│         │  Register Dataset / Notify New Data                                   │
│         ▼                 ▼                 ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐            │
│  │           Data Mesh Platform (Platform Account, us-east-1)      │            │
│  │                                                                  │            │
│  │  Route53 ──► ALB ──► ECS (Registration APIs)                    │            │
│  │                                                                  │            │
│  │  Lambda: Producer Notification API                               │            │
│  │    ├── Receives "new data available" from producers              │            │
│  │    ├── Writes metadata to DynamoDB                               │            │
│  │    └── Updates Glue Catalog partitions                           │            │
│  │                                                                  │            │
│  │  Lambda: Consumer Notification API                               │            │
│  │    ├── Publishes to SNS (us-east-1) → Consumer SQS              │            │
│  │    ├── Pushes to On-Prem Kafka via DX/VPN                       │            │
│  │    └── Stores notification in DynamoDB for polling               │            │
│  │                                                                  │            │
│  │  DynamoDB (Metadata Store, us-east-1)                            │            │
│  │  SNS Topic (us-east-1)                                           │            │
│  └──────────┬──────────────┬───────────────┬────────────────────────┘            │
│             │              │               │                                     │
│             ▼              ▼               ▼                                     │
│  ┌───────────────┐ ┌──────────────┐ ┌──────────────┐                            │
│  │  Snowflake    │ │  Starburst   │ │  Databricks  │                            │
│  │  (us-east-1)  │ │  (us-east-1) │ │  (us-east-1) │                            │
│  │               │ │              │ │              │                            │
│  │ External Tbls │ │ Glue Catalog │ │ External Tbls│                            │
│  │ Unmanaged     │ │ S3 Access    │ │ Unity Catalog│                            │
│  │ Iceberg Tbls  │ │ Point Alias  │ │              │                            │
│  │ Storage Integ.│ │              │ │              │                            │
│  └───────┬───────┘ └──────┬───────┘ └──────┬───────┘                            │
│          │                │                │                                     │
│          │  S3 VPC Endpoint (Gateway, us-east-1)                                │
│          ▼                ▼                ▼                                     │
│       Producer S3 Buckets (us-east-1)                                           │
│                                                                                  │
│  ┌────────────────────────────────────────────┐                                 │
│  │    Immuta Governance (us-east-1)           │                                 │
│  │    Starburst: Runtime API calls (hard dep) │                                 │
│  │    Snowflake: Native policy sync           │                                 │
│  │    Databricks: Native policy sync          │                                 │
│  └────────────────────────────────────────────┘                                 │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐          │
│  │    Consumer Notification Layer (us-east-1)                        │          │
│  │                                                                   │          │
│  │  Channel 1: SNS (us-east-1) ──► Consumer SQS (consumer accounts) │          │
│  │  Channel 2: Lambda/ECS ──► On-Prem Kafka (via DX/VPN)            │          │
│  │  Channel 3: Notification Polling API (ECS + DynamoDB)             │          │
│  └────────────────────────────────────────────────────────────────────┘          │
│                                                                                  │
│  ┌────────────────────────────────────────────┐                                 │
│  │           Consumers (us-east-1)             │                                 │
│  │  Query via Snowflake / Starburst / DBX     │                                 │
│  │  Notified via SNS-SQS / Kafka / Poll API   │                                 │
│  │  SQS queues in us-east-1                   │                                 │
│  └────────────────────────────────────────────┘                                 │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Key Architectural Facts

| Component | Account | Region | Technology |
|-----------|---------|--------|------------|
| Producer Data | Each producer's own AWS account | us-east-1 | S3 (Parquet or Iceberg) |
| Registration APIs | Platform team's AWS account | us-east-1 | ECS behind ALB, Route53 |
| Producer Notification API | Platform team's AWS account | us-east-1 | Lambda — receives data availability events |
| Consumer Notification API | Platform team's AWS account | us-east-1 | Lambda — publishes to SNS, Kafka, DynamoDB |
| Metadata Store | Platform team's AWS account | us-east-1 | DynamoDB |
| SNS Topic | Platform team's AWS account | us-east-1 | SNS → Consumer SQS (cross-account) |
| Notification Polling API | Platform team's AWS account | us-east-1 | ECS/Lambda + DynamoDB — consumers poll for notifications |
| Snowflake | Snowflake managed service | us-east-1 | Storage Integration, External Tables, Unmanaged Iceberg |
| Starburst | Platform/dedicated account | us-east-1 | Glue Catalog, S3 Access Point Aliases |
| Databricks | Databricks managed service | us-east-1 | External Tables, Unity Catalog |
| Immuta | Governance team's account | us-east-1 | Runtime API (Starburst), Native sync (SF/DBX) |
| Consumer Notifications | Platform + Consumer accounts | us-east-1 | SNS→SQS, Kafka push, Polling API |

---

## 3. Ideal State Architecture

### 3.1 Overview

In the Ideal State, **every participant flips to us-west-2**. us-west-2 becomes the new primary region for all operations.

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                     IDEAL STATE (us-west-2 as Primary)                           │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                           │
│  │ Producer A   │  │ Producer B   │  │ Producer C   │  ... (ALL producers)      │
│  │ (Own Account)│  │ (Own Account)│  │ (Own Account)│                           │
│  │              │  │              │  │              │                           │
│  │  S3 Bucket   │  │  S3 Bucket   │  │  S3 Bucket   │  ← PRIMARY WRITE         │
│  │  us-west-2   │  │  us-west-2   │  │  us-west-2   │    LOCATION              │
│  │              │  │              │  │              │                           │
│  │ S3 CRR:     │  │ S3 CRR:     │  │ S3 CRR:     │  ← REVERSED               │
│  │ west2→east1 │  │ west2→east1 │  │ west2→east1 │    REPLICATION             │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                           │
│         │                 │                 │                                    │
│         │  Register Dataset / Notify New Data                                   │
│         ▼                 ▼                 ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐            │
│  │        Data Mesh Platform (Platform Account, us-west-2)         │            │
│  │                                                                  │            │
│  │  Route53 ──► ALB ──► ECS (Registration APIs) [us-west-2]        │            │
│  │                                                                  │            │
│  │  Lambda: Producer Notification API [us-west-2]                   │            │
│  │    ├── Receives "new data available" from producers              │            │
│  │    ├── Writes metadata to DynamoDB (us-west-2 primary)           │            │
│  │    └── Updates us-west-2 Glue Catalog                            │            │
│  │                                                                  │            │
│  │  Lambda: Consumer Notification API [us-west-2]                   │            │
│  │    ├── Publishes to SNS (us-west-2) → Consumer SQS (us-west-2)  │            │
│  │    ├── Pushes to On-Prem Kafka (via DX/VPN from us-west-2)      │            │
│  │    └── Stores notification in DynamoDB for polling               │            │
│  │                                                                  │            │
│  │  DynamoDB Global Table (us-west-2 primary writes)                │            │
│  │  SNS Topic (us-west-2)                                           │            │
│  └──────────┬──────────────┬───────────────┬────────────────────────┘            │
│             │              │               │                                     │
│             ▼              ▼               ▼                                     │
│  ┌───────────────┐ ┌──────────────┐ ┌──────────────┐                            │
│  │  Snowflake    │ │  Starburst   │ │  Databricks  │                            │
│  │  (us-west-2)  │ │  (us-west-2) │ │  (us-west-2) │                            │
│  │               │ │              │ │              │                            │
│  │ External Tbls │ │ Glue Catalog │ │ External Tbls│                            │
│  │ Iceberg Tbls  │ │ (us-west-2)  │ │ Unity Catalog│                            │
│  │ Storage Integ.│ │ S3 Access Pt │ │ (us-west-2)  │                            │
│  │ (west2 S3)   │ │ (west2 S3)   │ │              │                            │
│  └───────┬───────┘ └──────┬───────┘ └──────┬───────┘                            │
│          │                │                │                                     │
│          │  S3 VPC Endpoint (Gateway, us-west-2)                                │
│          ▼                ▼                ▼                                     │
│       Producer S3 Buckets (us-west-2) — PRIMARY WRITE LOCATION                  │
│                                                                                  │
│  ┌────────────────────────────────────────────┐                                 │
│  │    Immuta Governance (us-west-2)           │                                 │
│  │    Starburst: Runtime API calls            │                                 │
│  │    Snowflake: Native policy sync           │                                 │
│  │    Databricks: Native policy sync          │                                 │
│  └────────────────────────────────────────────┘                                 │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐          │
│  │    Consumer Notification Layer (us-west-2)                        │          │
│  │                                                                   │          │
│  │  Channel 1: SNS (us-west-2) ──► Consumer SQS (us-west-2)         │          │
│  │  Channel 2: Lambda/ECS (us-west-2) ──► On-Prem Kafka (via DX)    │          │
│  │  Channel 3: Notification Polling API (us-west-2 ECS + DynamoDB)   │          │
│  └────────────────────────────────────────────────────────────────────┘          │
│                                                                                  │
│  ┌────────────────────────────────────────────┐                                 │
│  │           Consumers (us-west-2)             │                                 │
│  │  SQS queues in us-west-2                   │                                 │
│  │  Query via Snowflake / Starburst / DBX     │                                 │
│  │  (all in us-west-2)                        │                                 │
│  │  Notified via SNS-SQS / Kafka / Poll API   │                                 │
│  └────────────────────────────────────────────┘                                 │
└──────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│          us-east-1 (Secondary / Backup)              │
│                                                      │
│  Producer S3 Buckets (replicated from us-west-2)     │
│  ← Receives reverse CRR replication from west2      │
│  Purpose: Backup, failback, compliance               │
└──────────────────────────────────────────────────────┘
```

### 3.2 Key Advantage — Iceberg Path Problem Eliminated

In the Ideal State, the **Iceberg metadata absolute path problem disappears entirely** because:

1. Producers write directly to us-west-2 S3 buckets
2. Iceberg metadata files contain **us-west-2 paths natively** (e.g., `s3://producer-bucket-us-west-2/warehouse/table/data/file.parquet`)
3. Compute platforms in us-west-2 read from us-west-2 S3 — **paths match perfectly**
4. No need for path substitution, MRAP, `register_table`, or failover-time re-registration

### 3.3 Replication Direction

| Before Flip | After Flip |
|-------------|------------|
| Primary write: us-east-1 | Primary write: **us-west-2** |
| CRR: east1 → west2 | CRR: **west2 → east1** (reversed) |
| west2 is DR replica | east1 is backup replica |

### 3.4 What Changes for Each Component

| Component | Current (east1) | Ideal State (west2) |
|-----------|-----------------|---------------------|
| Producer S3 writes | Write to east1 bucket | Write to **west2** bucket |
| S3 CRR direction | east1 → west2 | **west2 → east1** |
| Registration API | ECS in east1 | ECS in **west2** |
| Producer Notification API | Lambda in east1 | Lambda in **west2** |
| Consumer Notification API | Lambda in east1 | Lambda in **west2** |
| SNS Topic | east1 | **west2** |
| DynamoDB | east1 primary | **west2** primary (Global Table) |
| Glue Catalog | east1 | **west2** |
| Snowflake | east1 account | **west2** account (internal objects via replication groups; external/Iceberg tables recreated via automation — not natively replicated) |
| Starburst | east1 cluster | **west2** cluster |
| Databricks | east1 workspace | **west2** workspace |
| Immuta | east1 instance | **west2** instance |
| Consumer SQS | east1 | **west2** |
| Kafka push | Lambda in east1 → DX → Kafka | Lambda in **west2** → DX → Kafka |
| VPC Endpoint | east1 gateway | **west2** gateway |

---

## 4. Hybrid State Architecture

### 4.1 Overview

In the Hybrid State, **most producers flip to us-west-2** but **a few producers cannot flip** and remain in us-east-1. For non-flipping producers, **Snowflake Private Listing (or SnowGrid)** bridges the gap by replicating their unmanaged Iceberg tables from the us-east-1 Snowflake account to the us-west-2 Snowflake account.

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                            HYBRID STATE                                          │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─── us-west-2 (PRIMARY REGION) ────────────────────────────────────────────┐  │
│  │                                                                            │  │
│  │  ┌──────────────┐  ┌──────────────┐                                       │  │
│  │  │ Producer A   │  │ Producer B   │  ... (FLIPPED producers)              │  │
│  │  │ ✅ FLIPPED   │  │ ✅ FLIPPED   │                                       │  │
│  │  │ S3 (west2)   │  │ S3 (west2)   │  ← Writing to us-west-2             │  │
│  │  │ CRR→east1    │  │ CRR→east1    │  ← Reverse replication              │  │
│  │  └──────┬───────┘  └──────┬───────┘                                       │  │
│  │         │                 │                                                │  │
│  │         ▼                 ▼                                                │  │
│  │  ┌──────────────────────────────────────────────────────────────────┐      │  │
│  │  │        Data Mesh Platform (us-west-2)                           │      │  │
│  │  │  Registration API | Producer Notification API                   │      │  │
│  │  │  Consumer Notification API | SNS | DynamoDB | Lambda            │      │  │
│  │  └──────────┬──────────────┬───────────────┬───────────────────────┘      │  │
│  │             │              │               │                               │  │
│  │             ▼              ▼               ▼                               │  │
│  │  ┌────────────────────┐ ┌──────────────┐ ┌──────────────┐                 │  │
│  │  │ Snowflake (west2)  │ │ Starburst    │ │ Databricks   │                 │  │
│  │  │                    │ │ (us-west-2)  │ │ (us-west-2)  │                 │  │
│  │  │ ┌────────────────┐ │ │              │ │              │                 │  │
│  │  │ │ Flipped Prod.  │ │ │ Flipped      │ │ Flipped      │                 │  │
│  │  │ │ Tables (native │ │ │ Producer     │ │ Producer     │                 │  │
│  │  │ │ west2 S3)      │ │ │ data via     │ │ data via     │                 │  │
│  │  │ │                │ │ │ west2 Glue   │ │ west2 Unity  │                 │  │
│  │  │ ├────────────────┤ │ │ & S3 Access  │ │ Catalog      │                 │  │
│  │  │ │ Non-flipped    │ │ │ Points       │ │              │                 │  │
│  │  │ │ Prod. Tables   │ │ │              │ │              │                 │  │
│  │  │ │ (via Private   │ │ │ ⚠️ Non-flip  │ │ ⚠️ Non-flip  │                 │  │
│  │  │ │ Listing from   │ │ │ data NOT     │ │ data NOT     │                 │  │
│  │  │ │ east1 SF acct) │ │ │ available    │ │ available    │                 │  │
│  │  │ └────────────────┘ │ │              │ │              │                 │  │
│  │  └────────────────────┘ └──────────────┘ └──────────────┘                 │  │
│  │                                                                            │  │
│  │  Immuta Governance (us-west-2)                                            │  │
│  │                                                                            │  │
│  │  Consumer Notification Layer (us-west-2)                                  │  │
│  │    SNS → Consumer SQS | Kafka | Notification API                         │  │
│  │                                                                            │  │
│  │  Consumers (us-west-2)                                                    │  │
│  │    Query all data via west2 compute platforms                             │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│         ▲                                                                        │
│         │  Snowflake Private Listing / SnowGrid                                 │
│         │  (Cross-region replication of unmanaged Iceberg tables)                │
│         │  east1 SF account ──► west2 SF account                                │
│         │                                                                        │
│  ┌─── us-east-1 (NON-FLIPPED PRODUCERS ONLY) ────────────────────────────────┐  │
│  │                                                                            │  │
│  │  ┌──────────────┐  ┌──────────────┐                                       │  │
│  │  │ Producer X   │  │ Producer Y   │  ... (CANNOT flip)                    │  │
│  │  │ ❌ NOT FLIPPED│  │ ❌ NOT FLIPPED│                                       │  │
│  │  │ S3 (east1)   │  │ S3 (east1)   │  ← Still writing to us-east-1       │  │
│  │  └──────┬───────┘  └──────┬───────┘                                       │  │
│  │         │                 │                                                │  │
│  │         ▼                 ▼                                                │  │
│  │  ┌────────────────────────────────────┐                                   │  │
│  │  │ Snowflake (us-east-1 account)      │                                   │  │
│  │  │ Unmanaged Iceberg Tables           │                                   │  │
│  │  │ pointing to east1 S3 buckets       │                                   │  │
│  │  │                                    │                                   │  │
│  │  │ Acts as PROVIDER for               │                                   │  │
│  │  │ Snowflake Private Listing ─────────┼──► Replicated to west2 SF acct   │  │
│  │  └────────────────────────────────────┘                                   │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Snowflake Private Listing / SnowGrid for Non-Flipped Producers

For each producer that **cannot flip** to us-west-2:

| Step | What Happens |
|------|-------------|
| 1 | Producer continues writing to **us-east-1 S3 bucket** (no change for them) |
| 2 | Snowflake **us-east-1 account** has unmanaged Iceberg tables pointing to east1 S3 |
| 3 | Platform team creates a **Snowflake Private Listing** on the east1 account, sharing the non-flipped producer's datasets |
| 4 | Snowflake **us-west-2 account** subscribes to the Private Listing |
| 5 | Snowflake handles **cross-region data replication** transparently |
| 6 | Data becomes available in the **us-west-2 Snowflake account** as a shared database |
| 7 | Consumers in us-west-2 query through the west2 Snowflake account — **no change for consumers** |
| 8 | **Non-flipped producers call the west2 Producer Notification API** (cross-region API call) to notify of new data |

> **IMPORTANT — Technical Validation Required for Private Listing with Unmanaged Iceberg Tables**
> Snowflake's standard replication (BCR-1528) silently skips external tables and unmanaged Iceberg tables. Private Listing / SnowGrid may handle this differently, but **this must be validated with Snowflake support** before the Hybrid design is finalized. If Private Listing cannot directly share unmanaged Iceberg tables, the Platform Team must first convert or wrap those tables in the east1 SF account into a shareable form (e.g., materialized views or managed Iceberg copies from the unmanaged source). This is a **potential blocker** for the Hybrid State.

### 4.3 Platform Availability by Compute Platform (Hybrid)

| Compute Platform | Flipped Producer Data | Non-Flipped Producer Data |
|-----------------|----------------------|---------------------------|
| **Snowflake** (us-west-2) | ✅ Native tables (west2 S3) | ✅ Via Private Listing (east1 → west2 SF) |
| **Starburst** (us-west-2) | ✅ Via west2 Glue + S3 Access Points | ❌ **Not available** — Private Listing is Snowflake-only |
| **Databricks** (us-west-2) | ✅ Via west2 Unity Catalog + S3 | ❌ **Not available** — Private Listing is Snowflake-only |

> **Important:** Non-flipped producer data is **only available through Snowflake** in the Hybrid State. Starburst and Databricks consumers who need non-flipped producer data must query through Snowflake until the producer completes their flip.

### 4.4 Hybrid State Limitations

| Limitation | Impact | Mitigation |
|-----------|--------|------------|
| Non-flipped data only in Snowflake | Starburst/Databricks consumers lose access to non-flipped datasets | Encourage producers to flip; provide Snowflake access as interim |
| Private Listing replication lag | Non-flipped data may be minutes behind real-time | Monitor replication lag; communicate SLA to consumers |
| east1 dependency for non-flipped | If east1 has outage, non-flipped data becomes unavailable | Accept risk for non-critical producers; push Tier 1 to flip |
| Additional Snowflake cost | Cross-region replication charges | Cost justified for transition period |
| Two data access patterns | Increases operational complexity | Automate; converge to Ideal State over time |

---

## 5. Tasks for Ideal State

### 5.1 Data Mesh Platform Team

| # | Task | Priority | Details |
|---|------|----------|---------|
| 1 | **Create VPC, subnets, S3 Gateway VPC Endpoint in us-west-2** | Critical | Network foundation for all west2 services |
| 2 | **Deploy ECS Registration APIs in us-west-2** | Critical | Mirror east1: ALB + ECS + Route53 |
| 3 | **Deploy Producer Notification API (Lambda) in us-west-2** | Critical | Configure for west2 Glue Catalog, DynamoDB |
| 4 | **Deploy Consumer Notification API (Lambda) in us-west-2** | Critical | Publish to west2 SNS, push to Kafka via west2 DX |
| 5 | **Create SNS topic in us-west-2** | Critical | Mirror east1 topic; set up cross-account subscriptions |
| 6 | **Configure DynamoDB Global Tables** | Critical | Add west2 replica; switch primary writes to west2 |
| 7 | **Deploy all Lambda functions in us-west-2** | Critical | Notification, processing, automation functions |
| 8 | **Set up Glue Catalog in us-west-2** | Critical | Register all datasets with west2 S3 paths. Recreate Lake Formation permissions separately (they do not replicate from east1) |
| 8a | **Modify Producer Notification Lambda to write to west2 Glue Catalog** | Critical | Code change: replace east1 Glue write path with west2. During transition, dual-write to both catalogs |
| 9 | **Update Route53 to point to us-west-2** | Critical | Failover routing or direct CNAME switch |
| 10 | **Provision Snowflake account in us-west-2** | Critical | Storage Integrations, External Volumes for west2 S3 |
| 11 | **Build external table / Iceberg table creation automation for west2 Snowflake** | Critical | DDL generation, validation |
| 12 | **Deploy Starburst cluster in us-west-2** | Critical | Configure: west2 Glue, S3 Access Points, VPC endpoint |
| 13 | **Deploy Databricks workspace in us-west-2** | Critical | Unity Catalog metastore, external locations for west2 S3 |
| 14 | **Deploy Immuta instance in us-west-2** | Critical | Policy-as-Code sync, integrate with west2 compute platforms |
| 15 | **Provision Direct Connect / VPN from us-west-2 to on-prem** | High | Required for Kafka notification channel |
| 16 | **Build self-service producer onboarding toolkit for west2** | High | IAM policies, bucket policies, validation scripts |
| 17 | **ECR cross-region replication** | High | Keep container images in sync — stale images would break ECS deployments in west2. All west2 infra should be deployed via the same IaC as east1, parameterized by region |
| 18 | **Build monitoring & alerting for west2 infrastructure** | High | CloudWatch dashboards, alarms, runbooks |
| 19 | **Decommission east1 platform services after full cutover** | Medium | After traffic fully shifts: decommission Registration APIs, SNS, Lambda, Glue Catalog, east1 DynamoDB primary role. Keep S3 replicas |

### 5.2 Producer Teams

| # | Task | Priority | Details |
|---|------|----------|---------|
| 1 | **Switch primary writes to us-west-2 S3 bucket** | Critical | Update ETL/pipeline configurations |
| 2 | **Reverse S3 CRR direction: west2 → east1** | Critical | Maintain east1 as backup replica |
| 3 | **Grant compute platform IAM roles access to west2 bucket** | Critical | Snowflake, Starburst, Databricks west2 roles |
| 4 | **Create S3 Access Points on west2 bucket** | Critical | Required for Starburst |
| 5 | **Update application/pipeline configurations** | Critical | All write paths target west2 |
| 6 | **Enable S3 versioning on both west2 bucket (CRR source) and east1 bucket (CRR destination)** | Critical | Required for CRR in both directions |
| 7 | **Enable S3 Replication Metrics and Monitoring** | High | CloudWatch alarms for reverse replication health |
| 8 | **Validate end-to-end data flow in west2** | High | Write → Replicate → Query verification |

### 5.3 Consumer Teams

| # | Task | Priority | Details |
|---|------|----------|---------|
| 1 | **Create SQS queue in us-west-2** | Critical | In consumer's own AWS account |
| 2 | **Subscribe west2 SQS to west2 SNS topic** | Critical | Cross-account subscription |
| 3 | **Update application to poll west2 SQS** | Critical | Or poll both during transition |
| 4 | **Update compute platform connection strings** | Critical | Point to west2 Snowflake/Starburst/DBX (or use CNAME) |
| 5 | **Implement idempotent message processing** | High | Required during transition window when both east1 and west2 notification paths may be briefly active simultaneously. Can be relaxed once east1 services are fully decommissioned |
| 5a | **On-prem Kafka consumers: validate message delivery via new west2 DX path** | High | Test throughput and latency through the west2 Direct Connect/VPN path |
| 6 | **Test end-to-end: notification → query → data** | High | Full pipeline validation in west2 |
| 7 | **Decommission east1 SQS after transition complete** | Low | Clean up old resources |

---

## 6. Tasks for Hybrid State

### 6.1 Data Mesh Platform Team (Additional to Ideal State Tasks)

All Ideal State tasks (Section 5.1, #1–18) apply. The following are **additional** tasks specific to Hybrid:

| # | Task | Priority | Details |
|---|------|----------|---------|
| H1 | **Maintain Snowflake us-east-1 account** | Critical | Required as Private Listing provider for non-flipped producers |
| H2 | **Set up Snowflake Private Listing / SnowGrid** | Critical | Create cross-region listing for each non-flipped producer's datasets |
| H3 | **Configure east1 SF account as Private Listing provider** | Critical | Share non-flipped producer tables via listings |
| H4 | **Configure west2 SF account as Private Listing consumer** | Critical | Subscribe to listings; make data available in west2 |
| H5 | **Build monitoring for Private Listing replication lag** | High | Alert on replication staleness thresholds |
| H6 | **Maintain producer registry (flipped vs non-flipped)** | High | Track which producers have flipped and which haven't |
| H7 | **Handle Starburst/Databricks access for non-flipped data** | High | Document limitation; provide Snowflake as alternative |
| H8 | **Validate Immuta governance on Private Listing shared tables in west2 SF** | High | Confirm Immuta can enforce row-level and column-level policies on cross-account shared Snowflake objects |
| H9 | **Ensure non-flipped producers can call west2 Producer Notification API** | High | Configure cross-region API call path; may need east1 endpoint proxy or direct cross-region call |
| H10 | **Build automation for producer transition (hybrid → ideal)** | Medium | When a producer flips: remove Private Listing, switch to native tables |
| H11 | **Maintain east1 Snowflake Storage Integrations** | Medium | East1 SF account needs continued access to east1 producer S3 |
| H12 | **Cost tracking for Private Listing replication** | Medium | Monitor and report cross-region replication costs |

### 6.2 Producer Teams (Non-Flipping)

| # | Task | Priority | Details |
|---|------|----------|---------|
| 1 | **Continue writing to us-east-1 S3 bucket** | Critical | No change to current write path |
| 2 | **Ensure Snowflake east1 IAM role has continued S3 access** | Critical | Specifically, the east1 Snowflake Storage Integration IAM role ARN (provided by Platform Team) must remain in the producer's S3 bucket policy. Changes to this policy must be coordinated with the Platform Team — removing this role breaks Private Listing silently |
| 3 | **Coordinate with Platform Team on Private Listing setup** | High | Validate data availability in west2 SF |
| 4 | **Create and commit to a timeline for eventual flip to west2** | Medium | Plan the migration even if delayed |

### 6.3 Consumer Teams (Same as Ideal State)

Consumer tasks are **identical to Ideal State** (Section 5.3). The consumer experience is the same — they operate entirely in us-west-2. The only difference:

- Non-flipped producer data accessed via Snowflake may have **slightly higher latency** due to Private Listing replication
- Non-flipped producer data is **only available in Snowflake** (not Starburst or Databricks)

---

## 7. Comparison: Ideal vs Hybrid

| Aspect | Ideal State | Hybrid State |
|--------|-------------|--------------|
| **Producer Adoption** | All producers flip to west2 | Most flip; some remain in east1 |
| **Iceberg Path Problem** | ✅ Eliminated entirely (native west2 paths) | ✅ Eliminated for flipped; handled by Private Listing for non-flipped |
| **Data Freshness** | Real-time for all producers | Real-time for flipped; Private Listing lag for non-flipped |
| **Snowflake Access** | All tables native in west2 | Flipped = native; non-flipped = via Private Listing |
| **Starburst Access** | ✅ Full access to all data | ⚠️ No access to non-flipped producer data |
| **Databricks Access** | ✅ Full access to all data | ⚠️ No access to non-flipped producer data |
| **Architecture Complexity** | Lower — single data path | Higher — two data paths (native + Private Listing) |
| **Operational Cost** | Lower — no Private Listing fees | Higher — Private Listing cross-region charges |
| **us-east-1 Dependency** | None (only backup replication) | Non-flipped producers depend on east1 S3 AND the east1 SF account as Private Listing provider — an east1 outage makes all non-flipped data unavailable in west2 SF as well |
| **Consumer Experience** | Uniform — all data via west2 | Mostly uniform; some Snowflake-only limitations |
| **Risk Profile** | Requires all producers to coordinate flip | Accommodates producers who can't flip on schedule |
| **Path to Steady State** | This IS the steady state | Transitional — converges to Ideal over time |

---

## 8. Snowflake Private Listing / SnowGrid Deep Dive

### 8.1 What is Snowflake Private Listing?

Snowflake Private Listing enables **cross-region, cross-account data sharing** within Snowflake. A provider account shares datasets via a private listing, and consumer accounts in **any Snowflake region** can subscribe to receive replicated data.

### 8.2 How It Works for Non-Flipped Producers

```
┌─── us-east-1 ───────────────────────────┐     ┌─── us-west-2 ───────────────────┐
│                                          │     │                                  │
│  Producer X S3 Bucket (east1)            │     │                                  │
│       │                                  │     │                                  │
│       ▼                                  │     │                                  │
│  Snowflake Account (east1)               │     │  Snowflake Account (west2)       │
│  ┌──────────────────────┐                │     │  ┌──────────────────────┐        │
│  │ Unmanaged Iceberg    │                │     │  │ Shared Database      │        │
│  │ Tables (pointing to  │  Private       │     │  │ (replicated from     │        │
│  │ east1 S3)            │  Listing ──────┼────►│  │  east1 via listing)  │        │
│  │                      │  / SnowGrid    │     │  │                      │        │
│  │ PROVIDER ACCOUNT     │                │     │  │ CONSUMER ACCOUNT     │        │
│  └──────────────────────┘                │     │  └──────────┬───────────┘        │
│                                          │     │             │                    │
│                                          │     │             ▼                    │
│                                          │     │  Consumers query via west2 SF    │
│                                          │     │                                  │
└──────────────────────────────────────────┘     └──────────────────────────────────┘
```

### 8.3 Setup Steps

| # | Step | Owner | Detail |
|---|------|-------|--------|
| 0 | **Pre-condition:** Verify east1 SF account has active Storage Integrations for each non-flipped producer's S3 bucket and that unmanaged Iceberg tables are queryable. Run sample SELECT to confirm before creating the listing | Platform Team | Validation gate |
| 1 | Verify east1 SF account is on **Business Critical edition** — required for cross-region Private Listing. If not, raise a contract upgrade request. **This is a potential blocker** | Platform Team | Required for cross-region replication |
| 2 | Create a Private Listing in east1 SF account | Platform Team | Include non-flipped producer databases/schemas |
| 3 | Set west2 SF account as the listing consumer | Platform Team | Cross-region subscription |
| 4 | Validate data availability in west2 SF account | Platform Team | Query sample tables, check freshness |
| 5 | Grant consumers access to shared database in west2 SF | Platform Team | Roles, grants, policies |
| 6 | Monitor replication lag | Platform Team | Set alerts for staleness > threshold |

### 8.4 Considerations

- **Replication Lag:** Snowflake manages replication internally. Lag is typically minutes but can vary with data volume
- **Cost:** Snowflake charges for cross-region data transfer and storage of replicated data
- **Schema Changes:** If the east1 provider adds/removes columns, the listing updates propagate automatically
- **Access Control:** Immuta policies applied in the west2 SF account govern consumer access to the replicated data
- **Limitation:** Private Listing works within Snowflake only — cannot serve Starburst or Databricks
- **Immuta Governance on Shared Data:** Immuta policy coverage over shared databases from Private Listings must be validated with the Governance Team — Immuta's ability to enforce row-level and column-level policies on cross-account shared Snowflake objects should not be assumed without testing

---

## 9. Migration Sequencing

### Phase 1: Foundation (Weeks 1–4)
- Deploy VPC, networking, S3 VPC Endpoints in us-west-2
- Set up DynamoDB Global Tables
- Provision Snowflake, Starburst, Databricks in us-west-2
- **Initiate Direct Connect / VPN provisioning from us-west-2 to on-prem in Week 1** — lead time is 4–8 weeks and this is on the critical path for the Kafka notification channel
- Deploy Immuta in us-west-2 with Policy-as-Code sync

### Phase 2: Platform Flip (Weeks 5–8)
- Deploy all Data Mesh Platform APIs in us-west-2
- Set up SNS topic in us-west-2
- Configure Glue Catalog in us-west-2
- Build and test external table automation
- Begin dual-region operation (both regions active for testing)

### Phase 3: Producer Migration (Weeks 9–16)
- Tier 1 (critical) producers flip first
- Tier 2 (important) producers flip next
- Tier 3 (standard) producers flip on their schedule
- For producers that cannot flip: set up Snowflake Private Listing

### Phase 4: Consumer Migration (Weeks 13–18)
- Consumers create west2 SQS queues
- Consumers update connection strings
- Validate end-to-end notifications and queries
- **Coordination gate:** Consumers should only cut over to west2 SQS and compute connections after the Platform Team confirms that all Tier 1 and Tier 2 producers they depend on have completed their flip. Verify producer dependency map before scheduling consumer cutover

### Phase 5: Cutover & Cleanup (Weeks 17–20)
- Route53 DNS switch to west2 as primary
- Decommission east1 platform services (keep S3 replicas)
- For Hybrid: maintain east1 SF account and Private Listings
- Monitor and validate steady-state

---

## 10. Risks & Mitigations

| # | Risk | Severity | Mitigation |
|---|------|----------|------------|
| 1 | **Producer coordination delays** — not all producers can flip simultaneously | High | Hybrid State accommodates stragglers; tiered migration schedule |
| 2 | **Direct Connect provisioning lead time** (4–8 weeks) | High | Start DX/VPN provisioning in Phase 1 |
| 3 | **Snowflake Private Listing limitations** — Starburst/Databricks cannot consume | Medium | Communicate limitation; encourage faster producer flip |
| 4 | **Private Listing replication lag** — non-flipped data may be stale | Medium | Monitor lag; communicate SLA; accept for non-critical data |
| 5 | **East1 outage during Hybrid** — non-flipped data becomes unavailable | Medium | Accept risk for producers who chose not to flip |
| 6 | **Cost overrun** — running infra in two regions during migration | Medium | Phase-based migration; decommission east1 services progressively |
| 7 | **Consumer migration coordination** — consumers must update SQS and connections | Medium | Provide CNAME-based routing; self-service tooling |
| 8 | **Immuta cross-region policy sync delay** | Medium | Policy-as-Code CI/CD pipeline; validate before flip |
| 9 | **Kafka notification gap during DX switch** | Medium | Pre-provision DX from west2; test before cutover |
| 10 | **Iceberg table recreation time in Snowflake** | Medium | Pre-build tables in west2; nightly sync automation |

---

*This document was prepared for architecture review. It supersedes the DR-focused approach with a proactive regional flip strategy.*

*Last updated: April 14, 2026*
