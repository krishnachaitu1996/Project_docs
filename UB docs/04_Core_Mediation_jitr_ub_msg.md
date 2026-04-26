# Usage Broker (UB) Ecosystem — Technical Specification

## Part 4: Core Mediation Engine — `jitr-ub-msg`

---

## 1. Application Identity

| Attribute           | Value                                          |
|---------------------|-------------------------------------------------|
| **Artifact ID**     | `jitr-ub-msg`                                    |
| **Main Class**      | `NetworkUsageMediationApp`                       |
| **Spring Profile**  | `UBMSG`                                          |
| **Port**            | 14060                                            |
| **Service Discovery**| Eureka (`@EnableDiscoveryClient`)               |
| **Component Scan**  | `com.vzw.billing`                                |
| **Key Dependencies**| `jitr-ub-msg-decoder`, `jitr-ub-ecs-distributor`, `jitr-ub-ecs-cdr-generator` |

---

## 2. Purpose

The **Core Mediation Engine** is the **heart of the UB ecosystem**. It is the most complex application, responsible for:

- **Decoding** raw usage files using the decoder library (BCD → JSON, ASCII → JSON, etc.)
- **Validating** each record against business rules (MDN, dates, record types, duplicates)
- **Enriching** records with internal fields (routing info, timestamps, identifiers)
- **Routing** valid UDRs to the correct JITR rating instance and auxiliary sinks
- **Error handling** via the ECS subsystem (Kafka + Cassandra)
- **Auditing** at file and record level to Oracle (MZAUD)
- **Reprocessing** errored records from ECS and input staging

---

## 3. Architecture

### 3.1 Module Structure

```
NetworkUsageMediationApp (Spring Boot Main)
  └─ NUFileProcessorScheduler
       ├─ Normal Processing
       │    ├─ LSMMediationProcessor   ─┐
       │    ├─ MMSMediationProcessor    ├─ extends BaseMediationProcessor
       │    ├─ RCSMediationProcessor    │     implements MediationProcessor
       │    └─ SMSMediationProcessor   ─┘
       │
       └─ Reprocess Processing
            └─ MediationReprocessor (for input staging stuck files)
```

### 3.2 Package Organization

| Package                          | Responsibility                                        |
|----------------------------------|-------------------------------------------------------|
| `config`                         | Workflow configuration, DB connections, app constants  |
| `core`                           | Base processor, interface, processor implementations  |
| `core.reprocess`                 | Reprocess decoder for ECS error files                 |
| `db.dao`                         | Oracle DAO (sequence numbers, audit)                  |
| `db.entity`                      | JPA/Oracle entities                                   |
| `dto`                            | Data transfer objects (FileInfo, UsageRecord, UDRContext) |
| `endpoint`                       | Actuator custom endpoints (suspend/resume/status)     |
| `file.action`                    | File move, compress, delete operations                |
| `file.audit`                     | Audit service for Oracle MZAUD                        |
| `file.filter`                    | Duplicate check and other filters                     |
| `file.router`                    | Usage routing orchestration                           |
| `file.writer`                    | Buffered text file writer with WriterFactory          |
| `process.impl`                   | Usage validators and senders per type                 |
| `udr.router`                     | UDR file-based output routing                         |
| `udr.error`                      | ECS service integration                               |
| `utils`                          | Constants, helpers, date utilities                    |

---

## 4. Processing Pipeline

### 4.1 Normal File Processing Flow

```
1. Cron Trigger (per usage type: LSM, MMS, RCS, SMS)
   │
2. pickInputFilePathsToProcess()
   │  - Uses OS `find` command (Unix) or Files.walk (Windows)
   │  - Respects maxFilesToBeProcessed limit
   │
3. For each file → Submit to ExecutorService thread pool
   │
4. processFile(Path inputFilePath, boolean ecsBAUFileFlag)
   │
   ├─ 4a. Move file: Input → Staging (per-hostname subdirectory)
   │
   ├─ 4b. validate(filePath)
   │       - Duplicate batch check (DUP_CHK filter)
   │       - File naming validation
   │
   ├─ 4c. readFileRelatedInfo(filePath) → FileInfo
   │       - Begin Batch: Get audit sequence ID from Oracle
   │       - Extract switch name, usage type from filename
   │
   ├─ 4d. getDecoder(filePath, auditSeqValue) → UsageDecoder
   │       - Instantiate appropriate decoder (LSM/MMS/RCS/SMS)
   │       - decoder.openFile()
   │
   ├─ 4e. DECODING LOOP:
   │       while (decodedJson = decoder.decode()) != null:
   │         - Create UsageRecord(decodedJson, rawBytes)
   │         - Accumulate into batch (up to maxRecordToBeProcessed=1000)
   │         - When batch full → usageRouter.route(UDRContext)
   │
   ├─ 4f. Route remaining records
   │
   ├─ 4g. Flush ECS error batch (if any pending)
   │       - ecsKafkaSender.sendErrorRecordToKafka(fileInfo.ecsErrorList)
   │
   ├─ 4h. usageRouter.drainBuffers(RouteInfo)
   │       - Flush all buffered output records to files
   │
   ├─ 4i. addTrailer(fileInfo)
   │       - Write trailer/footer to output files
   │
   ├─ 4j. WriterFactory.closeAll(inputFileName)
   │       - Close all temp output files, return file paths/sizes
   │
   ├─ 4k. onFileProcessingComplete(fileInfo, decoder)
   │       - End Batch: Write audit record to Oracle MZAUD
   │       - Rename temp files to final names
   │
   ├─ 4l. doFilesTransferToOutputFolder(fileInfo)
   │       - Move output files from staging → output directories
   │
   ├─ 4m. ECS Handshake: COMMIT
   │       - sendHandshaketoECS(COMMIT)
   │
   └─ 4n. Archive: Staging → Archive (GZip or raw move)
```

### 4.2 Error Handling Flow (Cancel Batch)

When any exception occurs during processing:

1. Set `isErrorFile = true`.
2. Close the decoder's file handle.
3. Execute `cancelBatch()`:
   - Delete any partial temp output files.
   - Write cancel-batch audit entry to Oracle.
   - Move input file to error directory.
4. ECS Handshake: **ROLLBACK**.
5. On `Error` (e.g., `OutOfMemoryError`):
   - **Suspend ALL processes** (`ubmsgFlag.setAppFlags("ALL", false)`).
   - Move file to error directory.

---

## 5. Routing Architecture

### 5.1 `UsageRouter` — Routing Orchestration

The routing chain follows this pattern per record:

```
UsageRouter.route(UDRContext)
  │
  ├─ UsageValidator.validateUsage(fileInfo, usageRecord)
  │    - MDN validation
  │    - Date/time validation
  │    - Record type validation
  │    - Business rule filters
  │
  ├─ InternalFieldsPopulator.populate(...)
  │    - Resolve BillerID → JITR Instance (V→RO, A→RT, Q→GT, N→RS, B→BS)
  │    - Normalize timezone offsets
  │    - Set internal routing codes
  │
  └─ UsageSender.sendToOutputFiles(routeInfo, usageRecord, refData)
       - Determine output route(s) based on biller ID
       - Write record to appropriate UdrRouter
       - Handle special cases (robocall, REVO, forward, LL audit)
```

### 5.2 `UdrRouter` — Buffered File Writer

`UdrRouter` implements a **buffered write pattern** for performance:

1. Records are accumulated in a `Map<String, List<Object>>` (route ID → buffered records).
2. When buffer reaches threshold (default: 1000 records), flush to file via `TextFileWriter`.
3. At end-of-file, `drainBuffers()` flushes all remaining records.
4. `WriterFactory` manages `TextFileWriter` instances per (config, inputFileName, routeId).

### 5.3 Output Route Mapping

For each usage type, records are routed to multiple output directories:

| Route ID     | Directory Pattern                           | Content                      |
|--------------|---------------------------------------------|------------------------------|
| `jitrRT`     | `dwf_jitr_msg/rt/`                          | JITR RT instance UDRs        |
| `jitrRO`     | `dwf_jitr_msg/ro/`                          | JITR RO instance UDRs        |
| `jitrGT`     | `dwf_jitr_msg/gt/`                          | JITR GT instance UDRs        |
| `jitrRS`     | `dwf_jitr_msg/rs/`                          | JITR RS instance UDRs        |
| `jitrBS`     | `dwf_jitr_msg/bs/`                          | JITR BS instance UDRs        |
| `LL`         | `dwf_ll/FS{S,M,R}/`                         | Low-level audit records      |
| `LL_Trailer` | `dwf_ll/FS{S,M,R}/`                         | Low-level audit trailer      |
| `RSS`        | `dwf_merge/rss/{SMS,MMS,RCS}/`              | RSS merge feed               |
| `AFB`        | `dwf_merge/afb/{SMS,MMS}/`                  | AFB feed (LSM, MMS only)     |
| `OPC`        | `dwf_merge/opc/{LSM,MMS}/`                  | OPC feed (LSM, MMS only)     |
| `TA`         | `dwf_cfi/THINAIR/`                          | ThinAir                      |
| `TA5`        | `dwf_cfi/THINAIR5/`                         | ThinAir 5G                   |
| `VISIBLE`    | `dwf_cfi/visible/{SMS,MMS,RCS}/`            | Visible brand                |
| `robocall`   | `dwf_original/{LSM,MMS,RCS}/`               | Robocall originals           |
| `droppedRevo`| `dwf_revo/report_dropped/`                  | REVO dropped reports         |
| `exceptionRevo`| `dwf_revo/report_exceptions/`             | REVO exception reports       |
| `forward`    | `dwf_original/{MMS}/`                       | MMS forwarding               |
| `error`      | Error directory                              | Unroutable records           |
| `vision`     | Vision routing                               | Direct Vision feed           |

---

## 6. Validation & Enrichment

### 6.1 Validators (per usage type)

| Validator              | Usage Type | Key Validations                                    |
|------------------------|------------|----------------------------------------------------|
| `LSMUsageValidator`    | LSM        | Structure code, date validity, MDN format           |
| `MMSUsageValidator`    | MMS        | Record type (MO/MT), date expiry, recipient fields  |
| `RCSUsageValidator`    | RCS        | Subscription ID type, MDN parsing, service timestamp|
| `LSMUsageValidator`    | SMS        | (shared with LSM)                                   |

### 6.2 Internal Fields Populators

| Populator                    | Usage Type | Key Enrichments                                  |
|------------------------------|------------|--------------------------------------------------|
| `LSMInternalFieldsPopulator` | LSM        | BillerID lookup, timezone offset, GRI update     |
| `MMSInternalFieldsPopulator` | MMS        | Domain mapping, MO/MT routing, recipient parsing |
| `RCSInternalFieldsPopulator` | RCS        | SIP/TEL MDN extraction, subscription ID handling |

### 6.3 Biller ID → JITR Instance Mapping

The first character of the Biller ID determines the JITR routing destination:

```java
MSG_BILLERID_JITR_INST_MAP.put("V", "RO");  // Vision → JITR RO
MSG_BILLERID_JITR_INST_MAP.put("A", "RT");  // → JITR RT
MSG_BILLERID_JITR_INST_MAP.put("Q", "GT");  // → JITR GT
MSG_BILLERID_JITR_INST_MAP.put("N", "RS");  // → JITR RS
MSG_BILLERID_JITR_INST_MAP.put("B", "BS");  // → JITR BS
MSG_BILLERID_JITR_INST_MAP.put("R", null);  // RSS routing
MSG_BILLERID_JITR_INST_MAP.put("O", null);  // OPC routing
MSG_BILLERID_JITR_INST_MAP.put("T", null);  // ThinAir routing
MSG_BILLERID_JITR_INST_MAP.put("5", null);  // ThinAir 5G routing
MSG_BILLERID_JITR_INST_MAP.put("P", null);  // Special handling
```

### 6.4 MMS Domain Mapping

MMS records use domain-based routing to determine the origination path:

```java
DOMAIN_MAP.put("vzwpix.com", "MM1");
DOMAIN_MAP.put("mm4.inphomatch.com", "MM1");
DOMAIN_MAP.put("mms.verisign.com", "MM1");
DOMAIN_MAP.put("mm4.verizon.com", "MM1");
DOMAIN_MAP.put("icmms1.sun5.lightsurf.net", "MM1");
```

---

## 7. Database Interactions

### 7.1 Oracle Database Connections

| Database    | Purpose                        | Connection Config         |
|-------------|--------------------------------|---------------------------|
| **MZAUD**   | Audit records (read/write)     | `spring.mzaud.*`          |
| **MZADMIN** | Configuration & reference data | `spring.mzadmin.*`        |

Both use HikariCP connection pooling with:
- `minimum-idle`: 2
- `maximum-pool-size`: 20
- `idle-timeout`: 300,000 ms
- `max-lifetime`: 120,000 ms
- `connection-test-query`: `SELECT 1 from dual`

### 7.2 Key Database Operations

| Operation              | Database | Description                                      |
|------------------------|----------|--------------------------------------------------|
| Begin Batch            | MZAUD    | Get next audit sequence ID                       |
| End Batch              | MZAUD    | Write file-level audit record (counts, times)   |
| Cancel Batch           | MZAUD    | Write cancellation audit with error code         |
| Duplicate Check        | MZAUD    | Check if file has been processed before          |
| BillerID Lookup        | MZADMIN  | Bulk lookup for MDN → BillerID mapping           |
| Reference Data Cache   | MZADMIN  | Load and cache reference tables (daily refresh)  |
| Sequence Number        | MZAUD    | DSS sequence number generation                   |

### 7.3 Reference Data Cache

Reference data is loaded at startup and refreshed daily (`refresh.db.cache.cron.job.time=0 1 0 * * *`):

- Error code reference table
- MARS network mappings
- Configuration parameters
- Process table definitions

---

## 8. Workflow Configurations

The application supports **multiple concurrent workflows**, each configured via properties:

### 8.1 Normal Processing Workflows

| Workflow Key | Process Name | Input Source         | Active |
|--------------|-------------|----------------------|--------|
| `lsm`        | LSM_Num     | DM LSM directory     | true   |
| `mms`        | MMS_Num     | DM MMS directory     | true   |
| `rcs`        | RCS_Num     | DM RCS directory     | true   |
| `sms`        | SMS_Num     | DM SMS directory     | true   |

### 8.2 Reprocessing Workflows

| Workflow Key        | Process Name      | Input Source              | Schedule   |
|---------------------|-------------------|---------------------------|------------|
| `lsm_numreprocess`  | LSM_NumReprocess  | Reprocess input directory | Every 60s  |
| `mms_numreprocess`  | MMS_NumReprocess  | Reprocess input directory | Every 60s  |
| `rcs_numreprocess`  | RCS_NumReprocess  | Reprocess input directory | Every 60s  |

### 8.3 ECS Reflow Workflows

| Workflow Key        | Process Name      | Input Source                       | Schedule   |
|---------------------|-------------------|------------------------------------|------------|
| `lsm_ecsreflow`     | LSM_EcsReflow     | `dwf_umm_reprocess/LSM/`          | Every 60s  |
| `mms_ecsreflow`     | MMS_EcsReflow     | `dwf_umm_reprocess/MMS/`          | Every 60s  |
| `rcs_ecsreflow`     | RCS_EcsReflow     | `dwf_umm_reprocess/RCS/`          | Every 60s  |

---

## 9. Reprocessing Architecture

### 9.1 Three Reprocessing Paths

1. **Input Staging Reprocess** (`NumReprocess`):
   - Picks up files stuck in input staging (older than 120 minutes).
   - Re-runs full decode → validate → enrich → route pipeline.

2. **ECS Reflow** (`EcsReflow`):
   - Picks up files from `dwf_umm_reprocess/{type}/` directory.
   - These files contain error record IDs (not raw records).
   - Fetches raw CDR bytes from Cassandra via `EcsErrorCdrCollectionService`.
   - Re-runs enrichment and routing.

3. **Error File Reprocess**:
   - Files in error directories can be manually moved back to input.
   - Detected by `isReprocessingFile()` in `BaseMediationProcessor`.

### 9.2 ECS Reflow Detail

```
ECS Reflow File (contains error_record_ids)
    │
    ├─ Read error_record_id (UUID) from file
    │
    ├─ Batch lookup: EcsErrorCdrCollectionService.fetchErrorCdrsById(Set<UUID>)
    │    └─ Async Cassandra queries: SELECT * FROM error_record_by_id WHERE error_record_id = ?
    │
    ├─ Reconstruct UsageRecord from:
    │    - raw_error_record (byte[])
    │    - raw_error_record_txt (String)
    │    - revo_error_record (String)
    │
    └─ Route through standard pipeline (validate → enrich → output)
```

---

## 10. ELK / Kibana Logging

Each file processing cycle generates structured ELK log entries:

```json
{
  "app": "jitr-ub-msg",
  "processTime": 1523,
  "inputFileName": "UB.VCO1.MSG.LSM.V.SM39SL.343...",
  "inputRecordCount": 118,
  "outputRecordCount": 116,
  "errorRecordCount": 2,
  "statusCode": "Success",
  "errorCode": null,
  "msgType": "String"
}
```

On error:
```json
{
  "statusCode": "cancelbatch",
  "errorCode": "E_FILE_UNDECODABLE",
  "outputRecordCount": 0
}
```

---

## 11. Suspend / Resume Mechanism

The application implements a **per-process suspend/resume** mechanism via custom Actuator endpoints:

- `UBMSGFlag` bean tracks running status per process name.
- `RUNNING_PROCESS_STATUS_MAP`: `Map<String, AtomicBoolean>` — per-workflow active flag.
- Suspend endpoint: Sets flag to `false` → processors skip execution on next cron tick.
- Resume endpoint: Sets flag to `true` → processing resumes.
- On OOM Error: **All processes suspended** automatically.

---

*Continue to [Part 5: Error Correction Subsystem (ECS)](05_ECS_Subsystem.md)*
