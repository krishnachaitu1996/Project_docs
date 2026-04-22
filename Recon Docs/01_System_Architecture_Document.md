# JITR Recon — System Architecture Document

## 1. Executive Summary

**JITR Recon** (JITR = JITTR Rating / "Just-In-Time Rating") is a Spring Boot 3.3.5 **billing reconciliation engine** that ensures data consistency between Verizon's **UBSR** (Unified Billing System Records) and **RBM** (Rating Billing Mart) databases. It runs as a scheduled batch system — not a request/response web application — comparing billing extracts, detecting discrepancies, and applying automated fixes via JIS API calls.

| Attribute | Value |
|-----------|-------|
| **Language** | Java 17 |
| **Framework** | Spring Boot 3.3.5, Spring Cloud 2022.0.3 |
| **Build Tool** | Maven |
| **Integration** | Apache Camel 3.21.5 |
| **Database** | Oracle (ojdbc11), HikariCP pooling |
| **Scheduling** | Spring `@Scheduled` with Quartz cron expressions |
| **Config Server** | Spring Cloud Config |
| **Feature Flags** | AWS AppConfig (via `feature-flags-java` SDK) |
| **Artifact** | `jitr-recon.jar` (Spring Boot executable) |
| **Repository** | `gitlab.verizon.com/RBMV_JITRRATINGBILLING/jitr_recon` |

---

## 2. Dual Package Structure

The codebase contains two parallel package hierarchies — a sign of an ongoing platform migration from a legacy architecture to **Vision 2.0**.

```
com.verizon.recon.*              ← LEGACY ("Vision 1.0")
com.verizon.jitr.recon.*         ← MODERN ("Vision 2.0" + shared core)
```

### 2.1 Legacy Packages (`com.verizon.recon`)

| Package | Purpose |
|---------|---------|
| `audit/` | Audit logging (insert to `MZADMIN.JITR_AUDIT_RECON`) |
| `bo/` | 40+ Business Objects: DTOs, domain models, cache holders |
| `constants/` | System-wide constants, SQL queries, query constants |
| `context/` | Spring config: `DatabaseContext`, `BatchConfig`, `PasswordEncryptor` |
| `dao/` | Data access layer: `DataAccessTemplate`, RowMapper implementations |
| `exception/` | Custom exceptions: `DataValidationException`, `ReconError`, etc. |
| `odr/` | On-Demand Rating subsystem (ODRProcessor, encoders for 20+ table types) |
| `processor/` | **51 fix processor classes** — the core reconciliation engine |
| `prorate/` | Proration processing (17 classes) |
| `reader/` | File parsers: `BillFixReader`, `RatingAggSort`, `BlFixFileParser` |
| `splitter/` | Large-file partitioning across zones/JVMs |
| `util/` | Utilities: email, thresholds, stats, compression |
| `validator/` | Input file validation: `FileValidator`, `VisionFileFormatter` |

### 2.2 Modern Packages (`com.verizon.jitr.recon`)

| Package | Purpose |
|---------|---------|
| `core/` | 24 classes: `StartUp`, `ReconScheduler`, `ReconCache`, Camel routes, processors |
| `config/` | `FeatureFlagConfig` (AWS AppConfig integration) |
| `common/` | Shared: `ReconThreadPoolManager`, `ELKLogger`, `ReconAppProperties`, `Path` |
| `web/` | REST controllers: `StatusController`, `ReconHealthCheckController` |
| `loader/` | `ReconAppContextLoader` — legacy Spring XML context bridge |
| `helper/` | `SubAccountHelper` |
| `hold/` | `HoldEvaluationService` — gating logic for batch execution |
| `jis/` | JIS integration: HTTP pooling, XML builders, parallel call utilities |
| `dvs/` | 50+ JAXB-generated DVS (Digital Video Services) model classes + encoders |
| `billingVision2/` | **140+ classes** — complete parallel implementation for Vision 2.0 billing |

### 2.3 billingVision2 Internal Structure

The `billingVision2` package mirrors the legacy processing chain with enhanced support:

| Sub-Package | Purpose |
|-------------|---------|
| `audit/` | `AuditVision2` — V2-specific audit trail |
| `billing/` | `Vision2dot0Process`, `BillingFileFormatterVision2` |
| `bo/` | 30+ V2 business objects (CustomerRecord, SubscriptionDetails, etc.) |
| `common/` | `PathVision2` — V2 directory path constants |
| `constants/` | `ConstantsVision2`, `Vision2QueryConstants`, `Vision2RbmExtractQueries` |
| `core/` | `PrebillProcessorVision2`, `ReconCacheVision2`, `ReconSchedulorVision2` |
| `dao/` | V2 RowMappers for new entity types |
| `jis/` | `JisServiceHandlerVision2`, `ParallelJISUtilVision2` |
| `processor/` | 47 V2 fix processors (`UbsrToRbm{Entity}BillingFix`) |
| `processtype/` | 60+ template-based flow processors |
| `reader/` | `RatingAggSortVision2` |
| `util/` | `UbToRbmVision2Helper`, `SubRcnFixesVision2Helper` |
| `validator/` | `FileValidatorVision2` |

---

## 3. Core Processing Pipeline

The system follows a sequential pipeline for each billing reconciliation run:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        SCHEDULED TRIGGER                                │
│           BatchConfig (@Scheduled cron) → ReconScheduler                │
└───────────────┬─────────────────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  PHASE 1: GATE CHECK                                                    │
│  • Hold file validation (RECON.HOLD, FULLBATCH.HOLD, etc.)             │
│  • Feature flag check (AWS AppConfig)                                   │
│  • Disk space verification                                              │
│  • HoldEvaluationService evaluation                                     │
└───────────────┬─────────────────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  PHASE 2: FILE VALIDATION                                               │
│  FileValidator.getValidFiles()                                          │
│  • Expected file count check (17–26 files per zone)                     │
│  • Duplicate file detection                                             │
│  • Date/format validation                                               │
│  • Home zone vs. non-home zone routing                                  │
└───────────────┬─────────────────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  PHASE 3: EXTRACTION (DB Stored Procedures)                             │
│  PreBillProcess → calls Oracle PL/SQL packages on RBM & UBSR schemas   │
│  • PreBillRbmExtract (60 threads) → RBM data                           │
│  • PreBillUbsrExtract (1 thread) → UBSR data                           │
│  • PreBillUBToUBExtract → UBSR-to-UBSR data                            │
│  • VisionFileFormatter → formats Vision extracts                        │
│  • Parallel preloading of reference data (tariffs, products, SFOs)      │
└───────────────┬─────────────────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  PHASE 4: COMPARISON                                                    │
│  ReconFileCompareUtil / Compare processors                              │
│  • Compares UBSR extracts vs. RBM extracts per billing entity           │
│  • Produces diff files classifying records as:                          │
│    - UBSR_ONLY (exists in UBSR, missing in RBM → needs INSERT)         │
│    - NON_UBSR_ONLY (exists in RBM, missing in UBSR → needs DELETE)     │
│    - NOT_MATCHING (exists in both, data differs → needs UPDATE)         │
│  • Threshold checks gate mass-delete/insert operations                  │
└───────────────┬─────────────────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  PHASE 5: FIX PROCESSING (Rating Route)                                 │
│  BillFixReader → RatingRouter → Fix Processors                          │
│  • BillFixReader sorts diff files by fix precedence                     │
│  • RatingRouter dispatches to 51+ specific fix processors               │
│  • Each processor: parse record → make JIS API call → track in DB       │
│  • Operations: Customer CRUD, Account CRUD, SPLAN, EventSource, etc.    │
│  • Parallel processing via thread pools (6–7 file-level threads)        │
│  • DbHandlerReconFix inserts into SUB_RCN_FIXING tracking table         │
└───────────────┬─────────────────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  PHASE 6: POST-PROCESSING & AUDIT                                       │
│  • Offline operations (rerate, REED, EVSRC, SFO/POMC)                  │
│  • Email alerts (success/failure/threshold breach)                       │
│  • Audit record insertion (MZADMIN.JITR_AUDIT_RECON)                    │
│  • File archival to archive directory                                    │
│  • Status updates and cleanup                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Multi-Datasource Architecture

The application connects to **8+ Oracle datasource instances** via HikariCP:

```
┌───────────────────────────────────────────────────────────────────┐
│                     JITR RECON APPLICATION                        │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ JdbcTemplate  │  │ JdbcTemplate  │  │ JdbcTemplate  │           │
│  │     UBSR      │  │     RBM       │  │    AUDIT      │           │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘           │
│         │                  │                  │                    │
│  ┌──────┬───────┐  ┌──────┬───────┐  ┌──────┬───────┐           │
│  │ JdbcTemplate  │  │ JdbcTemplate  │  │ JdbcTemplate  │           │
│  │   MzAdmin     │  │    REED       │  │  RBM Nodes   │           │
│  └──────┬───────┘  └──────┬───────┘  │  (1-6 shards) │           │
│         │                  │          └──────┬───────┘           │
└─────────┼──────────────────┼─────────────────┼───────────────────┘
          │                  │                 │
          ▼                  ▼                 ▼
┌─────────────┐   ┌─────────────┐   ┌─────────────────────────┐
│  UBSR DB    │   │  REED DB    │   │  RBM DB (Oracle RAC)    │
│  (primary   │   │  (error     │   │  Node1 → Node6          │
│   billing   │   │   recovery  │   │  (sharded for           │
│   records)  │   │   & error   │   │   parallel extract)     │
│             │   │   data)     │   │                         │
└─────────────┘   └─────────────┘   └─────────────────────────┘
          │
          ▼
┌─────────────┐   ┌─────────────┐
│  MzAdmin DB │   │  AUDIT DB   │
│  (admin     │   │  (audit     │
│   config &  │   │   trail)    │
│   ODR)      │   │             │
└─────────────┘   └─────────────┘
```

| Datasource | Schema(s) | Purpose |
|------------|-----------|---------|
| **UBSR** | UBSR | Primary billing system records — source of truth |
| **RBM** | RBMUBSR, GENEVAADMIN, RB_CUSTOM | Rating Billing Mart — target system that must match UBSR |
| **RBM Nodes 1–6** | Same as RBM | Sharded connections for parallel extraction (60 threads) |
| **MzAdmin** | MZADMIN | Admin configuration, ODR request tracking, routing metadata |
| **AUDIT** | MZAUD | Audit trail for all reconciliation operations |
| **REED** | REEDSCHEMA | Error recovery & error data (REED = Reconciliation Error Event Data) |

The `DataAccessTemplate` class dynamically routes queries to the correct `JdbcTemplate` by inspecting the SQL schema prefix (e.g., `UBSR.`, `MZADMIN.`, `GENEVAADMIN.`).

---

## 5. Design Patterns

| Pattern | Where Used | How |
|---------|-----------|-----|
| **Router/Dispatcher** | `RatingRouter` | Routes diff files to the appropriate fix processor based on file type |
| **Strategy** | `RatingRouterInterface` | Each fix processor implements a common interface with pluggable logic |
| **Template Method** | `BillingVisionFlowTemplate` (V2) | Abstract template for extract→compare→fix phases |
| **Singleton** | `ReconAppContextLoader` | Loads legacy Spring XML context once |
| **Factory** | `PooledHttpClientFactory` | Creates pooled HTTP client instances for JIS calls |
| **Observer/Callback** | Camel Routes | `PrebillRoute`, `PrebillDeviceRoute` — Camel routes trigger processors |
| **Fan-Out / Scatter-Gather** | `ReconSplitter` | Distributes files across JVM instances using modulus hashing |
| **Hold/Gate** | File-based holds | `.HOLD` files prevent batch execution (manual circuit breaker) |
| **Thread Pool** | `ReconThreadPoolManager` | Camel-based thread pool builder with configurable core/max sizes |
| **Cache Preload** | `ReconCacheData`, `ReconCacheVision2` | In-memory HashMaps preloaded before processing |
| **Dynamic Proxy** | `DataAccessTemplate` | Routes queries to correct datasource by schema detection |

---

## 6. Apache Camel Integration

Apache Camel 3.21.5 provides the ETL routing backbone:

- **`PrebillRoute`** and **`PrebillDeviceRoute`** define Camel `RouteBuilder` pipelines
- Routes are dynamically added to the `CamelContext` during `ReconScheduler.runRecon()`
- The `CamelContext` manages the lifecycle (start → process → wait → stop)
- `ReconThreadPoolManager` uses Camel's `ThreadPoolBuilder` to create `ExecutorService` instances
- Camel `Exchange` objects carry data through the processing pipeline
- CSV parsing via `camel-csv`, metrics via `camel-metrics`, encoding via `camel-base64`

**Camel is used for internal pipeline orchestration, not for external messaging (no Kafka, JMS, or RabbitMQ).**

---

## 7. Thread Pool & Parallel Processing Strategy

```
┌────────────────────────────────────────────────────────────────────┐
│                    PARALLELISM MODEL                               │
│                                                                    │
│  LEVEL 1: Zone Parallelism                                         │
│  ├─ 4–5 zone threads (zone.threads property)                      │
│  ├─ Each zone processes independently                              │
│  └─ Controlled by zone.parallel.process=Y/N                       │
│                                                                    │
│  LEVEL 2: Extract Parallelism                                      │
│  ├─ RBM Extract: 60 threads (for 6 sharded nodes)                 │
│  ├─ UBSR Extract: 1 thread (single source)                        │
│  ├─ Compare: 30 threads (rating) / 1 thread (billing)             │
│  └─ Preload: 3–8 threads (reference data)                         │
│                                                                    │
│  LEVEL 3: Fix-Level Parallelism                                    │
│  ├─ File-level: 6–7 threads (process multiple fix files)           │
│  ├─ JIS calls: 20 threads (parallel API calls)                     │
│  ├─ CPARD fixes: 8 threads (high-volume entity)                   │
│  └─ Prorate: 10 threads                                           │
│                                                                    │
│  LEVEL 4: Splitter Distribution                                    │
│  ├─ Customer modulus hash → JVM assignment                         │
│  ├─ Large customers → dedicated cluster                            │
│  └─ B2B customers → separate processing queue                     │
└────────────────────────────────────────────────────────────────────┘
```

---

## 8. Component Interaction Diagram

```
                    ┌──────────────────────┐
                    │    Spring Boot App    │
                    │      (StartUp)        │
                    │   @EnableScheduling   │
                    └──────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
     ┌────────────┐  ┌────────────────┐  ┌──────────────┐
     │ BatchConfig │  │ DatabaseContext │  │FeatureFlagCfg│
     │ 13 @Schedul │  │ 8+ DataSources │  │ AWS AppConfig│
     └──────┬─────┘  └────────────────┘  └──────────────┘
            │
   ┌────────┼────────────┬──────────────────┐
   │        │            │                  │
   ▼        ▼            ▼                  ▼
┌──────┐ ┌──────┐ ┌───────────┐  ┌────────────────┐
│Prebill│ │ Mini │ │Full/Delta │  │ ODR/Billing    │
│Recon  │ │Recon │ │  Recon    │  │ ODR Recon      │
└──┬───┘ └──┬───┘ └────┬──────┘  └──────┬─────────┘
   │        │           │                │
   └────────┴───────────┴────────────────┘
                    │
                    ▼
          ┌─────────────────┐
          │  ReconScheduler  │  ← Master orchestrator
          │   runRecon()     │
          └────────┬────────┘
                   │
    ┌──────────────┼──────────────────────────────┐
    │              │              │                │
    ▼              ▼              ▼                ▼
┌────────┐  ┌───────────┐  ┌──────────┐  ┌────────────┐
│FileVali│  │ PreBill   │  │BillFix   │  │ Rating     │
│ dator  │  │ Process   │  │ Reader   │  │ Router     │
└────────┘  └─────┬─────┘  └──────────┘  └──────┬─────┘
                  │                               │
           ┌──────┴────────┐              ┌───────┴──────────┐
           │               │              │                  │
           ▼               ▼              ▼                  ▼
    ┌────────────┐  ┌──────────┐  ┌────────────┐  ┌──────────────┐
    │ RBM Extract│  │UBSR Extr.│  │ 51+ Fix    │  │DbHandler     │
    │ (60 thrds) │  │ (1 thrd) │  │ Processors │  │ReconFix      │
    └────────────┘  └──────────┘  └──────┬─────┘  └──────────────┘
                                         │
                                    ┌────┴─────┐
                                    │          │
                                    ▼          ▼
                              ┌──────────┐ ┌──────────┐
                              │ JIS API  │ │ Database │
                              │ Calls    │ │  (CRUD)  │
                              └──────────┘ └──────────┘
```

---

## 9. Technology Stack Summary

| Layer | Technology |
|-------|-----------|
| **Runtime** | Java 17, Spring Boot 3.3.5 |
| **Web Container** | Embedded Tomcat (Undertow available as fallback) |
| **REST API** | Spring Web MVC (2 controllers for health/status only) |
| **Scheduling** | Spring `@Scheduled` + Quartz 2.3.2 cron |
| **Integration/ETL** | Apache Camel 3.21.5 (bean, CSV, metrics, base64 starters) |
| **Database** | Oracle (ojdbc11) via Spring JDBC (JdbcTemplate) |
| **Connection Pool** | HikariCP |
| **Configuration** | Spring Cloud Config Server + local properties |
| **Feature Flags** | AWS AppConfig (`feature-flags-java` 2.0-SNAPSHOT) |
| **Encryption** | Jasypt 3.0.5 + BouncyCastle 1.78.1 |
| **HTTP Client** | Apache HttpClient 4.5.14 (pooled) |
| **XML Processing** | JAXB (Jakarta), JDOM2, Jaxen, Woodstox |
| **Logging** | Log4j2 2.17.2 with LMAX Disruptor (async) |
| **Email** | Jakarta Mail 2.0.1 |
| **SSH/SCP** | JSch 0.1.55 |
| **JSON** | Jackson (core, databind, XML, JSR-310), org.json, json-simple |
| **Utilities** | Apache Commons (Lang3, Net, IO, Codec, Configuration, DBCP), Guava 33 |
| **API Docs** | Springfox Swagger 3.0.0 |
| **Build** | Maven with git-commit-id-plugin, buildnumber-plugin |
| **Testing** | JUnit |
