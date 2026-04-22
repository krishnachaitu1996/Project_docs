# JITR Recon — Error Handling & Resiliency

## 1. Custom Exception Hierarchy

```
java.lang.Exception
├── DataValidationException         ← Data/input validation failures
├── CustomizedException             ← Generic application-level exceptions
└── RestageDissimilarFileException  ← Restage file format mismatches

Error Objects (not exceptions):
├── ReconError                      ← Error metadata container with error codes
└── ReconErrorHandler (@Component)  ← Centralized error formatting and DB insertion
```

### 1.1 DataValidationException

**Purpose:** Thrown when input data fails validation — file format errors, unexpected field counts, invalid date formats, missing required fields.

**Used by:** `FileValidator`, `VisionFileFormatter`, `ReconSplitter`, prorate processors

### 1.2 CustomizedException

**Purpose:** Generic exception wrapper for unexpected application errors. Carries context about the processing phase and entity being processed.

### 1.3 RestageDissimilarFileException

**Purpose:** Thrown specifically during restage operations when the file being restaged doesn't match the expected format or schema of the target file. Critical for Vision 2.0 restage scenarios.

### 1.4 ReconError (Error Metadata)

**Purpose:** Not an exception — a structured error data object carrying error context through the processing pipeline.

**Key Fields:**
| Field | Purpose |
|-------|---------|
| `errorCode` | Categorized error identifier |
| `custAcctNumber` | Customer/account that caused the error |
| `processType` | Processing phase (EXTRACT, COMPARE, FIX) |
| `billCycle` | Bill cycle being processed |
| `errorMap` | Detailed error data (multiple key-value pairs) |

### 1.5 ReconErrorHandler (@Component)

**Purpose:** Centralized error handling service. Formats errors and inserts them into the UBSR database for tracking.

**Capabilities:**
- Error formatting with template-based messages from `messages.properties`
- Database insertion of error records
- UBSR-specific error logging
- Error aggregation for batch reporting

---

## 2. Threshold-Based Alerts & Hold System

### 2.1 Delete Threshold Checking

The system prevents mass-deletes by comparing the count of `NON_UBSR_ONLY` (delete) records against configured thresholds. If the count exceeds the threshold, processing is **halted** and an alert is sent.

**Threshold Configuration (per batch mode, per entity, per zone type):**

| Entity Type | PREBILL Threshold | MINI Threshold | FULL Threshold | ODR Threshold |
|-------------|------------------|---------------|---------------|--------------|
| CUSTOMER | 100 | 100 | 100 | 100 |
| BLACCT | 10,000 | 10,000 | 10,000 | 10,000 |
| LNSVCPROD | 15,000 | 20,000 | 40,000 | 1,000 |
| CUSTDVCEQPTRANS | 100,000 | 300,000 | 300,000 | 100,000 |
| PPLANMTN | 30,000 | 30,000 | 40,000 | 5,000 |
| PROMOCUSTMTN | 50,000 | 50,000 | 50,000 | 3,000 |
| EVSRC | 30,000 | 30,000 | 70,000 | 6,000 |
| CPARD | 1,000,000 | 1,000,000 | 1,000,000 | 1,000,000 |
| SFOPD | 20,000 | 20,000 | 30,000 | 3,000 |
| MIGRATIONMETADATA | 1,000,000 | 1,000,000 | 1,000,000 | 1,000,000 |

### 2.2 Insert Threshold Checking

Insert thresholds use a **percentage-based** approach:
```properties
percent.value.insert.threshold=10
percent.value.insert.vision2.threshold=10
```
If inserts exceed 10% of total records, processing is halted.

### 2.3 Delete Sample Percentage

Before allowing mass deletes, a sample is evaluated:
```properties
deleteSamplePercentage=1
```
1% of proposed deletions are checked for correctness before proceeding.

### 2.4 DynamicThresholdLimits

`DynamicThresholdLimits` (`com.verizon.recon.util`) calculates thresholds dynamically based on:
- Current record counts
- Historical baseline
- Entity-specific multipliers
- B2B vs. consumer segment

### 2.5 Thresholds Class

`Thresholds` (`com.verizon.recon.util`) manages the threshold evaluation lifecycle:
- Loads threshold configuration per batch mode
- Compares actual counts against limits
- Returns pass/fail result with violation details

### 2.6 Hold Evaluation Service

`HoldEvaluationService` (`com.verizon.jitr.recon.hold`) provides a comprehensive evaluation before allowing delete operations:

```
HoldEvaluationService.evaluate()
├── Enabled by: deleteHoldEvaluationEnabled=Y
├── Parse diff files for delete-eligible records
├── Sample deleteSamplePercentage (1%) of deletions
├── Check against DVS data (PplanMtnInfo, PromoCustMtnInfo, SpecialFeatureMtnInfo)
├── Evaluate against billing.hold.check.enabled thresholds
│
├── Result: PASS → Continue processing
├── Result: HOLD → Stop, send email alert, create HOLD file
└── Result: ERROR → Stop, log error, alert operations
```

**Threshold Flags:**
| Flag | Purpose |
|------|---------|
| `billing.ubsronly.threshold.check.enabled` | Enable billing-side delete threshold |
| `rating.nonubsronly.threshold.check.enabled` | Enable rating-side delete threshold |
| `rating.ubtoub.nonubsronly.threshold.check.enabled` | UBTOUB rating delete threshold |
| `rating.ubsronly.threshold.check.enabled` | UBSR insert threshold |
| `billing.hold.check.enabled` | Billing hold evaluation |
| `deleteHoldEvaluationEnabled` | Master delete hold switch |

---

## 3. REED Error Recovery System

REED (Reconciliation Error Event Data) is a dedicated error tracking and recovery subsystem with its own database schema.

### 3.1 Error Recording

```
Fix Processor Failure
    │
    ▼
ReconReed (DAO)
├── Insert error into REED schema:
│   ├── ehCaseId (case tracking)
│   ├── statusCd (error status)
│   ├── custIdNo, acctNo, mdn (record identity)
│   ├── msgType (error category)
│   ├── msgAction (CREATE/UPDATE/DELETE)
│   ├── msgData (error details)
│   ├── recycledNbrTimes (retry count)
│   ├── auditId (batch context)
│   ├── reconFixId (fix tracking)
│   └── RECON_SYSTEM, PROCESS_TYPE, UBSR_TABLE, RBM_TABLES
│
ReconGeneralReed (DAO)
├── Insert general error into REED_GENERAL_ERRORS:
│   ├── reedErrorId
│   ├── currExcpCd (current exception code)
│   ├── errMsg (error message text)
│   ├── wfName (workflow name)
│   └── recovery (recovery action/flag)
```

### 3.2 Error Data Update

`ReedErrorDataUpd` carries update payloads for modifying error records (e.g., marking them as retried or resolved).

---

## 4. Email Notification Patterns

### 4.1 Email Infrastructure

```
EmailUtils (com.verizon.recon.util)
├── SMTP Host: smtp.verizon.com
├── From: ReconAlert@Verizon.com (alerts) / ReconControls@Verizon.com (control)
├── To: ReconEmailAlertDistroNew@one.verizon.com
├── Jakarta Mail 2.0.1
└── Text-to-SMS: vzwpix.com gateway for mobile alerts
```

### 4.2 Alert Categories

| Category | Trigger | Severity | Recipients |
|----------|---------|----------|-----------|
| **Threshold Breach** | Delete/insert count exceeds threshold | CRITICAL | Email + SMS |
| **JIS Timeout** | >5 consecutive JIS timeouts | HIGH | Email + SMS |
| **Empty Extract** | Zero records in extract file | MEDIUM | Email |
| **Processing Error** | Unhandled exception during fix phase | HIGH | Email |
| **Disk Space Low** | Available disk below minimum | HIGH | Email (throttled) |
| **Run Completion** | Batch run finished (success or failure) | INFO | Email |
| **File Validation** | Missing or corrupt input files | MEDIUM | Email |
| **Gate Status** | Processing gate check results | INFO | Email |
| **Hold Created** | Automatic hold triggered | CRITICAL | Email + SMS |

### 4.3 Splitter Email

`EmailUtils` in the splitter package (`com.verizon.recon.splitter.core`) provides separate email notification for splitter operations with its own templates.

### 4.4 Email Throttling

Disk space alerts include throttling via `minimumDiskSpaceMailTime` to prevent alert flooding during prolonged low-disk conditions.

---

## 5. Empty Extract Handling

The system defines which extract files are **allowed** to be empty per batch mode, preventing false alerts on legitimately empty data sets.

### 5.1 UBSRTORBM Allowed Empty Files

```
EVSRCUSGSEG, CPARD, CPARDENDDT, CPARDDEL, FIXCARRYOVER, PRORATE,
DRAGONSPLAN, RERATE, SPLGPPD, SPLGPCPARD, SFOPOMC, PPLANPD,
DEFAULTSMSPRODUCT, CPARDINVALID, SUSPRODUCT, SXBRESSPLIDPO
```

### 5.2 UBTOUBSR Allowed Empty Files

```
SXBRESSPOLNNAF, SXBRESSPOLNPD, PROMOPD, SPLGPPDCPARD, CANTENNA,
PPLANPD, SFOSPOAC, SPOAPNAF, SFOPOPD, SPOLNPD, SFONAF
```

### 5.3 Vision 2.0 Allowed Empty Files

```
MIGRATEACCTVISION20_BILLING, SUSRECNEW_BILLING, SUSRECNEWP2_BILLING,
SUBATTRID9_BILLING, PRODATTRID1_BILLING, PRODATTRID4_BILLING,
PRODATTRID1NEW_BILLING
```

### 5.4 Allowed Empty Billing Cycles

```
Cycles: 31, 30, 29, 24, 27, 11, 17, 05, 14
```

These are cycle numbers that naturally produce zero records in some zones (e.g., month-end cycles that don't apply to all customers).

### 5.5 checkEmptyExtract Flag

| Property | Value | Behavior |
|----------|-------|----------|
| `checkEmptyExtract=N` | Disabled (default in dev) | Empty extracts are ignored |
| `checkEmptyExtract=Y` | Enabled (production) | Empty extracts trigger alert unless in allowed list |

---

## 6. Log4j2 Async Logging with Disruptor

### 6.1 Architecture

```
Application Thread
    │
    ├── log.info("message")
    │
    ▼
LMAX Disruptor Ring Buffer (lock-free)
    │
    ├── Background Thread
    │
    ▼
RollingRandomAccessFile Appender
    │
    └── Disk Write (async)
```

### 6.2 Performance Characteristics

- **LMAX Disruptor 3.3.4** provides lock-free inter-thread communication
- Application threads never block on disk I/O
- Ring buffer absorbs burst-mode logging during high-volume processing
- `RollingRandomAccessFile` provides memory-mapped file I/O for maximum write performance

### 6.3 Appender Configuration

| Setting | Value |
|---------|-------|
| **File Type** | RollingRandomAccessFile |
| **Rolling Policy** | TimeBasedTriggeringPolicy (daily) |
| **Size Trigger** | 100 MB per file |
| **Max Files** | 50 (rollover count) |
| **Immediate Flush** | true |
| **Root Level** | DEBUG |
| **Async Mode** | All asyncLoggers at debug level |

### 6.4 Separated Log Streams

The logging architecture separates concerns into dedicated log files:
- **jitrRecon.log** — Core processing (extract, compare, fix)
- **jitrJisCalls.log** — Every JIS API request/response (for debugging integration issues)
- **jitrReconSplitter.log** — Splitter distribution logic
- **jitr_recon_validation.log** — Threshold validation audit trail
- **ELKLogs.log** — Structured logs for centralized ELK aggregation

---

## 7. Resiliency Patterns Summary

| Pattern | Implementation | Benefit |
|---------|---------------|---------|
| **Hold Files** | File-system circuit breaker (`.HOLD` files) | Operations team can immediately stop processing |
| **Feature Flags** | AWS AppConfig remote toggle | Disable features without redeployment |
| **Threshold Guards** | Configurable per-entity, per-mode count limits | Prevents mass data corruption |
| **Delete Sampling** | 1% sample evaluation before mass delete | Catches invalid bulk operations |
| **JIS Timeout Hold** | Auto-hold after 5 consecutive timeouts | Prevents cascading JIS failures |
| **Retry Logic** | Single retry on JIS HTTP failure | Handles transient network issues |
| **Connection Pooling** | HikariCP (10 connections) + HTTP (50 connections) | Resource management |
| **Multi-Appender Logging** | Separate log files per concern | Rapid troubleshooting |
| **Email Throttling** | Time-based disk space alert suppression | Prevents alert storms |
| **Test Mode** | `testModeEnabled=Y` per batch mode | Extract+compare only, no fixes applied |
| **Allowed Empty Lists** | Per-entity empty file whitelist | Prevents false alerts on legitimate empties |
| **Runtime Window** | Cron window validation for splitters | Prevents stale trigger execution |
| **Disk Space Check** | Pre-processing disk space verification | Prevents mid-run out-of-space failures |
| **REED Error System** | Dedicated error database with retry tracking | Error recovery and audit trail |
| **Audit Trail** | Every run recorded in JITR_AUDIT_RECON | Full operational history |
