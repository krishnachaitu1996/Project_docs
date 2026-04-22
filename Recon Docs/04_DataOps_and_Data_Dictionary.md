# JITR Recon — DataOps & Data Dictionary

## 1. Business Objects (`com.verizon.recon.bo`)

The `bo/` package contains 40+ data transfer objects (DTOs) and domain models that carry data through the processing pipeline. They are plain Java objects (no JPA annotations) — all database I/O is through Spring `JdbcTemplate` with custom `RowMapper` implementations.

### 1.1 Core Domain Objects

| Class | Purpose | Key Fields |
|-------|---------|------------|
| **`SubXltBlRbmRecord`** | **Central data record** flowing through the fix pipeline. Represents a single UBSR↔RBM translation record from the `SUB_XLT_BL_RBM` table. | custId, acctId, mdn, entityType, action (CREATE/UPDATE/DELETE), field values map, billCycleNo, zoneId, auditId |
| **`CustDomain`** | Customer domain mapping — links customer IDs to domain groups | custIdNo, domainGroupId, domainId |
| **`CustDomainGrpId`** | Domain group identifier with zone mapping | domainGroupId, domainId, blCycNo |
| **`CustAcctMdnGroupInt`** | Grouped interface for customer→account→MDN hierarchy | custIdNo, acctNo, mdn, groupId |
| **`JisCustAcctMdnInt`** | JIS-specific customer/account/MDN integration record | custId, acctId, mdn, jisAction, result |
| **`VisionUbsrRecord`** | Single record from a Vision extract file (UBSR side) | All fields from Vision fixed-width file |
| **`GlobalVisionUbsr`** | Global UBSR vision record with zone context | Extends VisionUbsrRecord with zone, cycle |

### 1.2 Billing & Cycle Objects

| Class | Purpose | Key Fields |
|-------|---------|------------|
| **`BillCycleConfig`** | Billing cycle configuration — tracks which cycle/zone combos are active | billCycleNo, zoneId, status, auditId, startDate, endDate |
| **`BlIdAndTimezoneData`** | Billing ID with timezone offset | billingId, timezoneOffset, dstFlag |
| **`SubPendingCycleChange`** | Subscriber with pending billing cycle change | custIdNo, currentCycle, pendingCycle, effectiveDate |

### 1.3 Processing Control Objects

| Class | Purpose | Key Fields |
|-------|---------|------------|
| **`ProcessObj`** | Represents a processing unit with status | processName, processType, status (READY/RUNNING/COMPLETE), startTime, endTime |
| **`PreBillProcessingList`** | List of customers/domains queued for pre-bill processing | domainIdMap, custIdList, billCycleNo |
| **`ThreadContextInfo`** | Thread-local context carrying zone, audit, and mode info | auditId, zoneId, billCycleNo, batchMode, threadId |
| **`PropertyModifier`** | Dynamic property override during runtime | key, value, batchType |

### 1.4 JIS Integration Objects

| Class | Purpose | Key Fields |
|-------|---------|------------|
| **`JisRequestData`** | **50+ fields** — complete JIS API request payload | custId, acctId, mdn, operationType, effectiveDate, languageId, formatId, contactTypeId, addressFields, planFields, productFields, pricingFields, all entity-specific attributes |
| **`JisResponseData`** | JIS API response | responseCode, message, transactionId, errorCode |
| **`TerminateObj`** | Account termination request | acctId, terminateDate, reason |

### 1.5 Cache & Reference Objects

| Class | Purpose | Key Fields |
|-------|---------|------------|
| **`ReconCacheData`** | **In-memory cache** preloaded before processing. Contains HashMaps for tariffs, XLT products, zones, rates, rules, and reference data. | tariffMap, xltProductMap, zoneMap, rateMap, rulesMap, refDataMap |
| **`RefVisRatingProdHeir`** | Reference: Vision-to-Rating product hierarchy mapping | visionProdId, ratingProdId, hierarchy, effectiveDate |
| **`GlobalConfigTable`** | Global configuration from MZADMIN schema | configKey, configValue, description |
| **`RbmProcessingService`** | RBM processing service definition (target system metadata) | serviceId, serviceName, hostUrl, status |

### 1.6 Audit & Tracking Objects

| Class | Purpose | Key Fields |
|-------|---------|------------|
| **`AuditTable`** | Audit record for MZADMIN.JITR_AUDIT_RECON | auditId, mode, zone, billCycle, processType, counts, timestamps, status |
| **`JitrAuditRecon`** | Audit record structure | auditId, batchType, zone, cycle, startTime, endTime, totalRecords, fixedRecords |
| **`SubRcnFixData`** | Fix tracking record for SUB_RCN_FIXING table | fixId, auditId, custId, entityType, action, status, jisTransactionId |

### 1.7 Error & Recovery Objects

| Class | Purpose | Key Fields |
|-------|---------|------------|
| **`ReedRecord`** | REED error/recovery record | ehCaseId, statusCd, custIdNo, acctNo, mdn, msgType, msgAction, recycledNbrTimes |
| **`ReedErrorDataUpd`** | REED error data update payload | reedErrorId, currExcpCd, errMsg, recovery |
| **`RerateDataObj`** | Rerate request data | custId, acctId, mdn, rerateType, eventType, status |

### 1.8 Fix & Correction Objects

| Class | Purpose | Key Fields |
|-------|---------|------------|
| **`SortedFixTableData`** | Sorted collection of fix records for batch processing | fixList (sorted by precedence) |
| **`FixDGIDMapping`** | Fix Domain Group ID mapping | srcDomainGroupId, tgtDomainGroupId, fixType |
| **`VisionFormatterObj`** | Formatted Vision extract output record | formattedColumns, entityType, zoneId |

### 1.9 On-Demand & Prorate Objects

| Class | Purpose | Key Fields |
|-------|---------|------------|
| **`OnDemandInt`** | On-demand integration record | custId, acctId, requestType, status, requestDate |
| **`OnDemandProrateInt`** | On-demand prorate request | custId, acctId, mdn, prorateType, startDate, endDate |
| **`RequestOdr`** | ODR request data | requestId, custId, acctId, requestType, status |
| **`TriggerOdr`** | ODR trigger event | triggerId, requestId, triggerType, effectiveDate |
| **`OdrLargeCustData`** | Large customer ODR data (for skip list) | custId, acctId, txnCount, isLarge |

### 1.10 Miscellaneous Objects

| Class | Purpose | Key Fields |
|-------|---------|------------|
| **`B2BCustData`** | B2B customer data (large enterprise accounts) | custId, b2bFlag, customerType |
| **`MouData`** | Minutes-of-usage data | mdn, mouType, usageMinutes, period |
| **`DeviceWriter`** | Device equipment data writer | deviceId, simId, mdn, equipmentType |
| **`UbsrNonUbsrTables`** | Map of UBSR vs non-UBSR table names | ubsrTableName, nonUbsrTableName |
| **`ReconPropertyFile`** | Runtime property file reader | filePath, properties |

---

## 2. DAO Layer

### 2.1 DataAccessTemplate — Central DAO Service

`DataAccessTemplate` (`@Service @Transactional`) is the **single point of entry** for all database operations. It injects **13+ JdbcTemplate instances** covering all datasources and dynamically routes queries.

**Injected Templates:**

| Qualifier | Datasource |
|-----------|-----------|
| `jdbcTemplateUbsr` | UBSR (primary billing) |
| `jdbcTemplateMzAdmin` | MzAdmin (config/admin) |
| `jdbcTemplateAudit` | Audit (audit trail) |
| `jdbcTemplateRbm` | RBM (rating billing mart) |
| `namedParamjdbcTemplateReed` | REED (error recovery) |
| `jdbcTemplateRbmNode1` – `Node6` | RBM sharded nodes |
| `namedParamjdbcTemplateRbm` | RBM (named parameter) |
| `namedParamjdbcTemplateMzAdmin` | MzAdmin (named parameter) |
| `namedParamjdbcTemplateAudit` | Audit (named parameter) |

**Key Methods:**

| Method | Purpose |
|--------|---------|
| `query(sql, mapper)` | SELECT with RowMapper — auto-routes to correct datasource by SQL schema prefix |
| `queryByContext(sql, mapper, context)` | SELECT with thread context for datasource selection |
| `queryForList(sql, args)` | Returns `List<Map<String,Object>>` |
| `batchUpdateTable(sql, params)` | Batch INSERT/UPDATE/DELETE |
| `updateTable(sql, params)` | Single UPDATE/DELETE |
| `insertObj(sql, params)` | Single INSERT |
| `queryRow(sql, mapper, args)` | Single-row SELECT |
| `getJDBCTemplate(sql)` | Dynamic datasource router — inspects SQL for schema name |
| `getNamedParamJDBCTemplate(sql)` | Same for named-parameter templates |

**Dynamic Routing Logic:**
```
getJDBCTemplate(sql):
  if sql contains "UBSR."        → return jdbcTemplateUbsr
  if sql contains "MZADMIN."     → return jdbcTemplateMzAdmin
  if sql contains "GENEVAADMIN." → return jdbcTemplateRbm
  if sql contains "RBMUBSR."     → return jdbcTemplateRbm
  if sql contains "RB_CUSTOM."   → return jdbcTemplateRbm
  if sql contains "REEDSCHEMA."  → return jdbcTemplateReed
  default                        → return jdbcTemplateUbsr
```

### 2.2 Specialized DAO Classes

| DAO Class | Database | Purpose |
|-----------|----------|---------|
| **`ReconReed`** | REED | Error/recovery record management; batch `MapSqlParameterSource` list for bulk inserts |
| **`ReconGeneralReed`** | REED | General error records; exception code, message, workflow tracking |

### 2.3 RowMapper Implementations

| Mapper Class | Maps To | Source Table/Query |
|--------------|---------|--------------------|
| **`AuditTableMapper`** | `AuditTable` | JITR_AUDIT_RECON |
| **`CustDomainMapper`** | `CustDomain` | Customer domain lookup |
| **`CustDomainGroupIdMapper`** | `CustDomainGrpId` | Domain group ID lookup |
| **`ReedTableMapper`** | `ReedRecord` | REED tables |
| **`RefVisRatingProdHeirMapper`** | `RefVisRatingProdHeir` | REF_VIS_RATING_PROD_HEIR |
| **`RbmProcessingServiceMapper`** | `RbmProcessingService` | RBM processing service config |

---

## 3. In-Memory Caching Approach

### 3.1 ReconCacheData (Legacy)

Preloaded at the start of each reconciliation run with reference data needed throughout processing:

```
ReconCacheData (HashMap-based cache)
├── tariffMap         → Tariff codes to pricing rules
├── xltProductMap     → XLT product ID to product definition
├── zoneMap           → Zone IDs to zone metadata
├── rateMap           → Rate codes to rate values
├── rulesMap          → Processing rules per entity type
├── refDataMap        → General reference data
├── dragonProductMap  → Dragon special product mappings
└── policyMap         → Policy SFO configuration
```

**Lifecycle:**
1. `PreBillProcess.process()` → preload tasks populate cache
2. Cache remains in-memory for duration of batch run
3. Each zone/thread accesses cache read-only (thread-safe via `CopyOnWriteArrayList` and `ConcurrentHashMap`)
4. Cache discarded at batch completion

### 3.2 ReconCacheVision2 (Vision 2.0)

Extended cache for Vision 2.0 with additional reference data:

```
ReconCacheVision2
├── All legacy cache data (inherited)
├── subscriptionReferenceMap
├── productAttributeMap
├── v2EntityMappingMap
└── billingConfigCache
```

---

## 4. Oracle Stored Procedures (dbscripts/)

### 4.1 RBM Database Packages

| Package | Invoked By | Purpose |
|---------|-----------|---------|
| **`RCN_UBTORBM_RBM_EXT_FULL`** | FULLBATCH extraction | Full extract of RBM data for all customers in a bill cycle |
| **`RCN_UBTORBM_RBM_EXT_MINI`** | MINIBATCH extraction | Mini extract — only recently changed RBM records |
| **`RCN_UBTORBM_RBM_EXT_MINIODR`** | MINI ODR extraction | Mini extract with ODR-specific filters |
| **`RCN_EXTRACT_DROP_CARRYOVER`** | Post-extraction cleanup | Drops carryover data from previous cycles |
| **`RCN_EXTRACT_DROP_CARRYOVER_RBM2020`** | Post-extraction (2020+) | Updated carryover drop for RBM 2020 schema |
| **`RCN_EXTRACT_FIX_CARRYOVER`** | Fix phase | Carries forward unresolved fixes to next cycle |

### 4.2 UBSR Database Packages

| Package | Invoked By | Purpose |
|---------|-----------|---------|
| **`RCN_UBTOUB_VISION2_PRBL_EXT`** | PREBILL V2 extraction | UBSR-to-UBSR Vision 2.0 prebill extract |
| **`RCN_UBTOUB_VISION2_FULL_EXT`** | FULLBATCH V2 extraction | UBSR-to-UBSR Vision 2.0 full extract |
| **`RCN_UBTOUB_VISION2_MINI_EXT`** | MINIBATCH V2 extraction | UBSR-to-UBSR Vision 2.0 mini extract |
| **`RCN_UBTOUB_VISION2_MINIODR_EXT`** | MINI ODR V2 | Vision 2.0 mini ODR extraction |
| **`RCN_UBTOUB_VISION2_ODR_EXT`** | ODR V2 | Vision 2.0 on-demand rating extraction |

### 4.3 DDL Scripts

**UBSR DDL (12 files):**
- `MTDT_JITR_ROUTING.SQL` — routing metadata table for domain → process mapping
- `UBSR_RECON_ODR_PROCESSING_LIST.SQL` — ODR processing queue table
- `ILB_REQUEST.SQL` — ILB (InterLink Billing) request table
- `DDL_TRIGGER_SUB_XLT_BL_RBM_TRIG.SQL` — trigger on SUB_XLT_BL_RBM for auto-tracking
- `DDL_03092017_SUB_PRORATE_REQUEST.SQL` — prorate request table
- `CREATE_TYPE.SQL` — custom Oracle types
- Grant scripts for MZADMIN, REED, DB2CDC access

**RBM DDL (4 files):**
- `CREATE_TYPE.SQL` — RBM custom types
- Grant scripts for GENEVA_ADMIN schema access

### 4.4 DML Scripts (Data Seeding)

| Script | Purpose |
|--------|---------|
| `DML_PRELOADS_FULL.sql` | Data preload for full batch mode |
| `DML_PRELOADS_MINI.sql` | Data preload for mini batch mode |
| `DML_PRELOADS_DELTAMINI.sql` | Data preload for delta mini mode |
| `DML_PRELOADS_ODR.SQL.sql` | Data preload for ODR mode |
| `DML_PRELOADS_PRBL.SQL` | Data preload for prebill mode |
| `DML_PRELOADS_MINI_WITH_SPLIT.sql` | Mini batch with splitter preloads |
| `DMD_Preload.sql` | DMD-specific preloads |
| Various `B2B` and `Vision` variants | B2B and Vision 2.0 specific seeds |
| `CLEANUP_*.sql` | Audit ID cleanup scripts |

---

## 5. Key Database Tables

| Table | Schema | Purpose |
|-------|--------|---------|
| **`SUB_XLT_BL_RBM`** | UBSR | Main translation table — UBSR↔RBM record mapping |
| **`SUB_RCN_FIXING`** | UBSR | Fix tracking — every fix attempt recorded here |
| **`SUB_PRORATE_REQUEST`** | UBSR | Prorate request queue |
| **`JITR_AUDIT_RECON`** | MZADMIN | Audit trail for each reconciliation run |
| **`REF_RCN_BL_CYCLE_CONFIG`** | MZADMIN | Bill cycle configuration (active cycle/zone combos) |
| **`MTDT_JITR_ROUTING`** | UBSR | Domain → process routing metadata |
| **`REF_RECON_ONDEMAND`** | MZADMIN | ODR request queue |
| **`REF_VIS_RATING_PROD_HEIR`** | RBM | Vision-to-Rating product hierarchy reference |
| **`ILB_REQUEST`** | UBSR | InterLink Billing request records |
| **`REED_GENERAL_ERRORS`** | REEDSCHEMA | Error recovery event data |

---

## 6. Data Transformation Flow

```
VISION EXTRACT FILES (fixed-width text)
    │
    ▼  [VisionFileFormatter]
FORMATTED UBSR RECORDS (VisionUbsrRecord)
    │
    ▼  [PreBillRbmExtract / PreBillUbsrExtract]
RBM EXTRACT RECORDS (from Oracle stored procs)
    │
    ▼  [Compare Processor + key-rules.properties]
DIFF RECORDS classified as:
├── UBSR_ONLY     → SubXltBlRbmRecord(action=INSERT)
├── NON_UBSR_ONLY → SubXltBlRbmRecord(action=DELETE)
└── NOT_MATCHING  → SubXltBlRbmRecord(action=UPDATE)
    │
    ▼  [Fix Processor]
JIS REQUEST (JisRequestData) → JIS API CALL
    │
    ▼  [JIS Response]
FIX TRACKING RECORD (SubRcnFixData) → INSERT into SUB_RCN_FIXING
    │
    ▼  [Audit]
AUDIT RECORD (JitrAuditRecon) → INSERT into JITR_AUDIT_RECON
```
