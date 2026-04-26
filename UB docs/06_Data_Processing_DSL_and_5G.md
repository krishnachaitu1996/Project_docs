# Usage Broker (UB) Ecosystem — Technical Specification

## Part 6: Data Processing — DSL & 5G Pipeline

---

## Section A: Data Stream Layer — `jitr-data-dsl`

---

### 1. Application Identity

| Attribute           | Value                                          |
|---------------------|-------------------------------------------------|
| **Artifact ID**     | `jitr-data-dsl`                                  |
| **Group ID**        | `com.vzwcorp.jitr`                               |
| **Main Class**      | `DSLApplication`                                 |
| **Packaging**       | JAR (Spring Boot)                                |
| **Service Discovery**| Eureka (`@EnableDiscoveryClient`)               |
| **Component Scan**  | `com.vzw.erg.jitr.data`                          |

---

### 2. Purpose

The **Data Stream Layer (DSL)** processes **data and voice usage records** through a multi-stage pipeline:

- **Ingests** from Kafka RBM topics (3G, 4G, Recycle) or from DM file-system directories.
- **Applies business rules** — EIW (Equipment Identifier Whitelist), RIW (RIW strip), cell-site lookup, 4G/5G rules.
- **Enriches** with reference data from Oracle REED schema (cell-site geolocation, PLMN identifiers, GnodeB/EnodeB).
- **Outputs** RBM CSV files distributed to TDC/ODC/SDC/B2B directories.
- **Audits** at inbound, outbound, and error levels to Oracle MZAUD schema.

---

### 3. Architecture

#### 3.1 Package Organization

| Package        | Responsibility                                                |
|----------------|---------------------------------------------------------------|
| `config`       | DB connections (MZADMIN, REED, MZAUD), Kafka, thread pools   |
| `processor`    | Business rule execution chain                                 |
| `processor.impl`| EIWProcessor, EVFProcessor, RIWProcessor, BusinessRulesProcessorFor4G5G |
| `rbm`          | RBM transformation and output formatting                      |
| `service`      | Kafka consumer starters (RBM, UMM), producer, error handling |
| `db`           | Bulk lookup, repositories, entity classes                     |
| `vo`           | Value objects (DSLAudit, DSLObject, DSLListPerFile, etc.)    |
| `utils`        | Constants, date/format helpers                                |
| `actuator`     | Custom suspend/resume/status endpoints                       |

#### 3.2 Processing Chains

**Kafka Consumer Path:**
```
Kafka RBM Topics (3G, 4G, RECYCL)
    │
    └─ DSLRBMConsumerStarter → DSLRBMConsumerThread
         │
         ├─ Parse pipe-delimited record (129 fields)
         ├─ BaseProcessor chain:
         │    ├─ EIWProcessor (Equipment Identifier Whitelist lookup)
         │    ├─ RIWProcessor (RIW strip logic)
         │    ├─ BusinessRulesProcessorFor4G5G (4G/5G rule application)
         │    └─ EVFProcessor (EVF validation)
         ├─ Reference data enrichment (cell-site, PLMN, GnodeB/EnodeB)
         └─ RBM CSV output → TDC/ODC/SDC/B2B directories
```

**File Consumer Path:**
```
DM Input Directories
    │
    └─ FileProcessorThread → FileProcessorHelper
         │
         ├─ Read pipe-delimited file
         ├─ Same processor chain as above
         └─ Same output paths
```

#### 3.3 `BaseProcessor` Interface

```java
public interface BaseProcessor {
    void process(List<String> inputList, DSLListPerFile dslListPerChunk, DSLAudit dslAudit);
}
```

All processors implement this interface, applying their specific logic to batches of records.

---

### 4. Database Connections

| Database     | Purpose                                        | Connection Pool |
|--------------|------------------------------------------------|-----------------|
| **MZADMIN**  | Configuration, reference tables, bulk MDN lookup | HikariCP       |
| **REED**     | Cell-site data, PLMN identifiers, IP ranges     | HikariCP       |
| **MZAUD**    | Audit tables (inbound, outbound, errors, dropped)| HikariCP       |

#### 4.1 Audit Database Schema (MZAUD)

**Table: `JITR_AUDIT_DATA_INBOUND`** (52+ columns)

| Key Columns                        | Description                         |
|------------------------------------|-------------------------------------|
| `INBOUND_ID`                       | Primary key (sequence-generated)    |
| `INPUT_FILE_NAME`                  | Source file name                    |
| `USAGE_TYPE_CODE`                  | Data/Voice type code                |
| `INPUT_RECORD_COUNT`               | Records received                    |
| `DECODED_RECORD_COUNT`             | Records successfully decoded        |
| `ERROR_RECORD_COUNT`               | Records with errors                 |
| `DGID1` through `DGID24`          | Distribution Group ID counters      |
| `START_DTTM`, `END_DTTM`          | Processing timestamps               |
| `PROCESS_STATUS`                   | Current processing status           |

**Table: `JITR_AUDIT_DATA_OUTBOUND`** — Tracks output file-level metrics.

**Table: `JITR_AUDIT_DATA_ERRORS`** — Stores individual error records with codes.

**Oracle Object Type: `EIW_OBJ_TYPE`** — Used for bulk EIW operations with input/output fields.

#### 4.2 Reference Data Objects

| Value Object            | Data Source | Description                                   |
|-------------------------|-------------|-----------------------------------------------|
| `BSID`                  | REED        | Base Station ID mapping                       |
| `CellDefltHmActive`     | REED        | Cell default home/active status               |
| `ENodeB`                | REED        | 4G eNodeB cell-site data                      |
| `GNodeB`                | REED        | 5G gNodeB cell-site data                      |
| `PLMNIdentifier`        | REED        | Public Land Mobile Network identifiers        |
| `IPRangeObject`         | REED        | IP range to location mapping                  |
| `NetworkDetailsFromDB`  | MZADMIN     | Network configuration reference               |

---

### 5. Kafka Configuration

| Property                                             | Description                      |
|------------------------------------------------------|----------------------------------|
| `jitr.dsl.kafka.consumer.bootstrap.url`              | Kafka cluster address            |
| `jitr.dsl.rbm.topic.3g`                              | 3G RBM input topic               |
| `jitr.dsl.rbm.topic.4g`                              | 4G RBM input topic               |
| `jitr.dsl.rbm.topic.recycl`                          | Recycle RBM input topic          |
| `jitr.dsl.kafka.consumer.poll.records`               | Records per poll                  |
| `jitr.dsl.kafka.consumer.session.timeout`            | Consumer session timeout          |
| `jitr.dsl.kafka.consumer.heartbeat.interval`         | Heartbeat interval                |
| Topic-to-UsageType mapping                           | Maps Kafka topics to type codes   |

Consumer threads: Configurable (perf environment: 60 threads).

### 6. Output Distribution

Records are written as RBM CSV files and distributed to the following targets:

| Output Dir  | Description                                |
|-------------|--------------------------------------------|
| **TDC**     | Tampa Data Center                          |
| **ODC**     | Other Data Center                          |
| **SDC**     | Secondary Data Center                      |
| **B2B**     | Business-to-Business output                |

### 7. Additional Integrations

- **IBM MQ**: Integrated via `mq-jms-spring-boot-starter 3.1.0` for JMS message queue communication.
- **LMAX Disruptor**: Used for high-performance async logging.
- **Jasypt**: Password encryption for all DB credentials.

---

## Section B: 5G Multi-Module Pipeline — `jitr-ub-data`

---

### 1. Project Identity

| Attribute           | Value                                          |
|---------------------|-------------------------------------------------|
| **Artifact ID**     | `jitr-ub-data`                                   |
| **Type**            | POM (Multi-module parent)                        |
| **Version**         | `1.0`                                            |
| **Description**     | Application modules to process data usage in streaming fashion |
| **Parent POM**      | `ub-jitr-domain-parent:23.06.100-SNAPSHOT`       |

### 2. Purpose

The 5G pipeline processes **5G data usage records** (D5G) through a distributed microservice architecture:

1. **Ingest** — XML files or Apache Pulsar streams
2. **Process** — Decode, validate, enrich with billing data, apply business rules
3. **Route** — Dynamic Kafka topic routing based on biller ID
4. **Output** — Write rated records to files/streams
5. **Audit** — Aggregate and persist audit data to Oracle AUD_5G schema

---

### 3. Module Overview

```
jitr-ub-data/
  ├─ jitr-ub-data-d5g-record-def         (Shared Record Definitions)
  ├─ jitr-ub-data-d5g-filestreamer-app    (XML File Ingestion)
  ├─ jitr-ub-data-d5g-pulsarconsumer-app  (Pulsar Stream Ingestion)
  ├─ jitr-ub-data-d5g-app                 (Core 5G Processor)
  ├─ jitr-ub-data-d5g-output-app          (Output Writer)
  └─ jitr-ub-data-audit-aggregator        (Audit Aggregation)
```

---

### 4. Module Details

#### 4.1 `jitr-ub-data-d5g-record-def` — Shared Record Definitions

| Attribute          | Value                     |
|--------------------|---------------------------|
| **Version**        | 2.4                       |
| **Type**           | JAR Library               |
| **Purpose**        | Shared DTOs, constants, record structures used across all D5G modules |

#### 4.2 `jitr-ub-data-d5g-filestreamer-app` — XML File Consumer

| Attribute          | Value                                    |
|--------------------|------------------------------------------|
| **Main Class**     | `ApplicationMain`                        |
| **Port**           | 14250                                    |
| **App Name**       | `jitr-ub-data-filestreamer`              |
| **Exclusions**     | JndiConnectionFactory, DataSource, HibernateJPA, KafkaAutoConfiguration |

**Responsibilities:**
- Polls configured input folder(s) for XML files on a cron schedule.
- Parses XML using **XStream** library (v1.4.x).
- Compresses payloads using **Zstandard** (`zstd-jni 1.5.4-1`).
- Publishes parsed records to Kafka topic for downstream processing by `d5g-app`.
- Supports file locking (prevents double-processing across instances).
- Configurable input delay (minutes) and files-per-execution limits.

**Key Classes:**
| Class                   | Role                                      |
|-------------------------|-------------------------------------------|
| `XMLFileHandler5GMain`  | Main file processing orchestrator         |
| `XMLFileProducer5G`     | Kafka producer for parsed XML records     |

**File Lifecycle:**
```
Input Folder → Staging (locked) → Process → Kafka Publish
    ├─ Success → Complete Folder
    ├─ Decode Error → Undecodable Error Folder
    └─ Kafka Error → Kafka Reprocess Folder
```

**Database Connections:**
- MZAUD (audit inbound/outbound)
- AUD_5G (5G-specific audit)

#### 4.3 `jitr-ub-data-d5g-pulsarconsumer-app` — Pulsar Stream Consumer

| Attribute          | Value                     |
|--------------------|---------------------------|
| **Purpose**        | Alternative ingestion from Apache Pulsar |
| **Type**           | Spring Boot Application   |

Consumes 5G usage records from Apache Pulsar topics (as an alternative to file-based ingestion), transforms them, and publishes to the same Kafka topics consumed by `d5g-app`.

#### 4.4 `jitr-ub-data-d5g-app` — Core 5G Processor

| Attribute          | Value                                    |
|--------------------|------------------------------------------|
| **Main Class**     | `ApplicationMainD5G`                     |
| **Port**           | 14251                                    |
| **App Name**       | `jitr-ub-d5gapp`                         |
| **Component Scan** | `com.vzw.erg.umm.data.d5g`, `com.vzw.billing` |

**Responsibilities:**
- **Consumes** records from FileStreamer/Pulsar Kafka topics.
- **Translates** raw 5G records via `D5GTranslateAndProcessMapper`.
- **Performs bulk MDN lookups** against MZADMIN (`BulkLookup`, `BulkLookupDbOperations`).
- **Caches reference data** via `DataCacheService`.
- **Routes** billable records dynamically via `D5GBillableRecordRouter`.
- **Sends ECS errors** via ECS service integration.

**Dynamic Topic Routing (`D5GBillableRecordRouter`):**

Implements Kafka Streams `TopicNameExtractor` for routing based on billing determination:

| Route Destination | Kafka Topic Pattern                    | Description                 |
|-------------------|----------------------------------------|-----------------------------|
| `JITRRT`          | `@JITR.UB.D5G.MAIN.DSL.TOPIC.JITRRT@`| JITR Real-Time              |
| `JITRRO`          | `@JITR.UB.D5G.MAIN.DSL.TOPIC.JITRRO@`| JITR Rated-Only             |
| `JITRRS`          | `@JITR.UB.D5G.MAIN.DSL.TOPIC.JITRRS@`| JITR RS instance            |
| `JITRBS`          | `@JITR.UB.D5G.MAIN.DSL.TOPIC.JITRBS@`| JITR BS instance            |
| `DROPPED`         | Topic for dropped records              | Non-billable / invalid      |
| `ECS`             | ECS error topic                        | Error correction            |

**Key Format:** `sliceid~destination~...`

**Consumer Configuration:**
- Multi-threaded consumers (`AppConsumerThread`)
- Rebalance handling (`AppConsumerRebalanceHandler`)
- Configurable poll records, session timeout, heartbeat interval

#### 4.5 `jitr-ub-data-d5g-output-app` — Output Writer

| Attribute          | Value                                    |
|--------------------|------------------------------------------|
| **Main Class**     | `ApplicationMainOutput`                  |
| **Port**           | 14252                                    |
| **App Name**       | `jitr-ub-data-d5g-output-app`            |

**Responsibilities:**
- Consumes rated/routed records from downstream Kafka topics.
- Writes output files for multiple sinks:

| Output Type      | Package   | Description                              |
|------------------|-----------|------------------------------------------|
| **DROPPED**      | `dropped` | Non-billable records file output          |
| **NONBILLING**   | `dropped` | Non-billing / fraud copy records          |
| **LRA**          | `lra`     | Late Rated Adjustment records             |
| **BCE**          | `bce`     | Billing Collection Engine records         |
| **RSS**          | (config)  | Revenue Share Settlement                  |
| **AFB**          | `afb`     | Anti-Fraud & Billing records              |

**File Writer Configuration:**

| Property                           | Description                     |
|------------------------------------|---------------------------------|
| `max.file.records`                 | Max records per output file     |
| `max.non.billing.records`          | Max non-billing records per file|
| `max.time.reached.minutes`         | Time-based file rotation        |
| `max.dropped.records`              | Max dropped records per file    |
| `lra.output.max.line.count`        | LRA-specific max line count     |
| `bce.output.max.line.count`        | BCE-specific max line count     |

**BCE REVO Integration:**
- Dedicated topic for BCE Revenue Operations
- Separate consumer group and output directory

**Copy Configuration:**
- `ROBOCALL` copy — duplicate records flagged as robocall
- `COPYNONBILLINGFRAUD` — copy of non-billing fraud records

#### 4.6 `jitr-ub-data-audit-aggregator` — Audit Aggregation

| Attribute          | Value                                    |
|--------------------|------------------------------------------|
| **Main Class**     | `AuditServiceMain`                       |
| **Port**           | 14256                                    |
| **Scheduler**      | Quartz (clustered mode)                  |

**Responsibilities:**
- Consumes audit events from Kafka audit topic.
- Aggregates audit data by time slices.
- Persists to **AUD_5G Oracle schema** and **PROCCTRL** database.
- Provides centralized audit view across all 5G pipeline stages.

**Audit Entity Model:**

| Entity                 | Table                       | Description                    |
|------------------------|-----------------------------|--------------------------------|
| `AuditDataInbound`     | `UMM_AUDIT_DATA_INBOUND`   | Inbound data record audit      |
| `AuditDataOutbound`    | `UMM_AUDIT_DATA_OUTBOUND`  | Outbound data record audit     |
| `AuditDataError`       | `UMM_AUDIT_DATA_ERROR`     | Data error record audit        |
| `AuditDnupInbound`     | DNUP inbound table          | DNUP inbound audit             |
| `AuditDnupOutbound`    | DNUP outbound table         | DNUP outbound audit            |
| `AuditDnupError`       | DNUP error table            | DNUP error audit               |
| `AuditVoiceInbound`    | Voice inbound table         | Voice inbound audit            |
| `AuditVoiceOutbound`   | Voice outbound table        | Voice outbound audit           |
| `AuditVoiceError`      | Voice error table           | Voice error audit              |

**`AuditDataInbound` Table Structure:**

| Column                  | Type          | Description                    |
|-------------------------|---------------|--------------------------------|
| `data_inbound_row_id`   | NUMBER (PK)   | Sequence-generated primary key |
| `time_slice`            | VARCHAR2      | Time window for aggregation    |
| `file_name`             | VARCHAR2      | Source file identifier         |
| `usage_type`            | VARCHAR2      | Data type code                 |
| `recs_num`              | NUMBER        | Record count in this slice     |
| `status`                | VARCHAR2      | Processing status              |
| `status_reason`         | VARCHAR2      | Reason for status              |
| `hostname`              | VARCHAR2      | Processing host                |
| `process_start_ts`      | TIMESTAMP     | Slice processing start         |
| `process_end_ts`        | TIMESTAMP     | Slice processing end           |
| `created_date`          | DATE          | Audit record creation          |

**Quartz Configuration (Clustered):**

| Setting                            | Value                              |
|------------------------------------|------------------------------------|
| `isClustered`                      | `true`                             |
| Thread count                       | 5                                  |
| Check-in interval                  | 20,000 ms                          |
| Cron expression                    | Configurable per environment       |

**Database Connections:**
- AUD_5G (HikariCP) — Primary audit storage
- PROCCTRL (HikariCP) — Processing control / Quartz tables

---

### 5. 5G Error Code Reference

| Error Code        | Description                         |
|-------------------|-------------------------------------|
| `D5G_D_E3001`     | MDN_NOT_FOUND_ON_ETNI               |
| `D5G_D_E3002`     | UNHANDLED_BILLERID                  |
| `D5G_D_E9897`     | Internal processing error           |
| `D5G_D_E9898`     | Record parsing error                |
| `D5G_D_E9899`     | Validation error                    |
| `D5G_D_E9900-E9999` | Various internal/system errors    |

---

### 6. End-to-End 5G Data Flow

```
   XML Files (DM)                    Apache Pulsar
        │                                │
        ▼                                ▼
  d5g-filestreamer-app          d5g-pulsarconsumer-app
  (Port 14250)                   │
        │                        │
        └──── Kafka Topic ───────┘
                    │
                    ▼
           d5g-app (Port 14251)
           ├─ Translate & Process
           ├─ Bulk MDN Lookup (MZADMIN)
           ├─ Reference Data Cache
           └─ D5GBillableRecordRouter
                    │
        ┌───────────┼───────────────────┐
        ▼           ▼                   ▼
  JITR Topics    Auxiliary Topics    ECS Topic
  (RT/RO/RS/BS)  (DROPPED/LRA/BCE/  (errors)
        │         RSS/AFB/NONBILLING)
        │               │
        ▼               ▼
  (JITR Rating)   d5g-output-app (Port 14252)
                  ├─ Dropped file writer
                  ├─ LRA file writer
                  ├─ BCE file writer
                  ├─ AFB file writer
                  └─ RSS/NONBILLING writer
                         │
                         ▼
                  audit-aggregator (Port 14256)
                  ├─ Kafka audit topic consumer
                  ├─ Quartz scheduled aggregation
                  └─ Oracle AUD_5G persistence
```

---

*Continue to [Part 7: Gap Analysis & Appendices](07_Gap_Analysis_and_Appendices.md)*
