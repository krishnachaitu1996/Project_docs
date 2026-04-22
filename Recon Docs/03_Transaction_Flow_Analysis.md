# JITR Recon — Transaction Flow Analysis

## 1. Entry Points

There are three entry points into the reconciliation engine:

| Entry Point | Type | Trigger |
|-------------|------|---------|
| **`StartUp.java`** | Application bootstrap | JVM start |
| **`BatchConfig.java`** | Cron scheduler | `@Scheduled` cron expressions every 2 min |
| **`ReconScheduler.java`** | Master orchestrator | Called by BatchConfig → `runRecon()` |

Additionally, two REST endpoints exist for operational monitoring:
- `/jitr-recon/versioncheck` (`ReconHealthCheckController`) — health check returning build info, JVM uptime
- `/jitr-recon/status/*` (`StatusController`) — status reporting and configuration endpoints

---

## 2. End-to-End Flow: A Full Reconciliation Run

Below is the complete lifecycle of a single reconciliation batch run (e.g., a PREBILL run for zone 01, bill cycle 15).

### Phase 0: Application Startup

```
JVM Start
  │
  ▼
StartUp.main()
  ├── SpringApplication.run(StartUp.class)
  ├── @EnableScheduling activates cron system
  ├── @EnableEncryptableProperties loads Jasypt decryptor
  ├── @ComponentScan("com.verizon") discovers all beans
  ├── DatabaseContext creates 8+ HikariCP DataSources
  ├── FeatureFlagConfig creates FeatureFlagManager (AWS AppConfig)
  └── BatchConfig registers 13+ @Scheduled methods
```

### Phase 1: Cron Trigger & Feature Flag Gate

```
Cron fires: 0 0/2 * * * ? (every 2 minutes)
  │
  ▼
BatchConfig.runPrebill()
  ├── Check: feature.flag.enabled == Y ?
  │    ├── YES → featureFlagManager.getFlag("jitr-recon", profile, flag)
  │    │         ├── Flag = true  → PROCEED
  │    │         └── Flag = false → SKIP (log & return)
  │    └── NO  → PROCEED unconditionally
  │
  ▼
preBillRecon.runRecon()  (calls ReconScheduler)
```

### Phase 2: Hold File Validation

```
ReconScheduler.runRecon()
  │
  ├── Check hold files at C:/RECON/HOLD/
  │    ├── RECON.HOLD exists?       → STOP (global hold)
  │    ├── PREBILL.HOLD exists?     → STOP (mode hold)
  │    └── No hold files            → CONTINUE
  │
  ├── Check disk space on C:/RECON/
  │    └── Below threshold → Email alert, STOP
  │
  └── Initialize audit ID (from JITR_AUDIT_RECON_SEQ)
```

### Phase 3: File Discovery & Validation

```
ReconScheduler.runRecon() continued
  │
  ▼
FileValidator.getValidFiles()
  ├── Scan vision directory: C:/RECON/VISION/{zone}_{billcycle}/
  │
  ├── Count files per zone:
  │    ├── Home Consumer zone: expect 25 files
  │    ├── Home B2B zone: expect 26 files
  │    ├── Non-home zone: expect 17 files
  │    └── Minimum threshold: 17 files
  │
  ├── Validate each file:
  │    ├── Name format check (matches expected pattern)
  │    ├── Date validation (not future-dated unless flag allows)
  │    ├── Duplicate detection (content comparison if enabled)
  │    └── Encoding validation
  │
  ├── Classify files by entity type:
  │    ├── RCN_CUSTOMER
  │    ├── RCN_BL_ACCT (with variants: CUST_MTN, LN_STAT, SPLAN, SVC_PROD)
  │    ├── RCN_SF_MTN, RCN_LN_SVC_PROD
  │    ├── RCN_ECPD_PROF_CUST, RCN_PPLAN_MTN
  │    └── 20+ additional entity files
  │
  └── Return: Map<zone_billcycle, List<validFiles>>
```

### Phase 4: Vision 2.0 Billing File Handling

```
If vision2.0.enabled == Y:
  │
  ├── handle_RBFiles()
  │    └── Replace billing files with rating files for V2 processing
  │
  ├── CheckAtrributeConfig()
  │    └── Validate Vision 2.0 attribute configuration
  │
  └── Initialize ReconCacheVision2
       └── Preload V2 reference data into memory
```

### Phase 5: Extraction (Database Stored Procedures)

```
PreBillProcess.process(custIdList, context)
  │
  ├── Thread Pool: prebill.preloadthreads (3-8 threads)
  │    ├── Task 1: PreLoadXltRbmProducts()
  │    │    └── Load reference product data from RBM into cache
  │    ├── Task 2: PreLoadDragonProdsInRbm()
  │    │    └── Load Dragon special product mappings
  │    └── Task 3: PreLoadInsertPreBillProcessingList()
  │         └── Get domain IDs → Insert into PLIST table
  │
  ├── PreBillRbmExtract (60 threads across 6 RBM nodes)
  │    ├── Calls Oracle PL/SQL packages:
  │    │    ├── RCN_UBTORBM_RBM_EXT_FULL    (full batch)
  │    │    ├── RCN_UBTORBM_RBM_EXT_MINI    (mini batch)
  │    │    ├── RCN_UBTORBM_RBM_EXT_MINIODR (mini ODR)
  │    │    └── RCN_EXTRACT_DROP_CARRYOVER   (cleanup)
  │    └── Writes extracted data to local files
  │
  ├── PreBillUbsrExtract (1 thread)
  │    ├── Queries UBSR database for matching records
  │    └── Writes to local files
  │
  └── PreBillUBToUBExtract
       ├── Calls UBSR-to-UBSR extraction packages:
       │    ├── RCN_UBTOUB_VISION2_PRBL_EXT  (Vision 2.0 prebill)
       │    ├── RCN_UBTOUB_VISION2_FULL_EXT  (Vision 2.0 full)
       │    └── RCN_UBTOUB_VISION2_MINI_EXT  (Vision 2.0 mini)
       └── Writes to local files
```

### Phase 6: Comparison Phase

```
VisionToUbsrCompareProcessor / UbsrToUbsrCompareProcessor
  │
  ├── Compare threads: 30 (rating) / 1 (billing)
  │
  ├── For each entity type (Customer, Account, SPLAN, EventSource, ...):
  │    │
  │    ├── Load UBSR extract file
  │    ├── Load RBM extract file
  │    │
  │    ├── Apply key-rules.properties comparison steps:
  │    │    Step 0: Key field match (ID-level)
  │    │    Step 1-3: Core attribute comparison
  │    │    Step 4-7: Extended attribute comparison
  │    │    Step 8-11: Full field comparison
  │    │
  │    ├── Classify each record:
  │    │    ├── UBSR_ONLY      → exists in UBSR, missing in RBM → needs INSERT
  │    │    ├── NON_UBSR_ONLY  → exists in RBM, missing in UBSR → needs DELETE
  │    │    └── NOT_MATCHING   → both exist, fields differ → needs UPDATE
  │    │
  │    └── Write diff files to compare output directory
  │
  ├── Threshold checking:
  │    ├── Delete threshold (e.g., CUSTOMER:100, LNSVCPROD:15000)
  │    │    └── If delete count > threshold → HOLD (email alert, stop processing)
  │    ├── Insert threshold (e.g., percent.value.insert.threshold = 10%)
  │    │    └── If insert rate > threshold → HOLD
  │    └── Threshold values differ by batch mode and entity type
  │
  └── HoldEvaluationService.evaluate()
       ├── Parse diff files for hold-eligible operations
       ├── Sample deleteSamplePercentage (1%) of deletions
       └── If hold triggered → write hold file, send alert, STOP
```

### Phase 7: Fix Processing (Rating Route)

```
BillFixReader.collectFiles()
  │
  ├── Scan compare output directory
  ├── Match files against BILL_FIX_FILES_REGEX
  ├── Sort by fix precedence (static ordering map)
  │    Priority: Customer → Account → SPLAN → EventSource → Products → ...
  │
  └── Return: ordered list of diff files
```

```
RatingRouter.preRequisite()
  │
  ├── ratingAggSort.processFilterSort()   ← aggregate and sort fix files
  ├── initializeRouteMap()                 ← inject fix processors
  ├── Load RBMProcessingServiceList        ← target system configuration
  │
  ▼
FOR EACH diff file (in precedence order):
  │
  ├── Identify fix processor based on file type:
  │    ├── RCN_CUSTOMER        → UbToRbmCustomerFix
  │    ├── RCN_BL_ACCT         → UbToRbmAccountFix
  │    ├── RCN_BL_ACCT_SPLAN   → UbToRbmSplanFix
  │    ├── RCN_EVSRC           → UbToRbmEventSourceFix
  │    ├── RCN_SFO_PD          → UbToRbmSfoPDFix
  │    ├── RCN_ALL_PD          → UbToRbmAllPDFix
  │    ├── RCN_CPARD           → UbToRbmCpardDelFix / CpardInvalidFix
  │    ├── RCN_PPLAN_PD        → UbToRbmPplanPDFix
  │    ├── RCN_DRAGON_SPLAN    → UbToRbmDragonSplanFix
  │    ├── RCN_EVSRC_USG_SEG   → UbToRbmEvtSrcUsgSeg
  │    ├── RCN_SUS_PRODUCT     → UbtoRbmSusproductFix
  │    ├── RCN_DEFAULT_SMS     → UbToRbmDefaultSMSProductFix
  │    ├── UBTOUB files        → UbToUbFix / UbtoUbCantennaFix / UbtoUbCorrections
  │    └── 40+ more specialized fix processors
  │
  ├── Thread pool: 6-7 file-level threads (rating.fix.file.level.threads)
  │
  ▼
INSIDE EACH FIX PROCESSOR (implements RatingRouterInterface):
  │
  ├── Read diff file line by line
  ├── Parse into SubXltBlRbmRecord
  │    (fields: custId, acctId, mdn, action, entityType, fieldValues...)
  │
  ├── Determine operation:
  │    ├── UBSR_ONLY record → CREATE in RBM (JIS API call)
  │    ├── NON_UBSR_ONLY    → DELETE from RBM (JIS API call)
  │    └── NOT_MATCHING     → UPDATE in RBM (JIS API call)
  │
  ├── Execute JIS API Call:
  │    ├── JisServiceHandler builds XML request from SubXltBlRbmRecord
  │    ├── MakeJisCall sends HTTP POST to JIS endpoint
  │    │    ├── Host: jis.host (e.g., tpaldrbmva012.ebiz.verizon.com)
  │    │    ├── Port: jis.port (e.g., 14305)
  │    │    ├── Auth: Base64(jis.username:jis.psd)
  │    │    ├── 20 parallel JIS threads
  │    │    └── PooledHttpClient (50 connections, 1 retry)
  │    └── Parse JIS response (success/failure/timeout)
  │
  ├── Track fix in database:
  │    └── DbHandlerReconFix.buildAndRouteToReconFixes()
  │         ├── Build SubRcnFixData tracking object
  │         ├── INSERT into SUB_RCN_FIXING table
  │         └── Set status: SUCCESS / ERROR / RECENT_LDR_UPDATE
  │
  └── Flush to DB every db.flush.rate (20,000) records
```

### Phase 8: Post-Processing

```
ReconScheduler.startPostProcessingSteps()
  │
  ├── Offline Operations (home zone only):
  │    ├── updateRerateTableOffline()     (if rerateOffline=Y)
  │    ├── updateReedRerateTableOffline() (if reedRerateOffline=Y)
  │    ├── updateEvsrcDeleteOffline()     (if evsrcOffline=Y)
  │    └── runSfopomcOfflineProcess()     (if sfopomcOffline=Y)
  │
  ├── Prorate Processing:
  │    └── ProcessProrate handles SUB_PRORATE_REQUEST table
  │         ├── 10 prorate threads
  │         └── Applies proration rules to partial-cycle charges
  │
  ├── Rerate Processing:
  │    └── RerateProcessing (if rerate.scheduler.on=Y)
  │         ├── Runs at 9 AM & 5 PM
  │         └── Submits rerate requests for GAPI corrections
  │
  ├── Audit Logging:
  │    └── Audit.doAudit()
  │         ├── Batch insert to MZADMIN.JITR_AUDIT_RECON
  │         └── Records: auditId, mode, zone, billcycle, counts, timestamps, status
  │
  ├── File Management:
  │    ├── moveZoneFilesDeleteCompletedFile()
  │    ├── moveHoldPathFilesArchiveFolder()
  │    └── Archive to C:/RECON/ARCHIVE/{DDMMYYYY}/{zone}_{cycleid}
  │
  ├── Email Notification:
  │    └── EmailUtils sends summary to ReconAlert@Verizon.com
  │         ├── Success: processing counts, duration
  │         ├── Failure: error details, threshold breaches
  │         └── Text alerts to mobile numbers
  │
  └── Status Update:
       └── ReconStatus records completion in status table
```

---

## 3. Data Flow Across the Four Oracle Datasources

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         EXTRACTION PHASE                                    │
│                                                                             │
│  UBSR DB ────extract──┐                                                     │
│  (source   billing    │    ┌──────────┐                                     │
│   of truth) records   ├───▶│  LOCAL    │                                    │
│                       │    │  FILES    │                                    │
│  RBM DB  ────extract──┘    │ (extract) │                                    │
│  (Nodes 1-6)               └────┬─────┘                                    │
│  60 parallel threads             │                                          │
│                                  ▼                                          │
│                         ┌──────────────┐                                    │
│  MzAdmin DB ──config──▶ │   COMPARE    │ ◀── key-rules.properties          │
│  (MTDT_JITR_ROUTING,   │   ENGINE     │                                    │
│   REF_RECON_ONDEMAND)  └──────┬───────┘                                    │
│                                │                                            │
│                                ▼                                            │
│                         ┌──────────────┐                                    │
│                         │  DIFF FILES  │                                    │
│                         │ UBSR_ONLY    │                                    │
│                         │ NON_UBSR_ONLY│                                    │
│                         │ NOT_MATCHING │                                    │
│                         └──────┬───────┘                                    │
│                                │                                            │
│                         ┌──────────────┐                                    │
│                         │  FIX ENGINE  │                                    │
│                         │ (51+ procs)  │                                    │
│                         └──────┬───────┘                                    │
│                                │                                            │
│                    ┌───────────┼───────────┐                                │
│                    │           │           │                                │
│                    ▼           ▼           ▼                                │
│              ┌──────────┐ ┌──────────┐ ┌──────────┐                        │
│  RBM DB ◀───│ JIS API  │ │ UBSR DB  │ │ REED DB  │                        │
│  (updates   │ (CUD ops)│ │(tracking)│ │(error    │                        │
│   via JIS)  └──────────┘ └──────────┘ │ records) │                        │
│                                        └──────────┘                        │
│                                                                             │
│  AUDIT DB ◀── doAudit() (JITR_AUDIT_RECON table)                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Datasource Usage by Phase

| Phase | UBSR | RBM (1-6) | MzAdmin | AUDIT | REED |
|-------|------|-----------|---------|-------|------|
| Extraction | READ (records) | READ (records via stored procs) | READ (config/routing) | — | — |
| Comparison | — | — | — | — | — |
| Fix Processing | READ/WRITE (tracking) | WRITE (via JIS) | READ (config) | — | WRITE (errors) |
| Audit | — | — | WRITE (audit table) | WRITE (doAudit) | — |
| Post-Processing | WRITE (offline updates) | — | WRITE (status) | — | WRITE (rerate errors) |

---

## 4. JIS API Call Flow Detail

```
Fix Processor (e.g., UbToRbmCustomerFix)
  │
  ├── Build JisRequestData (50+ fields from SubXltBlRbmRecord)
  │
  ▼
JisServiceHandler.handleJisRequest()
  ├── Determine operation type: CREATE / UPDATE / DELETE / TERMINATE
  ├── Set effective timestamps
  ├── Handle Voice & Text impact fields
  ├── Handle GAPI corrections
  │
  ▼
JisCreateXmlHelper.buildXml()
  ├── Build SOAP/XML request body
  │
  ▼
MakeJisCall.execute()
  ├── Endpoint: http://{jis.host}:{jis.port}/{operation}
  ├── Auth: Basic (Base64 encoded)
  ├── Connection: PooledHttpClient (50 pool, 100M ms timeout)
  ├── Retry: 1 attempt on failure
  │
  ├── Response handling:
  │    ├── HTTP 200 → parse XML response → SUCCESS
  │    ├── HTTP 4xx → log error → mark ERROR
  │    ├── HTTP 5xx → retry once → if still fails → mark ERROR
  │    └── Timeout → increment timeout counter
  │         └── If timeouts > jis.timeout.hold.threshold.limit (5)
  │              → trigger hold alert
  │
  └── Return: JisResponseData (responseCode, message, transactionId)
```

---

## 5. Vision 2.0 Processing Flow (Parallel Path)

When `vision2.0.enabled=Y`, a parallel processing chain runs alongside or instead of legacy processing:

```
Vision 2.0 Trigger
  │
  ▼
ReconSchedulorVision2
  ├── Uses BillingVisionFlowTemplate (abstract template pattern):
  │    ├── ubToRbmExtract(Exchange)   [parallel]
  │    ├── ubToRbmCompare(Exchange)   [parallel]
  │    └── ubToRbmFix(Exchange)       [distributed]
  │
  ├── 60+ V2 processtype processors (template-based):
  │    ├── UbsrToRbmAccountAddressBillingProcessor
  │    ├── UbsrToRbmProductsBillingProcessor
  │    ├── UbsrToRbmSubscriptionBillingProcessor
  │    └── ... (60+ more)
  │
  ├── 47 V2 fix processors (UbsrToRbm{Entity}BillingFix)
  │
  ├── ParallelJISUtilVision2 for batched JIS calls
  │
  └── ReconCacheVision2 for V2 reference data caching
```

---

## 6. ODR (On-Demand Rating) Flow

ODR follows a simpler, single-customer flow triggered by entries in the `MZADMIN.REF_RECON_ONDEMAND` table:

```
BatchConfig.runodrRecon()  [every 10 min, 10-11 PM]
  │
  ▼
ODRProcessor.process()
  ├── Query REF_RECON_ONDEMAND for pending requests
  │    (status = READY)
  │
  ├── For each request:
  │    ├── Extract customer data from UBSR & RBM
  │    ├── Use DVS encoders (40+ types) to encode Vision data
  │    ├── Compare UBSR vs. RBM for this customer
  │    ├── Apply fixes via JIS API
  │    ├── Handle ODR-specific prorate rules
  │    └── Update status: READY → RUNNING → COMPLETE
  │
  ├── Special handling:
  │    ├── Large customer skip list (OdrLargeCustData)
  │    ├── DVS retry on failure (odr.dvs.failed.retry.count = 3)
  │    └── Rerate delay until odr.delay.rerate.starttime (7 AM)
  │
  └── ODR Billing (separate schedule):
       └── Processes billing-side of ODR requests
```
