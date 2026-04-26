# Usage Broker (UB) Ecosystem — Technical Specification

## Part 1: Executive Overview & System Architecture

| Attribute              | Value                                      |
|------------------------|--------------------------------------------|
| **Document Version**   | 1.0                                        |
| **Domain**             | Verizon Wireless — Mobile B2C & B2B        |
| **System**             | Usage Broker (UB) / JITR Ecosystem         |
| **Classification**     | Internal — Technical Architecture          |
| **Tech Stack**         | Java 17, Spring Boot 3.1, Kafka, Cassandra, Oracle, Kibana |
| **Parent POM**         | `ub-jitr-domain-parent` (Spring Boot 3.1.1) |

---

## 1. Functional Overview

### 1.1 What is the Usage Broker?

The **Usage Broker (UB)** is a middleware platform that sits between the **Network Event Generation layer** (Data Manager / DM) and the **Netcracker RBM (Rating & Billing Manager) / Vision 1.0** downstream rating engines. It acts as the **"middle-man"** in the Verizon wireless billing lifecycle.

Its core responsibilities are:

1. **Ingestion** — Receive raw usage files (SMS, MMS, RCS, Data) from network switches via the Data Manager.
2. **Decoding** — Convert raw binary (BCD), ASCII, CSV, JSON, and XML records into a normalized JSON representation.
3. **Validation & Filtering** — Apply business rules (duplicate checks, date expiry, MDN validation, record-type filters).
4. **Enrichment** — Augment records with internal fields (GRI, MsgDTM normalization, timezone adjustment, sequence numbers, Biller ID resolution).
5. **Routing** — Distribute enriched Usage Data Records (UDRs) to the correct JITR rating instance (RT, RO, RS, BS, GT) and auxiliary downstream sinks (RSS, OPC, ThinAir, Visible, AFB, REVO).
6. **Error Handling** — Capture errored records into the Error Correction Subsystem (ECS) backed by Kafka + Cassandra for reprocessing.
7. **Auditing** — Maintain comprehensive file-level and record-level audit trails in Oracle (MZAUD).

### 1.2 Position in the Billing Lifecycle

```
Network Towers / Switches
        │
        ▼
  Data Manager (DM) ──── Aggregation & Upstream
        │
        ▼
  ┌─────────────────────────────────────────────────┐
  │           USAGE BROKER (UB) ECOSYSTEM           │
  │                                                 │
  │  Ingestion → Decoding → Enrichment → Routing   │
  │            ↘ Error Correction (ECS) ↙           │
  └─────────────────────────────────────────────────┘
        │
        ▼
  JITR Instances (RT/RO/RS/BS/GT)
        │
        ▼
  Netcracker RBM / Vision 1.0 (Rating Engine)
```

### 1.3 Supported Usage Types

| Usage Type | Switch Type       | File Format   | Decoder         | Source       |
|------------|-------------------|---------------|-----------------|--------------|
| LSM        | LucentSMS         | Binary (BCD)  | `LucentSMSDecoder` | Lucent SMSC |
| MMS        | MotorolaMMS       | ASCII (CSV)   | `MotorolaMMSDecoder`| Motorola MMSC |
| SMS        | MotorolaSMS       | Binary (BCD)  | `MotorolaSMSDecoder`| Motorola SMSC |
| RCS        | RCS               | JSON (stream) | `RCSDecoder`    | RCS Platform |
| SMS (XML)  | SMS               | XML           | `SMSDecoder`    | Various      |
| Data/5G    | Various           | Pipe-delimited| DSL Processor   | 4G/5G Core   |

---

## 2. System Architecture

### 2.1 Repository Map

| # | Repository                    | Type        | Purpose                                          |
|---|-------------------------------|-------------|--------------------------------------------------|
| 1 | `ub-jitr-domain-parent`       | Parent POM  | Centralized dependency & version management       |
| 2 | `jitr-ub-msg-dist`            | Application | File ingestion, batching, Kafka publishing        |
| 3 | `jitr-ub-msg-decoder`         | Library     | Binary/ASCII/JSON/XML → JSON decoding             |
| 4 | `jitr-ub-msg`                 | Application | Core mediation: decode, validate, enrich, route   |
| 5 | `jitr-ub-ecs-distributor`     | Library     | Kafka producer for ECS error records              |
| 6 | `jitr-ub-ecs-cdr-generator`   | Library     | Cassandra reader for error record reprocessing    |
| 7 | `jitr-kafka-library`          | Library     | Shared Kafka producer with file-based retry        |
| 8 | `jitr-data-dsl`               | Application | Data Stream Layer — processes data/voice usage     |
| 9 | `jitr-ub-data`                | Multi-module| 5G data processing pipeline (filestreamer, app, output, audit) |

### 2.2 High-Level Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                          USAGE BROKER (UB) ECOSYSTEM                            │
│                                                                                  │
│   ┌──────────────┐     ┌───────────────┐     ┌──────────────────────────┐       │
│   │ jitr-ub-msg  │     │  jitr-ub-msg  │     │     jitr-ub-msg         │       │
│   │    -dist     │────▶│   -decoder    │◀────│    (Mediation Engine)   │       │
│   │  (Ingestion) │     │   (Library)   │     │  Decode│Validate│Route  │       │
│   └──────┬───────┘     └───────────────┘     └──────────┬───────────────┘       │
│          │                                              │                        │
│          ▼                                              │                        │
│   ┌──────────────┐                                      ▼                        │
│   │    Kafka     │                            ┌──────────────────────┐           │
│   │  Raw Topic   │                            │  Output Routing      │           │
│   │  Audit Topic │                            │  ┌────┬────┬────┐   │           │
│   │  ECS Topic   │                            │  │ RT │ RO │ RS │..│           │
│   └──────────────┘                            │  └────┴────┴────┘   │           │
│          │                                    └──────────┬───────────┘           │
│          ▼                                              │                        │
│   ┌──────────────┐    ┌────────────────┐                ▼                        │
│   │ jitr-data    │    │  ECS Subsystem │     ┌──────────────────────┐           │
│   │   -dsl       │    │ ┌────────────┐ │     │   JITR Instances     │           │
│   │ (Data/5G)    │    │ │ ECS-Dist   │ │     │  RT│RO│RS│BS│GT      │           │
│   └──────┬───────┘    │ │ (Kafka)    │ │     └──────────┬───────────┘           │
│          │            │ └────────────┘ │                │                        │
│          │            │ ┌────────────┐ │                ▼                        │
│          │            │ │ ECS-CDR    │ │     ┌──────────────────────┐           │
│          │            │ │(Cassandra) │ │     │  Netcracker RBM      │           │
│          │            │ └────────────┘ │     │  (Rating Engine)     │           │
│          │            └────────────────┘     └──────────────────────┘           │
│          │                                                                       │
│   ┌──────▼───────┐    ┌────────────────┐    ┌──────────────────────┐           │
│   │  jitr-ub-data│    │  Oracle DBs    │    │  Shared Libraries    │           │
│   │  (5G Pipeline)│   │ MZAUD│MZADMIN │    │ jitr-kafka-library   │           │
│   │  FS→App→Out  │    │ REED           │    │ ub-jitr-domain-parent│           │
│   └──────────────┘    └────────────────┘    └──────────────────────┘           │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Technology Stack Detail

| Layer            | Technology                        | Version     | Purpose                          |
|------------------|-----------------------------------|-------------|----------------------------------|
| Language         | Java                              | 17          | Core application language         |
| Framework        | Spring Boot                       | 3.1.1       | Application framework             |
| Build            | Maven                             | 3.x         | Dependency management & build     |
| Messaging        | Apache Kafka                      | 3.5.1       | Event streaming & async messaging |
| NoSQL            | Apache Cassandra                  | 4.x (OSS Driver) | Error record storage       |
| RDBMS            | Oracle                            | 19c         | Audit, config, reference data     |
| Service Discovery| Spring Cloud Netflix Eureka        | 2022.0.3    | Microservice registration         |
| Scheduling       | Spring TaskScheduler + Cron        | Built-in    | Periodic file polling             |
| Compression      | Snappy (Kafka), GZip (archive)    | —           | Data compression                  |
| Logging          | Log4j2 + LMAX Disruptor           | 2.19.0      | Async high-perf logging           |
| Monitoring       | Spring Actuator + Kibana           | —           | Health, metrics, ELK dashboards   |
| Encryption       | Jasypt + BouncyCastle              | 3.0.5 / 1.76| Password encryption               |

### 2.4 Data Flow Summary

The end-to-end pipeline follows this sequence:

1. **Generation** — Network switches generate CDRs (Call Detail Records) for SMS, MMS, RCS, and Data events.
2. **Aggregation** — Data Manager aggregates and delivers usage files to the UB filesystem.
3. **Ingestion** (jitr-ub-msg-dist) — Files are picked up, read line-by-line, batched into JSON, and published to Kafka.
4. **Decoding** (jitr-ub-msg + jitr-ub-msg-decoder) — Raw records are decoded from BCD/ASCII/JSON/XML into normalized JSON.
5. **Enrichment** (jitr-ub-msg) — Records are enriched with GRI, timestamps, routing info, and internal fields.
6. **Validation** (jitr-ub-msg) — Business rules applied (duplicate checks, MDN validation, date filters).
7. **Routing** (jitr-ub-msg) — Valid records are written to output directories for each JITR instance; errors sent to ECS.
8. **Rating** (Downstream) — JITR instances forward UDRs to Netcracker RBM/Vision for final rating.
9. **Error Correction** (ECS) — Errored records stored in Cassandra, available for reprocessing.

### 2.5 JITR Routing Instances

Records are routed based on the **Biller ID** (first character of the customer's billing identifier) to regional/functional JITR instances:

| Biller ID Prefix | JITR Instance | DSS Number | DSS Key |
|-------------------|---------------|------------|---------|
| V                 | RO (Retail Original) | 61    | JTRRO   |
| A                 | RT (Retail)   | 63         | JTRRT   |
| Q                 | GT            | 73         | JTRGT   |
| N                 | RS            | 64         | JTRRS   |
| B                 | BS (Business) | 71         | JTRBS   |
| R, O, T, 5, P    | Not routed (filtered/special handling) | — | — |

### 2.6 Downstream Output Sinks

Beyond JITR instances, records may also be routed to:

| Sink       | Path Pattern                              | Purpose                    |
|------------|-------------------------------------------|----------------------------|
| RSS        | `dwf_merge/rss/{SMS,MMS,RCS}/`            | RSS merge processing        |
| OPC        | `dwf_merge/opc/{LSM,MMS}/`               | OPC processing              |
| ThinAir    | `dwf_cfi/THINAIR/`                       | ThinAir/CFI integration     |
| ThinAir5   | `dwf_cfi/THINAIR5/`                      | ThinAir 5G                  |
| Visible    | `dwf_cfi/visible/{SMS,MMS,RCS}/`         | Visible brand routing       |
| AFB        | `dwf_merge/afb/{SMS,MMS}/`               | AFB processing              |
| Robocall   | `dwf_original/{LSM,MMS,RCS}/`            | Robocall originals          |
| REVO Drop  | `dwf_revo/report_dropped/`               | REVO dropped record reports |
| REVO Except| `dwf_revo/report_exceptions/`            | REVO exception reports      |
| LL Audit   | `dwf_ll/FS{S,M,R}/`                      | Low-level audit trail       |
| Forward    | `dwf_original/{MMS}/`                    | MMS forward handling        |

---

## 3. Deployment & Configuration

### 3.1 Application Ports

| Application       | Default Port |
|--------------------|-------------|
| jitr-ub-msg-dist   | 14190       |
| jitr-ub-msg        | 14060       |
| jitr-data-dsl      | Configurable|

### 3.2 Service Discovery

All applications register with **Spring Cloud Netflix Eureka** for service discovery:
- Instance ID format: `{hostname}:{appName}:{port}`
- IP preference: `false` (hostname-based)
- Cloud Config: Disabled (`spring.cloud.config.enabled=false`)

### 3.3 Actuator Endpoints

Each application exposes Spring Boot Actuator endpoints for operational management:

| Endpoint    | Purpose                              |
|-------------|--------------------------------------|
| `/health`   | Application health check             |
| `/info`     | Application information              |
| `/suspend`  | Pause processing (custom)            |
| `/resume`   | Resume processing (custom)           |
| `/status`   | Current processing status (custom)   |
| `/version`  | Application version (custom)         |

---

*Continue to [Part 2: jitr-ub-msg-dist (File Distributor)](02_File_Distributor_jitr_ub_msg_dist.md)*
