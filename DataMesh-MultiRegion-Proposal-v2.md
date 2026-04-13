# Data Mesh Platform — Multi-Region Resiliency: Ideal & Hybrid Architecture (v2)

**Version:** 2.0  
**Date:** April 14, 2026  
**Status:** Draft — For Architecture Review  
**Primary Region (Production):** us-east-1 (N. Virginia)  
**Secondary Region (DR):** us-west-2 (Oregon)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current Architecture](#2-current-architecture)
3. [Ideal State — Proposed Architecture](#3-ideal-state--proposed-architecture)
4. [Tasks for Ideal State](#4-tasks-for-ideal-state)
5. [Hybrid State — Proposed Architecture](#5-hybrid-state--proposed-architecture)
6. [Tasks for Hybrid State](#6-tasks-for-hybrid-state)
7. [DR Flip Sequence](#7-dr-flip-sequence)
8. [Comparison & Key Decisions](#8-comparison--key-decisions)

---

## 1. Executive Summary

This proposal defines the target architecture for making the Data Mesh platform resilient to a complete failure of the primary production region (**us-east-1**), with **us-west-2** as the disaster recovery (DR) secondary region.

Two target states are defined:

### Ideal State

All producers, the Data Mesh platform, and all AWS consumers are **fully prepared to flip to us-west-2** during a DR event.

- **us-east-1 (PRIMARY):** Active production — all components running, all producers writing here
- **us-west-2 (DR):** Fully pre-provisioned warm standby — all platform infra deployed, all compute platforms provisioned, consumer notification channels ready
- **Replication:** Continuous S3 CRR (east1→west2), DynamoDB Global Tables, Glue Catalog dual-write, SNS dual-publish, Immuta Policy-as-Code sync
- **During DR flip:** Producers reverse replication (west2→east1) and start writing to west2; platform activates west2 infra; consumers switch to west2 endpoints

### Hybrid State

Most producers are ready to flip, but **a few producers cannot flip**. For non-flipping producers:

- Their data remains in us-east-1 S3 buckets even during a flip
- **Snowflake Private Listing (SnowGrid)** replicates their unmanaged Iceberg table data from the us-east-1 Snowflake account to the us-west-2 Snowflake account
- Consumers in us-west-2 access non-flipped producer data via the west2 Snowflake account (Private Listing)
- Non-flipped producer data is **only available through Snowflake** — not through Starburst or Databricks

> **Critical Validation Required:** Snowflake standard replication (BCR-1528) silently skips unmanaged Iceberg tables. The ability of Private Listing / SnowGrid to handle unmanaged Iceberg tables **must be validated with Snowflake support** before the Hybrid design is finalized. If not supported directly, tables must be converted to a shareable form (e.g., managed Iceberg copies).

---

## 2. Current Architecture

All components operate exclusively in **us-east-1**. There is **no DR capability**.

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                       CURRENT STATE — us-east-1 ONLY                             │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  PRODUCERS                                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                           │
│  │ Producer A   │  │ Producer B   │  │ Producer C   │  ... (N producers)        │
│  │ (Own Account)│  │ (Own Account)│  │ (Own Account)│                           │
│  │ S3 (Parquet) │  │ S3 (Iceberg) │  │ S3 (Parquet) │                           │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                           │
│         │  Register / Notify New Data       │                                    │
│         ▼                 ▼                 ▼                                    │
│  DATA MESH PLATFORM (Platform Account)                                           │
│  ┌──────────────────────────────────────────────────────────────────┐            │
│  │  Route53 → ALB → ECS (Registration APIs)                        │            │
│  │  Lambda: Producer Notification API (receives from producers)     │            │
│  │  Lambda: Consumer Notification API (publishes to consumers)      │            │
│  │  DynamoDB (Metadata) · SNS Topic · Glue Catalog                  │            │
│  │  Notification Polling API (ECS/Lambda + DynamoDB)                │            │
│  └──────────┬──────────────┬───────────────┬────────────────────────┘            │
│             ▼              ▼               ▼                                     │
│  COMPUTE PLATFORMS                                                               │
│  ┌───────────────┐ ┌──────────────┐ ┌──────────────┐                            │
│  │  Snowflake    │ │  Starburst   │ │  Databricks  │                            │
│  │  External Tbl │ │  Glue Catalog│ │  External Tbl│                            │
│  │  Iceberg Tbl  │ │  S3 AccPt    │ │  Unity Cat.  │                            │
│  │  Storage Int. │ │              │ │              │                            │
│  └───────┬───────┘ └──────┬───────┘ └──────┬───────┘                            │
│          └── S3 VPC Gateway Endpoint ──────┘                                    │
│                    → Producer S3 Buckets (us-east-1)                             │
│                                                                                  │
│  GOVERNANCE                                                                      │
│  ┌────────────────────────────────────────────┐                                 │
│  │  Immuta: Starburst (runtime API, hard dep) │                                 │
│  │          Snowflake (native sync)           │                                 │
│  │          Databricks (native sync)          │                                 │
│  └────────────────────────────────────────────┘                                 │
│                                                                                  │
│  CONSUMER NOTIFICATIONS                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐          │
│  │  Ch1: SNS (east1) → Consumer SQS (consumer accounts, east1)       │          │
│  │  Ch2: Lambda/ECS → On-Prem Kafka (via Direct Connect / VPN)       │          │
│  │  Ch3: Notification Polling API (ECS + DynamoDB)                    │          │
│  └────────────────────────────────────────────────────────────────────┘          │
│                                                                                  │
│  CONSUMERS — Query via Snowflake/Starburst/Databricks                           │
│              Notified via SNS-SQS / Kafka / Polling API                         │
└──────────────────────────────────────────────────────────────────────────────────┘
```

**Risk:** A full us-east-1 outage renders the **entire platform completely unavailable** — APIs, compute, governance, notifications, and all consumer queries. Historical precedent: us-east-1 outages in 2017, 2020, 2021, 2023.

---

## 3. Ideal State — Proposed Architecture

In the Ideal State, **both regions are fully provisioned**. us-east-1 runs as active production; us-west-2 is a warm standby ready to activate during a DR flip. All producers, the platform, and consumers have infrastructure in both regions.

```
┌─────────────────────────────────────────┐           ┌─────────────────────────────────────────┐
│     us-east-1 (PRIMARY / PRODUCTION)    │           │     us-west-2 (SECONDARY / DR)          │
├─────────────────────────────────────────┤           ├─────────────────────────────────────────┤
│                                         │           │                                         │
│  PRODUCERS (ACTIVE WRITE)               │  S3 CRR   │  PRODUCER S3 REPLICAS                   │
│  ┌────────┐ ┌────────┐ ┌────────┐      │  east1 →  │  ┌────────┐ ┌────────┐ ┌────────┐      │
│  │Prod A  │ │Prod B  │ │Prod C  │      │  west2    │  │Prod A  │ │Prod B  │ │Prod C  │      │
│  │S3 east1│ │S3 east1│ │S3 east1│  ════╪══════════►│  │S3 west2│ │S3 west2│ │S3 west2│      │
│  │PRIMARY │ │PRIMARY │ │PRIMARY │      │(continuous)│  │REPLICA │ │REPLICA │ │REPLICA │      │
│  └───┬────┘ └───┬────┘ └───┬────┘      │           │  └────────┘ └────────┘ └────────┘      │
│      │  Notify  │          │            │           │                                         │
│      ▼          ▼          ▼            │           │                                         │
│  PLATFORM (ACTIVE)                      │  Global   │  PLATFORM (WARM STANDBY)                │
│  ┌──────────────────────────────┐       │  Tables   │  ┌──────────────────────────────┐       │
│  │ Registration API    [ACTIVE] │       │  ════════►│  │ Registration API     [STBY]  │       │
│  │ Producer Notif API  [ACTIVE] │       │  Glue     │  │ Producer Notif API   [STBY]  │       │
│  │ Consumer Notif API  [ACTIVE] │       │  Dual-Wrt │  │ Consumer Notif API   [STBY]  │       │
│  │ DynamoDB       [PRIMARY WRT] │  ════╪══════════►│  │ DynamoDB        [REPLICA RO] │       │
│  │ SNS Topic           [east1]  │       │  SNS      │  │ SNS Topic            [west2] │       │
│  │ Glue Catalog        [east1]  │       │  Dual-Pub │  │ Glue Catalog         [west2] │       │
│  │ Polling API         [ACTIVE] │       │  ════════►│  │ Polling API          [STBY]  │       │
│  │ Lambda Functions    [ACTIVE] │       │           │  │ Lambda Functions     [STBY]  │       │
│  └──────────┬───────────────────┘       │           │  └──────────┬───────────────────┘       │
│             ▼                           │           │             ▼                           │
│  COMPUTE PLATFORMS (ACTIVE)             │           │  COMPUTE PLATFORMS (WARM STANDBY)       │
│  ┌──────────┐ ┌────────┐ ┌──────────┐  │           │  ┌──────────┐ ┌────────┐ ┌──────────┐  │
│  │Snowflake │ │Starbrst│ │Databrcks │  │           │  │Snowflake │ │Starbrst│ │Databrcks │  │
│  │ ACTIVE   │ │ ACTIVE │ │ ACTIVE   │  │           │  │ STANDBY  │ │ STBY   │ │ STANDBY  │  │
│  │ExtTbl,Ice│ │Glue,S3 │ │ExtTbl,UC │  │           │  │ExtTbl,Ice│ │Glue,S3 │ │ExtTbl,UC │  │
│  │ AccPt    │ │AccPt   │ │          │  │           │  │ AccPt    │ │AccPt   │ │          │  │
│  └────┬─────┘ └───┬────┘ └────┬─────┘  │           │  └────┬─────┘ └───┬────┘ └────┬─────┘  │
│       └── VPC Endpoint ───────┘         │           │       └── VPC Endpoint ───────┘         │
│            → east1 S3                   │           │            → west2 S3                   │
│                                         │           │                                         │
│  GOVERNANCE                             │  Policy   │  GOVERNANCE                             │
│  ┌────────────────────┐                 │  as-Code  │  ┌────────────────────┐                 │
│  │ Immuta [ACTIVE]    │  ══════════════╪══════════►│  │ Immuta [STANDBY]   │                 │
│  │ SF sync, SB runtime│                 │  CI/CD    │  │ SF sync, SB runtime│                 │
│  │ DBX sync           │                 │           │  │ DBX sync           │                 │
│  └────────────────────┘                 │           │  └────────────────────┘                 │
│                                         │           │                                         │
│  CONSUMER NOTIFICATIONS (ACTIVE)        │  SNS      │  CONSUMER NOTIFICATIONS (STANDBY)       │
│  ┌────────────────────────────┐         │  Dual-Pub │  ┌─────────────────────────────┐        │
│  │ SNS→SQS (east1)  [ACTIVE] │  ══════╪══════════►│  │ SNS→SQS (west2)  [STANDBY]  │        │
│  │ Kafka via DX east1[ACTIVE] │         │           │  │ Kafka via DX west2 [STBY]   │        │
│  │ Polling API       [ACTIVE] │         │           │  │ Polling API        [STBY]   │        │
│  └────────────────────────────┘         │           │  └─────────────────────────────┘        │
│                                         │  Route53  │                                         │
│  CONSUMERS (ACTIVE)                     │  Failover │  CONSUMERS (READY)                      │
│  ┌────────────────────────────┐         │  ════════►│  ┌─────────────────────────────┐        │
│  │ SQS (east1) — polling      │         │           │  │ SQS (west2) — pre-created   │        │
│  │ Query SF/SB/DBX (east1)    │         │           │  │ Query SF/SB/DBX (west2) rdy │        │
│  │ Use CNAME endpoints        │         │           │  │ Use same CNAME endpoints    │        │
│  └────────────────────────────┘         │           │  └─────────────────────────────┘        │
│                                         │           │                                         │
└─────────────────────────────────────────┘           └─────────────────────────────────────────┘

CROSS-REGION SYNCHRONIZATION (Always Active):
══════════════════════════════════════════════════════════════════════════════════
 S3 CRR (east1→west2, 15-min SLA via RTC)  │  DynamoDB Global Tables (~2s lag)
 Glue Catalog Dual-Write (from Notif API)   │  SNS Dual-Publish (both topics)
 Route53 Failover Routing (health checks)   │  ECR Cross-Region Replication
 Immuta Policy-as-Code (CI/CD to both)      │  Snowflake Account Replication
══════════════════════════════════════════════════════════════════════════════════
```

### Key Design Points

| Aspect | us-east-1 (PRIMARY) | us-west-2 (DR) |
|--------|---------------------|-----------------|
| **Producer S3** | Active write location | Replica via S3 CRR (15-min SLA) |
| **Registration API** | Active (ECS behind ALB) | Warm standby (min task count) |
| **Producer Notification API** | Active (Lambda) | Deployed, standby |
| **Consumer Notification API** | Active (Lambda) | Deployed, standby |
| **DynamoDB** | Primary writes | Read replica (Global Tables, ~2s lag) |
| **SNS Topic** | Active — consumers poll east1 SQS | Receives dual-publish; consumer SQS standby |
| **Glue Catalog** | Active — updated by Notification API | Synced via dual-write from Notification API |
| **Snowflake** | Active account — external/Iceberg tables | Warm standby — tables recreated via automation |
| **Starburst** | Active cluster | Warm standby (min worker nodes) |
| **Databricks** | Active workspace | Warm standby workspace |
| **Immuta** | Active instance | Standby — policies synced via Policy-as-Code CI/CD |
| **Consumer SNS→SQS** | Active — consumers poll east1 SQS | Standby — consumers have pre-created west2 SQS |
| **On-Prem Kafka** | Lambda pushes via east1 DX/VPN | Lambda ready via west2 DX/VPN (pre-provisioned) |
| **Polling API** | Active (ECS + DynamoDB) | Standby (covered by platform failover) |
| **Consumers** | Active — query east1 compute platforms | Ready — switch to west2 compute on failover |

### During DR Flip (us-east-1 outage)

When a DR flip is triggered:
1. **Producers** reverse S3 CRR (now west2→east1) and start writing to us-west-2 S3 buckets
2. **Route53** fails over all API endpoints to us-west-2 ALBs
3. **DynamoDB** us-west-2 becomes the write region
4. **ECS services** scale up in us-west-2
5. **Snowflake** external/Iceberg tables recreated in west2 account via automation
6. **Starburst** west2 cluster scales up, reads from west2 Glue + S3
7. **Databricks** west2 workspace activated
8. **Immuta** west2 instance activated
9. **Consumer SNS→SQS** west2 SNS already receiving dual-publish; consumers switch to west2 SQS
10. **On-Prem Kafka** west2 Lambda pushes via west2 DX to on-prem Kafka
11. **Consumers** switch to west2 compute platforms (transparent via CNAME)

### Iceberg Path Problem — Solved for Flipped Producers

After the flip, producers write directly to us-west-2 S3 buckets. Iceberg metadata files contain **native west2 paths**. No MRAP, no path substitution, no failover-time re-registration needed. The Iceberg absolute path problem is eliminated for all producers that flip.

---

## 4. Tasks for Ideal State

### 4.1 Data Mesh Platform Team

| # | Task | Priority | Details |
|---|------|----------|---------|
| 1 | Create VPC, subnets, S3 Gateway VPC Endpoint in us-west-2 | Critical | Network foundation for all west2 services |
| 2 | Deploy ECS Registration APIs in us-west-2 (ALB + ECS + Route53) | Critical | Warm standby — min task count |
| 3 | Deploy Producer Notification API (Lambda) in us-west-2 | Critical | Configure for west2 Glue, DynamoDB |
| 4 | Deploy Consumer Notification API (Lambda) in us-west-2 | Critical | Publish to west2 SNS, push to Kafka via west2 DX |
| 5 | Create SNS topic in us-west-2 | Critical | Mirror east1 topic config; set up cross-account subs |
| 6 | Configure DynamoDB Global Tables (add west2 replica) | Critical | east1 = primary writes; west2 = read replica until flip |
| 7 | Deploy all Lambda functions in us-west-2 | Critical | All notification, processing, automation functions |
| 8 | Set up Glue Catalog in us-west-2 | Critical | Register all datasets with west2 S3 paths; recreate Lake Formation permissions |
| 8a | Modify Notification Lambda for Glue dual-write | Critical | Code change: write partitions to BOTH east1 and west2 Glue Catalogs |
| 9 | Implement SNS dual-publish | Critical | Consumer Notification API publishes to BOTH east1 and west2 SNS topics |
| 10 | Configure Route53 failover routing | Critical | Health checks on east1; failover to west2 ALB |
| 11 | Provision Snowflake account in us-west-2 | Critical | Storage Integrations, External Volumes for west2 S3 |
| 12 | Build external table / Iceberg table recreation automation for west2 Snowflake | Critical | DDL from DynamoDB metadata; run nightly + on-demand during flip |
| 13 | Deploy Starburst cluster in us-west-2 | Critical | west2 Glue, S3 Access Points, VPC endpoint; warm standby |
| 14 | Deploy Databricks workspace in us-west-2 | Critical | Unity Catalog, external locations for west2 S3 |
| 15 | Deploy Immuta instance in us-west-2 | Critical | Policy-as-Code CI/CD sync to both instances |
| 16 | Provision Direct Connect / VPN from us-west-2 to on-prem | High | 4-8 week lead time — start immediately; required for Kafka |
| 17 | Build self-service producer onboarding toolkit for west2 | High | IAM policies, bucket policies, validation scripts |
| 18 | Build monitoring & alerting for west2 infrastructure | High | CloudWatch dashboards, replication lag alarms |
| 19 | Deploy Notification Polling API in us-west-2 | High | ECS/Lambda + DynamoDB (Global Tables) |
| 20 | ECR cross-region replication for container images | High | Stale images would break ECS deployments |
| 21 | Build and test DR flip runbook | High | End-to-end failover procedure with validation gates |
| 22 | Decommission plan for east1 standby after failback | Medium | Clean return to primary after DR event |

### 4.2 Producer Teams

| # | Task | Priority | Details |
|---|------|----------|---------|
| 1 | Create mirror S3 bucket in us-west-2 | Critical | Same prefix structure as east1 bucket |
| 2 | Enable S3 versioning on BOTH buckets (east1 + west2) | Critical | Required for CRR in both directions |
| 3 | Enable S3 CRR: east1 → west2 with RTC (15-min SLA) | Critical | Continuous replication during normal ops |
| 4 | Grant compute platform IAM roles access to west2 bucket | Critical | Snowflake, Starburst, Databricks west2 roles |
| 5 | Create S3 Access Points on west2 bucket (for Starburst) | Critical | Share alias with Platform Team |
| 6 | Enable S3 Replication Metrics & Monitoring | High | CloudWatch alarms: ReplicationLatency > 10 min |
| 7 | Prepare to reverse CRR (west2→east1) during DR flip | High | Document procedure; test in DR drill |
| 8 | Prepare to switch writes to west2 during DR flip | High | Pipeline/ETL config changes ready to execute |
| 9 | Validate end-to-end: east1 write → west2 replica accessible | High | Quarterly DR drill participation |

### 4.3 Consumer Teams

| # | Task | Priority | Details |
|---|------|----------|---------|
| 1 | Create SQS queue in us-west-2 (own AWS account) | Critical | Pre-created for DR readiness |
| 2 | Subscribe west2 SQS to west2 SNS topic | Critical | Cross-account subscription |
| 3 | Use platform CNAME endpoints (not direct URLs) | Critical | Enables transparent Route53 failover |
| 4 | Prepare to switch polling to west2 SQS during DR | High | Or poll BOTH queues if using dual-publish |
| 5 | Implement idempotent message processing | High | Handle duplicates during dual-publish / transition |
| 6 | On-prem Kafka consumers: validate west2 DX path | High | Test message delivery via west2 Direct Connect |
| 7 | Test end-to-end in west2 during DR drill | High | notification → query → data validation |
| 8 | Document connection failover procedure | Medium | Runbook for switching compute platform connections |

---

## 5. Hybrid State — Proposed Architecture

In the Hybrid State, **most producers can flip** but **a few cannot**. The architecture is identical to the Ideal State, with one addition: **Snowflake Private Listing** bridges non-flipped producer data from us-east-1 to us-west-2.

```
┌─────────────────────────────────────────┐           ┌─────────────────────────────────────────┐
│     us-east-1 (PRIMARY / PRODUCTION)    │           │     us-west-2 (SECONDARY / DR)          │
├─────────────────────────────────────────┤           ├─────────────────────────────────────────┤
│                                         │           │                                         │
│  PRODUCERS                              │  S3 CRR   │  PRODUCER S3 REPLICAS                   │
│  ┌────────┐ ┌────────┐                  │  east1 →  │  ┌────────┐ ┌────────┐                  │
│  │Prod A  │ │Prod B  │  [CAN FLIP]     │  west2    │  │Prod A  │ │Prod B  │                  │
│  │S3 east1│ │S3 east1│  ═══════════════╪══════════►│  │S3 west2│ │S3 west2│  [REPLICAS]     │
│  │PRIMARY │ │PRIMARY │                  │           │  │REPLICA │ │REPLICA │                  │
│  └────────┘ └────────┘                  │           │  └────────┘ └────────┘                  │
│                                         │           │                                         │
│  ┌────────┐ ┌────────┐                  │  ╳ NO    │  (NO S3 REPLICA for non-flipped)        │
│  │Prod X  │ │Prod Y  │  [CANNOT FLIP]  │  CRR     │                                         │
│  │S3 east1│ │S3 east1│                  │           │                                         │
│  │PRIMARY │ │PRIMARY │                  │           │                                         │
│  └───┬────┘ └───┬────┘                  │           │                                         │
│      │          │                        │           │                                         │
│      ▼          ▼                        │           │                                         │
│  Snowflake (east1) reads                │           │  Snowflake (west2) receives via         │
│  unmanaged Iceberg from                 │  Private  │  Private Listing subscription            │
│  east1 S3 buckets                       │  Listing  │  ┌─────────────────────────────┐        │
│  ┌──────────────────────┐  ════════════╪══════════►│  │ Non-flipped producer data   │        │
│  │ SF east1: PROVIDER   │              │  SnowGrid │  │ available as shared DB      │        │
│  │ Unmanaged Iceberg Tbl│              │           │  │ in west2 SF account         │        │
│  └──────────────────────┘              │           │  └─────────────────────────────┘        │
│                                         │           │                                         │
│  PLATFORM (ACTIVE)                      │  Global   │  PLATFORM (WARM STANDBY)                │
│  ┌──────────────────────────────┐       │  Tables   │  ┌──────────────────────────────┐       │
│  │ Registration API    [ACTIVE] │       │  Glue     │  │ Registration API     [STBY]  │       │
│  │ Producer Notif API  [ACTIVE] │       │  Dual-Wrt │  │ Producer Notif API   [STBY]  │       │
│  │ Consumer Notif API  [ACTIVE] │  ════╪══════════►│  │ Consumer Notif API   [STBY]  │       │
│  │ DynamoDB       [PRIMARY WRT] │       │  SNS      │  │ DynamoDB        [REPLICA RO] │       │
│  │ SNS · Glue · Polling API     │       │  Dual-Pub │  │ SNS · Glue · Polling API     │       │
│  │ Lambda Functions    [ACTIVE] │       │           │  │ Lambda Functions     [STBY]  │       │
│  └──────────────────────────────┘       │           │  └──────────────────────────────┘       │
│                                         │           │                                         │
│  COMPUTE PLATFORMS (ACTIVE)             │           │  COMPUTE PLATFORMS (WARM STANDBY)       │
│  ┌──────────┐ ┌────────┐ ┌──────────┐  │           │  ┌──────────┐ ┌────────┐ ┌──────────┐  │
│  │Snowflake │ │Starbrst│ │Databrcks │  │           │  │Snowflake │ │Starbrst│ │Databrcks │  │
│  │ ACTIVE   │ │ ACTIVE │ │ ACTIVE   │  │           │  │ STANDBY  │ │ STBY   │ │ STANDBY  │  │
│  │ all data │ │all data│ │ all data │  │           │  │flip+priv │ │flip    │ │ flip     │  │
│  │          │ │        │ │          │  │           │  │listing   │ │only    │ │ only     │  │
│  └──────────┘ └────────┘ └──────────┘  │           │  └──────────┘ └────────┘ └──────────┘  │
│                                         │           │  ⚠ SB/DBX: NO non-flipped data         │
│  GOVERNANCE · NOTIFICATIONS · CONSUMERS │  ════════►│  GOVERNANCE · NOTIFICATIONS · CONSUMERS │
│  (Same as Ideal State)                  │           │  (Same as Ideal State)                  │
│                                         │           │                                         │
└─────────────────────────────────────────┘           └─────────────────────────────────────────┘

HYBRID-SPECIFIC: Snowflake Private Listing / SnowGrid
═══════════════════════════════════════════════════════
 east1 SF account (PROVIDER) → west2 SF account (CONSUMER)
 Replicates non-flipped producer unmanaged Iceberg tables cross-region
 ⚠ Only available in Snowflake — Starburst/Databricks cannot consume
 ⚠ If east1 has full outage, Private Listing replication stops
 ⚠ Must validate with Snowflake: unmanaged Iceberg + Private Listing (BCR-1528)
```

### Hybrid — Data Availability by Compute Platform (During DR Flip)

| Compute Platform | Flipped Producer Data | Non-Flipped Producer Data |
|-----------------|----------------------|---------------------------|
| **Snowflake** (west2) | ✅ Native tables (west2 S3) | ✅ Via Private Listing (east1 → west2 SF) |
| **Starburst** (west2) | ✅ Via west2 Glue + S3 Access Points | ❌ Not available (Private Listing is SF-only) |
| **Databricks** (west2) | ✅ Via west2 Unity Catalog + S3 | ❌ Not available (Private Listing is SF-only) |

### Hybrid — Non-Flipped Producer Notification Path

Non-flipped producers still call the **Producer Notification API**. During normal operations, this is the east1 API (active). During a DR flip, they call the west2 API via the Route53 CNAME (cross-region API call). The Platform must ensure:
- Non-flipped producers use the CNAME endpoint (not a direct east1 URL)
- The west2 Producer Notification API can process notifications for east1 S3 paths

---

## 6. Tasks for Hybrid State

All Ideal State tasks (Section 4) are prerequisites. The following are **additional** tasks for the Hybrid State.

### 6.1 Data Mesh Platform Team (Additional)

| # | Task | Priority | Details |
|---|------|----------|---------|
| H1 | Maintain Snowflake us-east-1 account as Private Listing provider | Critical | Required for non-flipped producer data |
| H2 | Set up Snowflake Private Listing / SnowGrid for each non-flipped producer | Critical | Create cross-region listing per non-flipped dataset |
| H3 | Configure east1 SF as listing provider (share non-flipped tables) | Critical | Validate unmanaged Iceberg support with Snowflake |
| H4 | Configure west2 SF as listing consumer (subscribe to listings) | Critical | Make non-flipped data available in west2 |
| H5 | Validate Immuta governance on Private Listing shared tables | High | Confirm row/column-level policies work on cross-account shares |
| H6 | Ensure non-flipped producers can call west2 Producer Notification API | High | CNAME must route correctly; test cross-region path |
| H7 | Build monitoring for Private Listing replication lag | High | Alert on staleness thresholds |
| H8 | Maintain producer registry (flipped vs non-flipped status) | High | Track migration progress |
| H9 | Document Starburst/Databricks limitation for non-flipped data | High | Communicate to consumers |
| H10 | Build automation for producer transition: hybrid → ideal | Medium | When producer flips: remove listing, switch to native |
| H11 | Maintain east1 SF Storage Integrations | Medium | Continued access to east1 producer S3 |
| H12 | Cost tracking for Private Listing cross-region replication | Medium | Monitor and report |

### 6.2 Non-Flipping Producer Teams

| # | Task | Priority | Details |
|---|------|----------|---------|
| 1 | Continue writing to us-east-1 S3 (no change) | Critical | Normal production write path unchanged |
| 2 | Ensure east1 SF Storage Integration IAM role remains in S3 bucket policy | Critical | Removing this role breaks Private Listing silently |
| 3 | Use CNAME endpoint for Producer Notification API (not direct east1 URL) | High | Enables cross-region routing during DR |
| 4 | Coordinate with Platform Team on Private Listing setup | High | Validate data availability in west2 SF |
| 5 | Create and commit to timeline for eventual flip to west2 | Medium | Plan migration path |

### 6.3 Consumer Teams (Additional to Ideal)

| # | Task | Priority | Details |
|---|------|----------|---------|
| +1 | Understand: non-flipped producer data only available via Snowflake in west2 | High | Cannot query via Starburst/Databricks |
| +2 | Accept Private Listing replication lag for non-flipped data | Medium | May be minutes behind real-time |

---

## 7. DR Flip Sequence

### Normal Operations (us-east-1 Active)

```
  Producers ──write──► east1 S3 ══CRR══► west2 S3 (replica)
       │
       └──notify──► Platform APIs (east1 ACTIVE)
                         │
                         ├──► Glue east1 (active) + Glue west2 (dual-write)
                         ├──► DynamoDB east1 (primary) ══Global Tables══► DynamoDB west2 (replica)
                         ├──► SNS east1 (active) → Consumer SQS east1
                         ├──► SNS west2 (dual-pub) → Consumer SQS west2 (dormant)
                         └──► Kafka via east1 DX
                         
  Consumers ──query──► Snowflake/Starburst/Databricks (east1 ACTIVE)
  Consumers ──poll───► SQS east1 (active)
```

### During DR Flip (us-east-1 Down)

```
  Step 1: ASSESS (5 min)
    Confirm sustained us-east-1 outage; get VP approval

  Step 2: ACTIVATE PLATFORM (5-10 min)
    Route53 → failover to west2 ALB
    DynamoDB west2 becomes write region
    Scale up ECS services in west2
    Verify Lambda functions in west2

  Step 3: ACTIVATE ICEBERG (10-15 min)
    Validate latest metadata.json in west2 S3
    Run register_table for Iceberg tables in west2 Glue
    Verify all referenced data files exist

  Step 4: ACTIVATE COMPUTE (10-20 min, parallel)
    Snowflake: Run external table recreation automation
    Starburst: Scale up west2 cluster
    Databricks: Activate west2 workspace
    Immuta: Verify west2 instance connectivity

  Step 5: ACTIVATE NOTIFICATIONS (5 min, parallel)
    Verify SNS west2 receiving messages
    Verify Kafka flowing via west2 DX
    Verify Polling API reachable via CNAME

  Step 6: PRODUCERS FLIP (coordinated)
    Reverse CRR: west2 → east1
    Switch writes to west2 S3 buckets
    (Non-flipped producers: no change, Private Listing continues)

  Step 7: NOTIFY CONSUMERS (5 min)
    Status: Platform operational in us-west-2
    List available vs unavailable datasets

  TOTAL RTO: 45-60 minutes (with parallelism)
  RPO: ≤ 15 minutes (S3 CRR replication lag)
```

### After DR Flip (us-west-2 Active)

```
  Producers ──write──► west2 S3 ══CRR══► east1 S3 (reversed replication)
       │
       └──notify──► Platform APIs (west2 ACTIVE)
                         │
                         ├──► Glue west2 (now active)
                         ├──► DynamoDB west2 (now primary writes)
                         ├──► SNS west2 → Consumer SQS west2 (active)
                         └──► Kafka via west2 DX
                         
  Consumers ──query──► Snowflake/Starburst/Databricks (west2 ACTIVE)
  Consumers ──poll───► SQS west2 (active)
  
  [Hybrid only] Non-flipped producers → east1 S3 → east1 SF → Private Listing → west2 SF
                ⚠ Depends on east1 availability
```

---

## 8. Comparison & Key Decisions

### Ideal vs Hybrid Comparison

| Aspect | Ideal State | Hybrid State |
|--------|-------------|--------------|
| **Producer Adoption** | All producers ready to flip | Most ready; some cannot flip |
| **Iceberg Path Problem** | ✅ Eliminated after flip (native west2 paths) | ✅ Eliminated for flipped; Private Listing for non-flipped |
| **Snowflake Access** | ✅ All data native in west2 after flip | ✅ Flipped = native; non-flipped = Private Listing |
| **Starburst / Databricks** | ✅ Full access to all data | ⚠ No access to non-flipped producer data |
| **Architecture Complexity** | Lower — single data path | Higher — two data paths (native + Private Listing) |
| **Cost** | Lower — no Private Listing fees | Higher — Private Listing cross-region charges |
| **us-east-1 Dependency (during flip)** | None (east1 is down) | Non-flipped data depends on east1 SF availability |
| **Consumer Experience** | Uniform | Mostly uniform; Snowflake-only limitation for some datasets |
| **Steady State** | This IS the target | Transitional — converges to Ideal over time |

### Key Decisions Required

| # | Decision | Options | Recommendation | Status |
|---|----------|---------|----------------|--------|
| D1 | Failover strategy | Active-Active vs. Active-Passive | **Active-Passive (Warm Standby)** | Pending |
| D2 | Producer multi-region mandate | Mandatory for all vs. Tiered | **Tiered: Tier 1 mandatory, Tier 2 recommended** | Pending |
| D3 | Snowflake edition | Current vs. Business Critical | **Upgrade to Business Critical** (failover groups + Client Redirect) | Pending |
| D4 | Glue Catalog sync | Replication utility vs. Dual-write | **Dual-write from Notification API** | Pending |
| D5 | SNS notification DR | Dual-publish vs. Active-only | **Dual-publish** (zero consumer action on flip) | Pending |
| D6 | Kafka connectivity | Provision DX from west2 vs. Accept unavailability | **Provision DX from west2** (4-8 week lead) | Pending |
| D7 | Failover trigger | Automatic vs. Manual | **Manual** with automated alerting | Pending |
| D8 | Consumer connections | CNAME vs. Client Redirect vs. Manual | **CNAME** for SB/DBX; **Client Redirect** for SF | Pending |
| D9 | Snowflake Private Listing with unmanaged Iceberg | Validate with Snowflake support | **Must validate before Hybrid design finalized** | **Blocker** |
| D10 | Immuta on Private Listing shared tables | Validate with Immuta/Governance team | **Must validate governance coverage** | **Blocker** |
| D11 | DR drill cadence | Optional vs. Mandatory | **Mandatory quarterly for Tier 1 + all platform teams** | Pending |

### Key Blockers

| # | Blocker | Severity | Owner | Resolution Path |
|---|---------|----------|-------|-----------------|
| B1 | **Direct Connect from us-west-2 to on-prem** — 4-8 week lead time | Critical | Platform + Network Team | Start provisioning immediately in Phase 1 |
| B2 | **Snowflake Private Listing + Unmanaged Iceberg** — BCR-1528 | Critical | Platform Team + Snowflake | Validate with Snowflake support; fallback: managed Iceberg copies |
| B3 | **Snowflake Business Critical edition** — may require contract upgrade | High | Platform Team + Procurement | Verify current edition; initiate upgrade if needed |
| B4 | **Immuta governance on Private Listing shared tables** — untested | High | Governance Team | Validate in pre-prod; fallback: manual policy application |
| B5 | **Iceberg metadata absolute paths** for existing tables during flip | High | Platform Team | Build register_table + path substitution automation |

---

### RPO / RTO Summary

| Component | RPO | RTO | Notes |
|-----------|-----|-----|-------|
| S3 Data (Parquet) | 15 min (CRR + RTC) | 0 min | Pre-replicated |
| S3 Data (Iceberg) | 15 min (CRR + RTC) | 10-15 min | Validation + register_table |
| DynamoDB Metadata | ~2 sec (Global Tables) | 0 min | Active replica |
| Glue Catalog | ~0 (dual-write) | 0 min | Already populated |
| Platform APIs | ~2 sec | 5-10 min | Route53 + ECS scale-up |
| Snowflake (external/Iceberg) | N/A (recreated) | 10-20 min | **Bottleneck** — table count dependent |
| Starburst | ~0 (Glue dual-write) | 5-10 min | Cluster scale-up |
| Databricks | N/A (recreated) | 10-15 min | Similar to Snowflake |
| Immuta | Last CI/CD run | 5-10 min | Policy-as-Code sync |
| SNS→SQS (dual-publish) | ~0 | 0 min | Both regions active |
| On-Prem Kafka | ~2 sec | 5-10 min | Blocked if no west2 DX |
| Polling API | ~2 sec | 5-10 min | Covered by platform failover |
| **Overall** | **≤ 15 min** | **45-60 min** | Bottleneck: Snowflake table recreation |

---

*Prepared for architecture review. All decisions marked "Pending" require sign-off before implementation.*

*Last updated: April 14, 2026*
