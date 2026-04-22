# JITR Recon — Environment & Configuration Guide

## 1. Spring Cloud Config Server

### 1.1 Connection Setup

The application uses Spring Cloud Config for externalized configuration management.

**bootstrap.properties (Local development):**
```properties
spring.cloud.config.uri=http://tpaldrbmva016.ebiz.verizon.com:8888
spring.profiles.active=native
spring.cloud.config.fail-fast=true
spring.main.allow-circular-references=true
management.endpoints.web.exposure.include=refresh
```

**bootstrap.properties (Deployed — PROPS/ folder):**
```properties
spring.cloud.config.uri=@RECON.CLOUD.CONFIG.URI@/config-server
```

The `@RECON.CLOUD.CONFIG.URI@` token is replaced at deployment by the CI/CD pipeline or deployment script.

### 1.2 Configuration Loading Order

```
1. bootstrap.properties          → Config server connection
2. Config Server (remote)        → Externalized properties
3. application.properties        → Local defaults / overrides
4. application-{profile}.properties → Mode-specific settings
5. System properties / env vars  → Runtime overrides
```

### 1.3 Runtime Refresh

The `/actuator/refresh` endpoint is exposed, allowing properties to be refreshed without restarting the JVM:
```
POST http://localhost:14030/jitr-recon/actuator/refresh
```

---

## 2. Mode-Specific Profiles

Each batch mode runs on its own profile with a dedicated server port. Only one mode runs per JVM instance.

| Profile | Activated Via | Server Port | Properties File |
|---------|--------------|-------------|-----------------|
| **prebill** | `-Dspring.profiles.active=prebill` | 14030 | `application-prebill.properties` |
| **mini** | `-Dspring.profiles.active=mini` | 14040 | `application-mini.properties` |
| **deltamini** | `-Dspring.profiles.active=deltamini` | 14010 | `application-deltamini.properties` |
| **full** | `-Dspring.profiles.active=full` | 14030 | `application-full.properties` |
| **odr** | `-Dspring.profiles.active=odr` | 14050 | `application-odr.properties` |
| **special** | `-Dspring.profiles.active=special` | 14010 | `application-special.properties` |
| **recon** | (base config) | — | `application-recon.properties` |

**Context Path (all modes):** `/jitr-recon`

### 2.1 Mode Behavioral Differences

| Feature | Prebill | Mini | DeltaMini | Full | ODR | Special |
|---------|---------|------|-----------|------|-----|---------|
| Handicap Indicator | Disabled | Disabled | Disabled | Enabled | Disabled | Disabled |
| Splitter | Yes | Yes | Yes | Yes | No | No |
| Zone Parallel | No | No | No | Configurable | No | No |
| RBM Extract Threads | 60 | 60 | 60 | 60 | Configurable | Configurable |
| Vision File Count | 25 (consumer) | 25 | 25 | 25 | Varies | Varies |
| Hold Check | Yes | Yes | Yes | Yes | Yes | Yes |
| Test Mode | Configurable | Configurable | Configurable | Configurable | Configurable | N/A |

---

## 3. Property Encryption (Jasypt + BouncyCastle)

### 3.1 Mechanism

All database passwords are encrypted using Jasypt and stored in the `ENC(...)` wrapper format:

```properties
spring.ubsr.psd=ENC(RL025rTXrIDccLmoHFOc0Q==)
spring.mzadmin.psd=ENC(kKTj5VfsyI/Qqx1P27N7Ug==)
spring.mzaud.psd=ENC(mm3Tc31KPxCTtANGbru+Cg==)
spring.rbm.psd=ENC(JQ7+jOii5BvtbJpKSUhngBLO+rt0/00f)
spring.reed.psd=ENC(KIzwGj9WA7mjrp/SLvKNpA==)
```

### 3.2 Decryption Setup

- **Library:** `jasypt-spring-boot-starter:3.0.5`
- **Crypto Provider:** BouncyCastle (`bcprov-jdk18on:1.78.1`)
- **Activation:** `@EnableEncryptableProperties` on `StartUp.java`
- **Default Key:** `"Recon@123"` (in `DatabaseContext.java` — overridable via JVM args)
- **Key Override:** `-Djasypt.encryptor.password=<secret>` at JVM startup

### 3.3 DatabaseContext Password Decryption

```java
// DatabaseContext.java
private String key = "Recon@123";  // default encryption key

// Passwords are decrypted transparently by Jasypt during Spring property resolution
// No manual decryption code needed — the @EnableEncryptableProperties annotation
// intercepts property reads and decrypts ENC(...) values automatically
```

---

## 4. Thread Pool Sizing Parameters

### 4.1 Extraction Phase

| Property | Default | Purpose |
|----------|---------|---------|
| `zone.threads` | 4–5 | Zone-level parallel processing |
| `prebill.rbm.extract.threads` | 60 | RBM extraction (across 6 nodes) |
| `full.rbm.extract.threads` | 60 | Full batch RBM extraction |
| `prebill.ubsr.extract.threads` | 1 | UBSR single-source extraction |
| `mdn.ubsr.extract.threads` | 1 | MDN extraction from UBSR |
| `prebill.preloadthreads` | 3–8 | Reference data preloading |
| `prebill.primary.extract.threads` | 10 | DVC SIM file extraction |
| `prebill.secondary.extract.threads` | 10 | Non-DVC SIM file extraction |
| `sub.account.extract.thread.size` | 5 | Sub-account processing |

### 4.2 Comparison Phase

| Property | Default | Purpose |
|----------|---------|---------|
| `prebill.compare.threads` | 1 | UBSR billing comparison |
| `full.compare.threads` | 1 | Full batch billing comparison |
| `prebill.rating.compare.threads` | 30 | RBM rating comparison |
| `full.rating.compare.threads` | 30 | Full batch rating comparison |
| `prebill.large.compare.threads` | 1 | DVC file comparison |
| `full.large.compare.threads` | 1 | Full batch DVC comparison |

### 4.3 Fix Phase

| Property | Default | Purpose |
|----------|---------|---------|
| `rating.fix.file.level.threads.core.pool` | 6 | File-level fix parallelism (core) |
| `rating.fix.file.level.threads.max.pool` | 7 | File-level fix parallelism (max) |
| `jis.threads` | 20 | JIS API call parallelism |
| `rating.fix.threads.core.pool.cpard` | 8 | CPARD-specific threads |
| `rating.fix.threads.core.pool.rerate` | 1–5 | Rerate processing |
| `rating.fix.threads.core.pool.prorate` | 10 | Prorate processing |
| `rating.fix.threads.core.pool.evsrc` | 10 | Event source processing |

### 4.4 DVS Threads

| Property | Default | Purpose |
|----------|---------|---------|
| `dvs.threads.cust` | 2 | Customer DVS queries |
| `dvs.threads.acct` | 2 | Account DVS queries |
| `dvs.threads.mdn` | 5 | MDN DVS queries |

### 4.5 Batch & Flush Rates

| Property | Default | Purpose |
|----------|---------|---------|
| `batch.size` | 1000 | Records per batch in UBSR |
| `db.flush.rate` | 20000 | General DB commit interval |
| `db.flush.rate.rerate` | 1000 | Rerate commit interval |
| `billing.fix.flush.rate` | 1000 | Billing fix commit interval |
| `ubsr.split.dgid.flush.size` | 10000 | DGID-based buffer flush |
| `compare.file.write.flush.size` | 10000 | File output stream flush |

---

## 5. Feature Flag Configuration

### 5.1 YAML Files

Feature flags are defined in YAML files stored in the resources directory and managed via AWS AppConfig:

**Location:** `src/main/resources/nonprod/fature-flag.yml` and `src/main/resources/prod/fature-flag.yml`

```yaml
configProfileName: vision2BillingAACfgProfile
featureFlags:
  - vision2AAFileCollectorFFlag:
      description: "Vision2 AA DAILY file collector"
      flag-enabled-state: true
      deprecation-date: 2026/04/30
      deploymentStrategy: AppConfig.Linear50PercentEvery30Seconds
      attributes:
        actPctOwner: PCT223
        target: all_users
        jiraTicket: V2DT-40190
        enabledForUsers: allUser
```

### 5.2 Application Properties for Feature Flags

| Property | Purpose |
|----------|---------|
| `feature.flag.enabled` | Master switch (Y/N) — if N, all flags bypassed |
| `FF.applicationName` | Application name in AWS AppConfig |
| `FF.configProfile` | Config profile for flag evaluation |
| `FF.FileCollector.flagName` | Specific flag name to evaluate |
| `FF.serverUrl` | AWS AppConfig server URL |
| `FF.serverPort` | AWS AppConfig server port |
| `FF.environment` | Environment (nonprod/prod) |

### 5.3 Property-Based Process Toggles

Beyond AWS AppConfig, there are 60+ property-based flags controlling specific features:

| Category | Key Flags |
|----------|----------|
| **Vision 2.0** | `vision2.0.enabled`, `vision2.ph2`, `mdnless.enable`, `mdnless.billing.enable` |
| **Processing** | `dvc.enable`, `handicapInd.enable`, `gapi.flag.enable`, `enable.Pgp.Finder` |
| **Scheduling** | `odr.scheduler.on`, `rerate.scheduler.on`, `rerate.loader.scheduler.on` |
| **Test Modes** | `prebill.testModeEnabled`, `mini.testModeEnabled`, `full.testModeEnabled`, `odr.testModeEnabled` |
| **Offline** | `rerate.update.offline`, `reedRerate.update.offline`, `evsrc.delete.offline`, `sfopomc.update.offline` |
| **Splitter** | `SPLITTER_PRE_ON`, `SPLITTER_FULL_ON`, `SPLITTER_MINI_ON`, `SPLITTER_DMD_ON` |
| **Parallelism** | `zone.parallel.process`, `ubtoub.parallel.process`, `ubtorbm.parallel.process`, `node.parallel.process` |
| **Thresholds** | `billing.ubsronly.threshold.check.enabled`, `rating.nonubsronly.threshold.check.enabled`, `deleteHoldEvaluationEnabled` |

---

## 6. Directory Structure Requirements

### 6.1 Root Directory Layout

```
C:/RECON/                                ← root.dir
├── VISION/                              ← prebill.vision.file.dir (prebill input)
├── MINIVISION/                          ← mini.vision.file.dir (mini input)
├── DELTAMINIVISION/                     ← deltamini.vision.file.dir
├── FULLVISION/                          ← full.vision.file.dir (full input)
├── ODRVISION/                           ← odr.vision.file.dir (ODR input)
├── DMDVISION/                           ← DMD.vision.file.dir
├── MINIODRVISION/                       ← mini ODR large cust files
│
├── VISION_BILLING/                      ← prebill Vision 2.0 billing
├── MINIVISION_BILLING/                  ← mini Vision 2.0 billing
├── FULLVISION_BILLING/                  ← full Vision 2.0 billing
├── DELTAMINIVISION_BILLING/             ← delta mini Vision 2.0 billing
│
├── ARCHIVE/                             ← archive.dir (processed files)
│   └── {DDMMYYYY}/
│       └── {zone}_{cycleid}/
├── ARCHIVE_BILLING/                     ← archive.dir.billing (V2 archives)
│
├── ERROR/                               ← error.dir (failed files)
├── ERROR_BILLING/                       ← error.dir.billing (V2 errors)
│
├── HOLD/                                ← recon.holdfile.path
│   ├── RECON.HOLD                       ← blocks ALL batches
│   ├── PREBILL.HOLD                     ← blocks prebill only
│   ├── MINIBATCH.HOLD                   ← blocks mini only
│   ├── DELTAMINIBATCH.HOLD              ← blocks delta mini
│   ├── FULLBATCH.HOLD                   ← blocks full only
│   └── ODR.HOLD                         ← blocks ODR only
├── HOLD_BILLING/                        ← recon.billing.holdfile.path
│
├── SCHEDULER/
│   └── RATING/                          ← scheduler.file.path
│
├── BPR/                                 ← bpr.dir (billing process reports)
│
└── splitter/
    └── INPUT/                           ← Splitter source path
        ├── MINI_ARCHIVE/
        ├── FULL_ARCHIVE/
        └── PRE_ARCHIVE/

C:/LOGS/                                 ← Logging directory
├── jitrRecon.log                        ← Main execution logs
├── jitrJisCalls.log                     ← JIS API call logs
├── jitrReconSplitter.log                ← Splitter operation logs
├── jitrReconUI.log                      ← UI/Console logs
├── ELKLogs.log                          ← ELK integration logs
└── jitr_recon_validation.log            ← Delete validation logs
```

### 6.2 Zone Directory Convention

Inside each vision directory, files are organized by `{zoneId}_{billCycleId}`:
```
C:/RECON/VISION/
├── 01_15/                               ← Zone 01, Bill Cycle 15
│   ├── RCN_CUSTOMER_01_15.dat
│   ├── RCN_BL_ACCT_01_15.dat
│   ├── RCN_SF_MTN_01_15.dat
│   └── ... (17-26 extract files)
├── 02_15/                               ← Zone 02, Bill Cycle 15
└── ...
```

---

## 7. Database Connection Parameters

### 7.1 Development Environment

```properties
spring.jdbc.driver=oracle.jdbc.driver.OracleDriver
spring.jdbc.url=jdbc:oracle:thin:@tpsrbmvhldd-scan.verizon.com:2056/ub2ead31
spring.jdbc.url.rbm=jdbc:oracle:thin:@tpsrbmvhldd-scan.verizon.com:2056/r2e1ad31
spring.jdbc.url.rbm1=jdbc:oracle:thin:@tpsrbmvhldd-scan.verizon.com:2056/r2e1ad31
# (repeat for rbm2-6)
```

### 7.2 HikariCP Pool Settings

```properties
spring.datasource.hikari.connection-timeout=20000      # 20 seconds
spring.datasource.hikari.minimum-idle=10               # Minimum idle connections
spring.datasource.hikari.maximum-pool-size=10           # Max connections
spring.datasource.hikari.idle-timeout=10000             # 10 seconds idle
spring.datasource.hikari.max-lifetime=1800000           # 30 minutes (production)
spring.datasource.hikari.auto-commit=true
spring.datasource.hikari.connectionTestQuery=SELECT 1 from DUAL
hikari.newcode.enabled=Y                                # Use new HikariConfig path
```

### 7.3 Database Users

| User | Database | Purpose |
|------|----------|---------|
| `ubsr` | UBSR | Primary billing records |
| `mzadmin` | MzAdmin | Admin/config operations |
| `mzaud` | Audit | Audit trail writes |
| `geneva_admin` | RBM | Rating billing mart |
| `reed` | REED | Error recovery data |

---

## 8. Messages & Error Templates

### 8.1 messages.properties Structure

The `messages.properties` file contains **100+ message templates** used for email alerts and logging.

**Subject Templates (SUBxxxxx):**

| Code | Template | Used For |
|------|----------|----------|
| SUB00001 | File validation errors | Missing/corrupt input files |
| SUB00002 | Cycle-specific file validation | Zone/cycle file count mismatch |
| SUB00028 | JIS timeout alerts | JIS API not responding |
| SUB00038 | Delete hold evaluation results | Threshold breach notification |
| SUB00043 | Gate status updates | Processing gate check results |

**Application Messages (APPxxxxx):**

| Code | Template | Used For |
|------|----------|----------|
| APP00001 | Generic processing error | Unhandled exception alert |
| APP00002 | Unexpected file counts | File count validation |
| APP00005 | Empty file alerts | Zero-record extract files |
| APP00006 | Process duration warnings | Long-running batch alert |
| APP00018-59 | Various validation/error messages | Thresholds, DB exceptions, data issues |

**System Messages (SYSxxxxx):**

| Code | Template | Used For |
|------|----------|----------|
| SYS00001 | Generic system error | System-level failures |

---

## 9. Logging Configuration

### 9.1 Log4j2 Architecture

```
Log4j2 (2.17.2) + LMAX Disruptor (3.3.4)
├── Async logging via Disruptor ring buffer
├── RollingRandomAccessFile appenders
├── Time-based rolling: daily
├── Size-based rolling: 100 MB per file
├── Max retention: 50 files per appender
└── Immediate flush: true
```

### 9.2 Log Files

| Logger | File | Purpose |
|--------|------|---------|
| `jitrReconLog` | `C:/LOGS/jitrRecon.log` | Main application execution |
| `jitrJisCalls` | `C:/LOGS/jitrJisCalls.log` | All JIS API call details |
| `jitrReconSplitterLog` | `C:/LOGS/jitrReconSplitter.log` | File splitting operations |
| `jitrReconUI` | `C:/LOGS/jitrReconUI.log` | REST controller activity |
| `ELKLogs` | `C:/LOGS/ELKLogs.log` | ELK stack integration |
| `jitrReconValidation` | `C:/LOGS/jitr_recon_validation.log` | Delete validation audit |

### 9.3 Log Pattern

```
%d{MM-dd-yyyy HH:mm:ss:SSS} | [%thread] | %C{1} | %L | %-5.5p | %m%n
```

| Field | Example | Meaning |
|-------|---------|---------|
| `%d` | `04-22-2026 14:30:45:123` | Timestamp |
| `%thread` | `[pool-1-thread-3]` | Thread name |
| `%C{1}` | `ReconScheduler` | Short class name |
| `%L` | `245` | Line number |
| `%-5.5p` | `INFO ` | Log level (padded) |
| `%m` | `Processing zone 01_15` | Message |

### 9.4 ELK Logger

`ELKLogger` (`com.verizon.jitr.recon.common`) writes structured logs to the ELK-specific appender for centralized log aggregation.

---

## 10. Deployment Tokens

Properties marked with `@TOKEN@` are replaced during the CI/CD deployment pipeline:

| Token | Purpose |
|-------|---------|
| `@RECON.CLOUD.CONFIG.URI@` | Config server URL |
| `@RECON.SUPPORT.ALERT.EMAIL.DISTRO@` | Support team email distribution list |
| `@RECON.ALERT.ENV@` | Environment name for alerts (SIT/PROD) |

---

## 11. DST (Daylight Saving Time) Configuration

The application maintains a comprehensive DST date mapping from 2007 to 2024 for accurate billing timestamp calculations:

```properties
dst.start.2024=2024-03-10
dst.end.2024=2024-11-03
dst.start.2023=2023-03-12
dst.end.2023=2023-11-05
# ... back to 2007
```

This is used by `JisUtil` and timing-sensitive comparisons to handle timezone-aware billing date calculations.
