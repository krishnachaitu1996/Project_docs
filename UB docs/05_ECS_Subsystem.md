# Usage Broker (UB) Ecosystem — Technical Specification

## Part 5: Error Correction Subsystem (ECS)

---

This section covers **two libraries** that together form the ECS pipeline:

- `jitr-ub-ecs-distributor` — Publishes error records to Kafka
- `jitr-ub-ecs-cdr-generator` — Reads error records from Cassandra for reprocessing

---

## 1. ECS Overview

The Error Correction Subsystem provides a **closed-loop error recovery mechanism**:

```
Mediation Engine (jitr-ub-msg)
  │
  ├─ On decode/validation error:
  │   → ECSKafkaSender.sendErrorRecordToKafka()  [jitr-ub-ecs-distributor]
  │   → Error record written to Kafka ECS Topic
  │
  ├─ ECS Handshake: INITIAL (at file start)
  ├─ ECS Handshake: COMMIT  (on success)
  └─ ECS Handshake: ROLLBACK (on failure)
       │
       ▼
  Downstream ECS Consumers
  (outside UB boundary — process errors, store in Cassandra)
       │
       ▼
  ECS Reflow Request → creates UUID list file
       │
       ▼
  EcsErrorCdrCollectionService.fetchErrorCdrsById()  [jitr-ub-ecs-cdr-generator]
  → Read from Cassandra: error_record_by_id
  → Return raw bytes to jitr-ub-msg for reprocessing
```

---

## 2. `jitr-ub-ecs-distributor` — Error Record Publisher

### 2.1 Library Identity

| Attribute        | Value                            |
|------------------|----------------------------------|
| **Artifact ID**  | `jitr-ub-ecs-distributor`        |
| **Type**         | JAR Library                      |
| **Consumer**     | `jitr-ub-msg`                    |
| **Kafka Role**   | Producer (ECS error topic)       |

### 2.2 `ECSKafkaSender` (@Service)

The primary class responsible for sending error records to Kafka.

#### Error Record Publishing

```java
public void sendErrorRecordToKafka(List<ECSError> ecsErrorList)
```

For each error record in the list:
1. Builds a **metadata JSON key** from the `ECSError.metaDataMap` (EnumMap of MetadataKeys).
2. Uses `auditFileId % ecsTopicPartitionCount` as the **Kafka partition** for co-location.
3. Sends a `ProducerRecord<String, byte[]>`:
   - **Key**: JSON metadata string
   - **Value**: `errorRecordAsBytes` (raw binary CDR)
   - **Topic**: configurable ECS error topic
4. Tracks error count; throws `RuntimeException` if count exceeds `maxKafkaErrorThresholdVal` (default: 10).

#### Handshake Protocol

```java
public void sendHandshaketoECS(HandshakeMsg handshakeMsg)
```

Three handshake types maintain file-level transactional semantics:

| Handshake        | When Sent                                   | Payload                          |
|------------------|----------------------------------------------|----------------------------------|
| **INITIAL**      | Before file processing begins               | `{auditFileId, timestamp, type: INITIAL}` |
| **COMMIT**       | After successful file completion             | `{auditFileId, timestamp, type: COMMIT}` |
| **ROLLBACK**     | After file processing failure                | `{auditFileId, timestamp, type: ROLLBACK}`|

**Timestamp Caching**: The INITIAL handshake timestamp is cached in a `ConcurrentHashMap<Long, String>` keyed by `auditFileId`. COMMIT/ROLLBACK messages reuse this cached timestamp for correlation.

### 2.3 Data Transfer Objects

#### `ECSError`
```java
public class ECSError {
    private byte[] errorRecordAsBytes;                    // Raw binary CDR bytes
    private EnumMap<MetadataKeys, String> metaDataMap;    // Error metadata
}
```

#### `HandshakeMsg`
```java
public class HandshakeMsg {
    private HandshakeType handshakeType;  // INITIAL, COMMIT, ROLLBACK
    private long auditFileId;
    private String timestamp;
}
```

#### `UsageFileStatus` (Enum)
```java
public enum UsageFileStatus {
    PROCESSING_IN_PROGRESS,
    PROCESSING_COMPLETED,
    PROCESSING_CANCELLED
}
```

### 2.4 `MetadataKeys` — Error Record Metadata

Each error record carries a rich metadata envelope:

| Key                        | Description                                              |
|----------------------------|----------------------------------------------------------|
| `ERROR_RECORD_ID`          | UUID – unique identifier for this error instance         |
| `CURRENT_ERROR_CODE`       | Current validation/decode error code                     |
| `PREVIOUS_ERROR_CODE`      | Error code from prior processing attempt (reprocess)     |
| `USAGE_TYPE`               | LSM / MMS / RCS / SMS                                    |
| `SWITCH_NAME`              | Source switch identifier                                 |
| `USAGE_AUDIT_FILE_ID`      | Oracle audit sequence ID for the parent file             |
| `REPROCESS_COUNT`          | Number of times this record has been reprocessed         |
| `REVO_ERROR_RECORD`        | REVO-formatted error record (if applicable)              |
| `RAW_RECORD_TXT`           | Human-readable text representation of the raw record     |
| `DMC_INPUT_FILE_NAME`      | Original DM filename                                     |
| `RECORD_SEQUENCE_NUMBER`   | Record's position within the file                        |
| `BILLER_ID`                | Resolved biller ID for the record's MDN                  |
| `ENCODING_TYPE`            | BCD / ASCII / JSON / XML                                 |
| `SOURCE_IDENTIFIER_CODE`   | UB_JAVA / REED_RCL                                      |

### 2.5 Error Threshold Protection

To prevent runaway error scenarios from overwhelming the ECS Kafka topic:

```java
if (errorCount >= maxKafkaErrorThresholdVal) {
    throw new RuntimeException("MaxKafkaErrorThreshold reached: " + errorCount);
}
```

- Default threshold: **10 errors per file**.
- When exceeded: file processing **aborted** (cancel batch), file moved to error directory.
- Configurable via `maxKafkaErrorThresholdVal` property.

---

## 3. `jitr-ub-ecs-cdr-generator` — Error Record Reader

### 3.1 Library Identity

| Attribute        | Value                                |
|------------------|--------------------------------------|
| **Artifact ID**  | `jitr-ub-ecs-cdr-generator`         |
| **Type**         | JAR Library                          |
| **Consumer**     | `jitr-ub-msg` (ECS Reflow workflow)  |
| **Data Source**   | Apache Cassandra                    |

### 3.2 Cassandra Data Model

#### Table: `error_record_by_id`

| Column                      | Type        | Description                       |
|-----------------------------|-------------|-----------------------------------|
| `error_record_id`           | `UUID` (PK) | Unique error record identifier   |
| `raw_error_record`          | `blob`       | Original binary CDR bytes        |
| `raw_error_record_txt`      | `text`       | Human-readable representation    |
| `revo_error_record`         | `text`       | REVO-formatted error             |
| `record_expiry_timestamp`   | `timestamp`  | TTL-based expiry marker          |

### 3.3 `EcsErrorCdrCollectionService`

```java
public Map<UUID, ErrorRecordEntity> fetchErrorCdrsById(Set<UUID> errorRecordIds)
```

**Flow:**

1. Accepts a batch of error record UUIDs.
2. Delegates to `ECSCassandraRepository.fetchByIds()`.
3. Repository executes async queries (`executeAsync()`) using prepared statements.
4. Collects results into a `Map<UUID, ErrorRecordEntity>`.
5. On failure during batch: **retries once** (for transient Cassandra issues).
6. Returns the map to the calling mediation processor.

### 3.4 `ECSCassandraRepository`

```java
PreparedStatement:
  SELECT error_record_id, raw_error_record, raw_error_record_txt,
         revo_error_record, record_expiry_timestamp
  FROM error_record_by_id
  WHERE error_record_id = ?
```

- **Consistency Level**: Configurable (default: `LOCAL_ONE`).
- **Async execution**: Uses `CqlSession.executeAsync()` for parallel lookups.
- **Batch processing**: Splits large UUID sets into configurable batch sizes.

### 3.5 `CassandraConfiguration`

Programmatic Cassandra driver configuration (no `application.conf`):

| Setting                        | Value / Config Key                        |
|--------------------------------|-------------------------------------------|
| Contact Points                 | `ecs.cassandra.contactPoint` (comma-separated) |
| Port                           | `ecs.cassandra.port`                      |
| Local Datacenter               | `ecs.cassandra.localDatacenter`           |
| Keyspace                       | `ecs.cassandra.keyspace`                  |
| Consistency Level              | `ecs.cassandra.consistencyLevel`          |
| Pool Local Size                | `ecs.cassandra.poolLocalSize`             |
| Pool Remote Size               | `ecs.cassandra.poolRemoteSize`            |
| Request Timeout                | `ecs.cassandra.requestTimeoutMs`          |
| Connection Timeout             | `ecs.cassandra.connectionTimeoutMs`       |
| Reconnection Base Delay        | `ecs.cassandra.reconnectionBaseDelayMs`   |
| Reconnection Max Delay         | `ecs.cassandra.reconnectionMaxDelayMs`    |
| Retry Policy                   | Default driver retry                       |

---

## 4. ECS End-to-End Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                    jitr-ub-msg (Mediation Engine)                │
│                                                                  │
│  1. Begin processing file                                        │
│     → ECSKafkaSender.sendHandshaketoECS(INITIAL)                │
│                                                                  │
│  2. Decode record → Validation fails                             │
│     → Create ECSError{rawBytes, metadata}                        │
│     → Accumulate in fileInfo.ecsErrorList                        │
│                                                                  │
│  3. End of file                                                  │
│     → ECSKafkaSender.sendErrorRecordToKafka(ecsErrorList)       │
│     → ECSKafkaSender.sendHandshaketoECS(COMMIT)                 │
│                                                                  │
│  (If error occurs at any step)                                   │
│     → ECSKafkaSender.sendHandshaketoECS(ROLLBACK)               │
└─────────────────────────────────────┬───────────────────────────┘
                                      │
                        Kafka ECS Topic│
                                      ▼
┌─────────────────────────────────────────────────────────────────┐
│              ECS Downstream Consumers (External)                 │
│                                                                  │
│  - Receive error records from Kafka                              │
│  - Apply error correction rules                                  │
│  - Store corrected records in Cassandra (error_record_by_id)    │
│  - Generate reflow request files (with error_record_id UUIDs)   │
└─────────────────────────────────────┬───────────────────────────┘
                                      │
                   Reflow Request File │ (UUID list)
                                      ▼
┌─────────────────────────────────────────────────────────────────┐
│             jitr-ub-msg (ECS Reflow Workflow)                    │
│                                                                  │
│  1. Pick up reflow file from dwf_umm_reprocess/{type}/           │
│  2. Read error_record_id UUIDs from file                         │
│  3. EcsErrorCdrCollectionService.fetchErrorCdrsById(uuids)      │
│     → Async Cassandra lookups → ErrorRecordEntity objects        │
│  4. Reconstruct UsageRecord from raw bytes                       │
│  5. Re-run validate → enrich → route pipeline                   │
│  6. Output to normal JITR routing paths                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Shared Kafka Library — `jitr-kafka-library`

### 5.1 Library Identity

| Attribute        | Value                            |
|------------------|----------------------------------|
| **Artifact ID**  | `jitr-kafka-library`             |
| **Type**         | JAR Library                      |
| **Consumers**    | `jitr-ub-msg-dist`, `jitr-ub-msg`|

### 5.2 `SpringKafkaProducerAsync`

Asynchronous Kafka producer with file-based failure recovery:

```java
public CompletableFuture<SendResult<String, String>> sendKafkaText(
    String topic, String key, String value)
```

**Success path**: Returns `CompletableFuture<SendResult>` with topic/partition/offset.

**Failure path**: On `ProducerRecord` send failure:
1. Constructs a JSON record: `{topic, key, value, partition, timestamp}`.
2. Writes to a local file at `jitr.kafka.library.path` directory.
3. Filename: `{topic}_{timestamp}.json`.

### 5.3 `KafkaRetryFailureMechanism`

Cron-scheduled retry for failed Kafka messages:

```java
@Scheduled(cron = "${jitr.kafka.library.retryCron}")
public void retryFailedMessages()
```

1. Scans `jitr.kafka.library.path` for `*.json` files.
2. Reads each file → deserializes to `ProducerRecord` fields.
3. Resends via `KafkaTemplate.send()` (without retry flag to prevent infinite loop).
4. On success: deletes the retry file.
5. On failure: leaves the file for next cron cycle.

---

*Continue to [Part 6: Data Processing — DSL & 5G Pipeline](06_Data_Processing_DSL_and_5G.md)*
