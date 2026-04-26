# Usage Broker (UB) Ecosystem — Technical Specification

## Part 2: File Distributor — `jitr-ub-msg-dist`

---

## 1. Application Identity

| Attribute           | Value                                          |
|---------------------|-------------------------------------------------|
| **Artifact ID**     | `jitr-ub-msg-dist`                              |
| **Group ID**        | `com.vzw.jitr`                                  |
| **Main Class**      | `NetworkUsageDistributorApp`                     |
| **Spring Profile**  | `msg-dist`                                       |
| **Port**            | 14190                                            |
| **Scheduler**       | Spring `@EnableScheduling` with Cron triggers    |
| **Parent POM**      | `ub-jitr-domain-parent:23.12.100-SNAPSHOT`       |

---

## 2. Purpose

The **File Distributor** is the **entry point** of the UB MSG pipeline. It solves the problem of:

- **Ingesting** raw usage files deposited by the Data Manager (DM) onto a shared filesystem.
- **Batching** individual usage records into optimally-sized JSON payloads.
- **Publishing** these payloads to Kafka for downstream consumption (by `jitr-data-dsl`).
- **Auditing** at the file level (record counts, byte sizes, timestamps).
- **Archiving** processed files (compressed GZip or raw move).

It handles **four usage types** in parallel via dedicated cron-scheduled processors.

---

## 3. Component Breakdown

### 3.1 Class Hierarchy

```
NetworkUsageDistributorApp (Spring Boot Main)
  └─ NUFileProcessorScheduler (Scheduler Service)
       ├─ LSMDistributorProcessor  ─┐
       ├─ MMSDistributorProcessor   ├─ extends BaseDistributorProcessor
       ├─ RCSDistributorProcessor   │     implements DistributorProcessor (Runnable)
       └─ RecycleDistributorProcessor─┘
```

### 3.2 Core Classes

#### `NetworkUsageDistributorApp`
- **Role**: Spring Boot entry point implementing `CommandLineRunner`.
- **Behavior**: On startup, invokes `NUFileProcessorScheduler.startSchedulerJobs()`.
- **Component Scan**: `com.vzw.billing.umm.distributor`, `com.vzw.billing.kafka`

#### `NUFileProcessorScheduler`
- **Role**: Configures and starts the four distributor processors on cron schedules.
- **Key Logic**:
  1. Builds `BaseWorkFlowConfig` from Spring `Environment` properties.
  2. Validates directory paths exist (creates if missing).
  3. Initializes a `FixedThreadPool` (configurable, default 25 threads).
  4. Creates 4 `Runnable` processor instances (LSM, MMS, RCS, Recycle).
  5. Schedules each on the same cron pattern via `TaskScheduler.schedule()`.

#### `BaseDistributorProcessor` (Abstract)
- **Role**: Core processing logic shared by all four processors.
- **Key Method — `run()`**:
  1. Checks `config.isActive()` — respects suspend/resume.
  2. Calls `pickInputFilePathsToProcess()` — returns a stream of file paths.
  3. Submits each file to the thread pool via `ExecutorService.submit()`.
  4. Waits for all futures to complete.

- **Key Method — `processFile(Path)`**:
  1. Extracts metadata from filename (usage type code, switch name).
  2. Sets `sourceIdentifierCode` (`UB_JAVA` for normal, `REED_RCL` for recycle).
  3. Reads file line-by-line using `Files.lines()`.
  4. Calls `format(rawUsage, fileInfo)` (subclass-specific).
  5. Batches records (up to `maxRecordToBeProcessed` = 200) with `~` delimiter.
  6. Publishes each batch to Kafka via `SpringKafkaProducerAsync.sendKafkaText()`.
  7. Publishes a file-level audit JSON to the audit Kafka topic.
  8. Archives the file (either raw move or GZip compress).
  9. On error: moves file to error directory.
  10. On OOM: suspends processing (`config.setActive(false)`).

#### `DistributorProcessor` (Interface)
```java
public interface DistributorProcessor extends Runnable {
    String getInputFilePattern();    // Regex for file selection
    void processFile(Path filePath); // File processing logic
    String format(String usage, FileInfo fileInfo) throws Exception; // Record formatting
    String getSwitchType();          // e.g., "LucentSMS", "MotorolaMMS"
}
```

### 3.3 Processor Implementations

| Processor                  | Switch Type     | Input File Pattern Config Key                    | Format Logic                              |
|----------------------------|-----------------|--------------------------------------------------|-------------------------------------------|
| `LSMDistributorProcessor`  | `LucentSMS`    | `jitr.ub.msg.dist.lsm.input.file.pattern`       | Parse JSON, append `~` delimiter          |
| `MMSDistributorProcessor`  | `MotorolaMMS`   | `jitr.ub.msg.dist.mms.input.file.pattern`       | Parse JSON, clone/split MO multi-recipient records |
| `RCSDistributorProcessor`  | `RCS`           | `jitr.ub.msg.dist.rcs.input.file.pattern`       | Parse JSON, append `~` delimiter          |
| `RecycleDistributorProcessor`| `RECYCL`      | `jitr.ub.msg.dist.recycle.input.file.pattern`    | Parse JSON, detect sub-type from filename |

#### MMS Clone/Split Logic (Critical Business Rule)

For MMS MO (Mobile Originated) records with **multiple recipients** (semicolon-separated `recipientAddress`):

1. Original record is split into N individual records (one per recipient).
2. Each clone gets:
   - Its own `recipientAddress` (single value)
   - Updated `GRI` value (split count in position `[length-2]`)
   - Updated `auditID` (sequence number embedded in positions 3-4)
   - Corresponding `originalRecipientBillingIDs` entry
3. `FileInfo.cloneCount` is incremented by `(N - 1)` for audit accuracy.

---

## 4. File Naming Convention

Input files follow a strict naming pattern:

```
UB.{DCCode}.MSG.{UsageType}.{Version}.{SwitchName}.{SeqPart}...{Date}_{Time}.{SeqNum}
```

Example: `UB.VCO1.MSG.LSM.V.SM39SL.343...20110810_220100.000329998`

Parsed fields (split by `.`):
- `[2]` = `MSG` (label)
- `[3]` = Usage Type Code (LSM, MMS, RCS)
- `[5]` = Switch Name
- `[9]` = File Stored Timestamp (`yyyyMMdd_HHmmss`)

---

## 5. Kafka Integration

### 5.1 Published Messages

**Raw Usage Topic** (`jitr.ub.msg.dist.rawTopic`):
```json
{
  "fileName": "UB.VCO1.MSG.LSM.V.SM39SL.343...20110810_220100.000329998",
  "switchType": "LucentSMS",
  "rawRecord": "<record1_json>~<record2_json>~...<recordN_json>"
}
```
- Records within `rawRecord` are delimited by `~` (tilde).
- Batch size: up to 200 records per message.

**Audit Topic** (`jitr.ub.msg.dist.auditTopic`):
```json
{
  "cloneCount": 0,
  "hostName": "server01",
  "fileName": "UB.VCO1.MSG.LSM.V.SM39SL.343...20110810_220100.000329998",
  "usageTypeCode": "LSM",
  "sourceIdentifierCode": "UB_JAVA",
  "switchName": "SM39SL",
  "dmUsageTypeCode": "LSM",
  "startTime": "2024-01-15 10:30:01.500",
  "endTime": "2024-01-15 10:30:02.100",
  "decodedCount": 118,
  "inputCount": 118,
  "inputBytesNumber": 20237,
  "decodedBytesNumber": 20237,
  "errorCount": 0,
  "errorBytesNumber": 0,
  "duplicateCount": 0,
  "duplicateBytesNumber": 0,
  "splitCount": 0,
  "clonedBytesNumber": 0
}
```

### 5.2 Kafka Producer Configuration

| Property                                     | Value      |
|----------------------------------------------|------------|
| `spring.kafka.producer.compression-type`     | `snappy`   |
| `spring.kafka.producer.properties.enable.idempotence` | `true` |
| Bootstrap servers                             | Configurable via `@UB.MSG.DIST.KAFKA.BOOTSTRAP.SERVERS@` |

---

## 6. File Lifecycle

```
Input Directory
    │
    ├─ (Cron poll) ──▶ File regex match per usage type
    │
    ▼
Input Staging Directory (per hostname)
    │
    ├─ (Process) ──▶ Read → Format → Batch → Kafka Publish
    │
    ├─ Success ──▶ Archive Directory
    │               ├─ archiveRawFile=true  → raw file move
    │               └─ archiveRawFile=false → GZip compress + delete original
    │
    └─ Error ──▶ Error Directory (raw file moved)
```

### 6.1 File Pickup Strategy

- **Unix/Linux**: Uses `find` command with POSIX regex via `ProcessBuilder` for performance.
- **Windows**: Uses `Files.walk()` with regex filter (development support).
- Files are **moved atomically** from input → staging to prevent double-processing by concurrent instances.
- Maximum files per poll cycle: configurable (`maxFilesToBeProcessed` = 100).

---

## 7. Configuration Reference

| Property                                | Default | Description                                |
|-----------------------------------------|---------|--------------------------------------------|
| `jitr.ub.msg.dist.inputPath`            | —       | Root input directory for DM files          |
| `jitr.ub.msg.dist.inputStagingPath`     | —       | Per-host staging directory                 |
| `jitr.ub.msg.dist.errorFilePath`        | —       | Error file destination                     |
| `jitr.ub.msg.dist.archivePath`          | —       | Archive destination                        |
| `jitr.ub.msg.dist.archiveRawFile`       | false   | If true, archive as-is; if false, GZip     |
| `jitr.ub.msg.dist.maxRecordToBeProcessed`| 200    | Records per Kafka batch                    |
| `jitr.ub.msg.dist.maxFilesToBeProcessed` | 100    | Files per cron cycle                       |
| `jitr.ub.msg.dist.maxFileThread`         | 25     | Thread pool size for parallel file processing |
| `jitr.ub.msg.dist.scheduleCronPattern`   | —      | Cron expression for polling                |
| `jitr.ub.msg.dist.rawTopic`             | —       | Kafka topic for raw usage                  |
| `jitr.ub.msg.dist.auditTopic`           | —       | Kafka topic for audit records              |

---

## 8. Error Handling & Resilience

| Scenario                  | Behavior                                                    |
|---------------------------|-------------------------------------------------------------|
| File processing exception | File moved to `errorFilePath`; processing continues for other files |
| `OutOfMemoryError`        | Processing **suspended** (`config.setActive(false)`); file moved to error; heap stats logged |
| Kafka send failure        | `jitr-kafka-library` writes failed message to local file for retry |
| Invalid config on startup | Application shutdown (`ConfigurableApplicationContext.close()`) |
| Kafka idempotence         | Enabled (`enable.idempotence=true`) to prevent duplicate publishes |

---

## 9. Data Transfer Objects

### `FileInfo`
Tracks per-file processing state:
- `inputFilePath`, `inputFileName`, `switchName`, `hostName`
- `dmUsageTypeCode`, `sourceIdentifierCode`
- `usageCount`, `cloneCount`, `usageBatchSize`
- `batchStartTimestamp`, `batchEndTimestamp`
- `inputFileSize`, `seqNum`

### `UsageAudit`
File-level audit record published to Kafka:
- All `FileInfo` fields plus computed metrics
- `decodedCount`, `errorCount`, `duplicateCount`, `splitCount`
- `inputBytesNumber`, `decodedBytesNumber`, `errorBytesNumber`

---

*Continue to [Part 3: jitr-ub-msg-decoder (Decoder Library)](03_Decoder_Library_jitr_ub_msg_decoder.md)*
