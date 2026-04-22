# JITR Recon — Batch Job Inventory & Scheduling Model

## 1. Batch Modes Overview

JITR Recon operates as a **scheduled batch engine**. There is no user-initiated processing; every reconciliation cycle is triggered by a cron scheduler. The system supports **7 distinct batch modes**, each targeting a different scope of billing data.

| Batch Mode | Constant ID | Server Port | Purpose |
|------------|-------------|-------------|---------|
| **PREBILL** | 97 | 14030 | Pre-billing reconciliation — runs before bill cycle closes; synchronizes UBSR ↔ RBM for the upcoming bill |
| **MINIBATCH** | 98 | 14040 | Mini reconciliation — smaller, more frequent pass targeting recently changed records |
| **DELTAMINI** | 89 | 14010 | Delta mini — incremental mini-batch processing only changes since last mini run |
| **FULLBATCH** | 90 | 14030 | Full reconciliation — comprehensive pass across all customers/accounts/products; largest workload |
| **ODR** | 99 | 14050 | On-Demand Rating — processes individual rating requests from `MZADMIN.REF_RECON_ONDEMAND` table |
| **ODR BILLING** | — | 14030 | On-Demand Rating Billing side — billing-specific ODR processing |
| **DMD** | — | 14020 | Data Management/Demand — special processing for data management requests |
| **SPECIAL** | — | 14010 | Special reconciliation scenarios (B2B, restage, etc.) |

### Batch Mode Characteristics

| Mode | Extract Scope | Typical Runtime Window | Thread Profile | Splitter |
|------|--------------|----------------------|---------------|----------|
| PREBILL | Bill-cycle specific customers | 8 AM – 11 PM | 60 RBM / 1 UBSR / 30 compare | Yes |
| MINI | Recently modified records | 1 AM – 11 PM | Same as PREBILL | Yes |
| DELTAMINI | Delta since last mini | Varies | Same as PREBILL | Yes |
| FULL | All customers in zone | 1 AM – 11 PM | 60 RBM / 1 UBSR / 30 compare | Yes |
| ODR | Single customer on-demand | 10 PM – 11 PM (restricted) | Configurable | No |
| DMD | Data management batch | 1 AM – 11 PM | Configurable | Yes |
| SPECIAL | Ad-hoc | Manual trigger | Configurable | No |

---

## 2. Cron Scheduling Configuration

All scheduled methods are in `BatchConfig.java` (`@Component`) and use Spring `@Scheduled` annotations with cron expressions loaded from property files.

### 2.1 Reconciliation Schedulers

| Schedule Method | Cron Property | Default Value | Behavior |
|----------------|---------------|---------------|----------|
| `runPrebill()` | `prebill.scheduler.cron` | `0 0/2 * * * ?` | Every 2 min — polls for PREBILL work |
| `runminibatch()` | `mini.scheduler.cron` | `0 0/2 * * * ?` | Every 2 min — polls for MINI work |
| `rundeltaminibatch()` | `deltamini.scheduler.cron` | `0 0/2 * * * ?` | Every 2 min — polls for DELTAMINI work |
| `runfullBatch()` | `full.scheduler.cron` | `0 0/2 * * * ?` | Every 2 min — polls for FULL work |
| `runodrRecon()` | `odr.scheduler.cron` | `0 0/10 22-23 * * ?` | Every 10 min (10–11 PM only) |
| `runBillingodrRecon()` | `billingodr.scheduler.cron` | `0 0/2 0-1,20-23 * * ?` | Every 2 min (midnight–1 AM, 8–11 PM) |
| `runDmdBatch()` | `dmd.scheduler.cron` | `0 0/2 1-23 * * ?` | Every 2 min (1 AM – 11 PM) |

### 2.2 Splitter Schedulers

| Schedule Method | Cron Property | Default Value | Behavior |
|----------------|---------------|---------------|----------|
| `startPreSplitter()` | `splitter.prebill.scheduler.cron` | `0 0/2 8-23 * * ?` | Every 2 min (8 AM – 11 PM) |
| `startMiniSplitter()` | `splitter.mini.scheduler.cron` | `0 0/2 1-23 * * ?` | Every 2 min (1 AM – 11 PM) |
| `startDeltaMiniSplitter()` | `splitter.deltamini.scheduler.cron` | `0 0/2 1-23 * * ?` | Every 2 min (1 AM – 11 PM) |
| `startFullSplitter()` | `splitter.full.scheduler.cron` | `0 0/2 1-23 * * ?` | Every 2 min (1 AM – 11 PM) |
| `startDMDSplitter()` | `dmdSplitter.full.scheduler.cron` | `0 0/2 1-23 * * ?` | Every 2 min (1 AM – 11 PM) |
| `zeroDollarFileTransfer()` | `zeroDollar.full.scheduler.cron` | `0 0/2 9-23 * * ?` | Every 2 min (9 AM – 11 PM) |

### 2.3 Other Schedulers

| Schedule Method | Cron Property | Default Value | Behavior |
|----------------|---------------|---------------|----------|
| `ReconStatusScheduler` | `recon.status.delay.prop` | `0 0 */1 * * ?` | Every hour — status check |
| `AlertScheduler` | — | — | Alert monitoring on schedule |
| B2B Start | `process.b2b.start` | `0 0 23 * * *` | Daily at 11 PM |
| Rerate Start | `process.rerate.start` | `0 0 09,17 * * *` | 9 AM & 5 PM daily |
| Rerate Loader | `process.rerateLoader.start` | `0 0/10 * * * *` | Every 10 minutes |

### 2.4 Scheduling Flow

Each scheduled method follows the same pattern:

```
@Scheduled(cron = "${xxx.scheduler.cron}")
public void runXxx() {
    1. Check feature.flag.enabled property
    2. If enabled → call featureFlagManager.getFlag(application, profile, flag)
    3. If flag allows → call reconScheduler.runRecon()  [or splitter.startSplitter()]
    4. If flag denies → log "disabled by feature flag" and skip
}
```

Additionally, splitter schedulers validate runtime windows:
```
validRuntime(currentDate, cronExpression, intervalMinutes)
→ ensures current time is within the expected execution window
→ prevents stale triggers from executing
```

---

## 3. Feature Flag Gating (AWS AppConfig)

### 3.1 Architecture

```
┌──────────────┐       ┌─────────────────┐       ┌─────────────────┐
│  BatchConfig  │──────▶│FeatureFlagManager│──────▶│  AWS AppConfig   │
│  (scheduler)  │       │   (SDK client)   │       │  (remote store)  │
└──────────────┘       └─────────────────┘       └─────────────────┘
```

### 3.2 Configuration

```java
// FeatureFlagConfig.java
@Bean
public FeatureFlagManager featureFlagManager() {
    return new FeatureFlagManager.Builder(serverUrl, serverPort, environment).build();
}
```

### 3.3 Flag Properties

| Property | Purpose |
|----------|---------|
| `feature.flag.enabled` | Master switch — if `N`, all flags are bypassed (jobs run unconditionally) |
| `FF.applicationName` | Application identifier in AppConfig |
| `FF.configProfile` | Config profile name (e.g., `vision2BillingAACfgProfile`) |
| `FF.FileCollector.flagName` | Flag name (e.g., `vision2AAFileCollectorFFlag`) |

### 3.4 Feature Flag YAML (AWS AppConfig)

```yaml
# nonprod/fature-flag.yml and prod/fature-flag.yml
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

### 3.5 Decision Flow

```
Is feature.flag.enabled == Y ?
├── YES → Call featureFlagManager.getFlag(app, profile, flag)
│         ├── Flag = true  → Execute batch job
│         └── Flag = false → Skip batch job (log warning)
└── NO  → Execute batch job unconditionally
```

---

## 4. Splitter Mechanism — Zone-Based File Distribution

### 4.1 Purpose

The `ReconSplitter` partitions large extract files across multiple JVM instances for parallel processing. It ensures each customer's data lands on a single JVM to maintain transactional integrity.

### 4.2 Splitting Algorithm

```
INPUT: Source directory with extract files organized by zone_billcycle

FOR EACH zone_billcycle:
    1. Generate splitter audit ID
    2. Determine if home zone or non-home zone:
       ├── NON-HOME → Copy all files to single JVM (no split)
       └── HOME → Apply modulus splitting:

           MODULUS CALCULATION:
           jvmSlot = (customerID % (maxMod - minMod + 1)) + minMod
           
           Maps each customer to a specific JVM based on their ID

    3. Check for special processing:
       ├── Regular customers → processZone()
       ├── Big B2B accounts → processZoneForBigAccount() [dedicated cluster]
       ├── B2B single set → processZoneB2B1Set()
       └── Full parallel → processParallel() [4-way split]

    4. Write output to: jvm_path/acctId/files

    5. Write completion markers:
       ├── PBRC (PreBill completed)
       ├── MNRC (Mini completed)
       ├── FLRC (Full completed) / FLR2 (Full V2)
       ├── MNRB (DeltaMini completed)
       └── DVRB (DMD completed)

    6. Insert into BILLCYCLECONFIG tracking table
```

### 4.3 Cluster Configuration

| Mode | Primary Cluster | Secondary Cluster | Mod Range |
|------|----------------|------------------|-----------|
| PREBILL | `${prebill.primary.cluster}` | `${prebill.secondary.cluster}` | min=1, max=1 |
| MINI | `${mini.primary.cluster}` | `${mini.secondary.cluster}` | min=1, max=1 |
| FULL | `${full.primary.cluster}` | `${full.secondary.cluster}` | min=1, max=2 |
| FULL (Large) | `${full.large.customer.primary.cluster}` | — | separate mod range |
| ODR/OnDemand | `${ondemandfull.primary.cluster}` | — | min=1, max=1 |

### 4.4 Large Customer Handling

Large B2B customers (identified via `SplitterLargeCustList.txt` or properties) are routed to a dedicated high-capacity cluster to prevent processing bottlenecks.

```
Is customer in large customer list?
├── YES → Route to full.large.customer.primary.cluster
│         with fullLargeCustMinMod/MaxMod range
└── NO  → Route to standard cluster with standard mod range
```

---

## 5. Key-Rules Engine

### 5.1 Purpose

The `key-rules.properties` file defines the **comparison rigor** applied during the extract-compare phase. Each rule maps a process type to a set of numbered steps that determine which fields are compared and at what level of strictness.

### 5.2 Rule Structure

```
DIRECTION_ENTITY=step0|step1|step2|step3|...|stepN
```

### 5.3 Rule Categories

**UBSR-to-UBSR Comparisons:**
```properties
UBSRTOUBSR_ACCTPGP=0|1|2|3|4|5|6|7
UBSRTOUBSR_SFOPOPD=0|1|2|3|4|5|6|7
UBSRTOUBSR_PROMOPD=0|1|2|3|4|5|6|7
UBSRTOUBSR_SPOLNPD=0|1|2|3|4|5|6|7
UBSRTOUBSR_SPLGPPDCPARD=0|1|2|3|4|5|6|7
UBSRTOUBSR_CANTENNA=0|1|2|3|4|5|6|7
UBSRTOUBSR_PPLANPD=0|1|2|3|4|5|6|7
UBSRTOUBSR_SFOSPOAC=0|1|2|3|4|5|6|7
```

**UBSR-to-RBM Comparisons:**
```properties
UBSRTORBM_CUST=0|1|2|3|4|5|6|7|8|9|10
UBSRTORBM_ACCT=0|1|2|3|4|5|6|7|8|9|10
UBSRTORBM_EVSRC=0|1|2|3|4|5|6|7|8|9|10
UBSRTORBM_SFOPD=0|1|2|3|4|5|6|7|8|9|10
UBSRTORBM_ALLPD=0|1|2|3|4|5|6|7|8|9|10
UBSRTORBM_CPARD=0|1|2|3|4|5|6|7|8|9|10
```

**Vision-to-UBSR Comparisons:**
```properties
VISIONTOUBSR_CUSTOMER=0|1|2|3|4|5|6|7
VISIONTOUBSR_BLACCT=0|1|2|3|4|5|6|7
VISIONTOUBSR_SFMTN=0|1|2|3|4|5|6|7
VISIONTOUBSR_LNSVCPROD=0|1|2|3|4|5|6|7
```

**Billing Comparisons (Vision 2.0):**
```properties
BILLING_SUBSCRIPTION=0|1|2|3|4|5|6|7|8|9|10|11
BILLING_PRODUCTS=0|1|2|3|4|5|6|7|8|9|10|11
BILLING_ACCOUNT=0|1|2|3|4|5|6|7|8|9|10|11
```

### 5.4 How Rules Drive Processing

```
1. ReconScheduler determines batch type (PREBILL/MINI/FULL/etc.)
2. For each comparison direction (UBSRTORBM, UBSRTOUBSR, VISIONTOUBSR):
   a. Load the rules from key-rules.properties
   b. For each entity (CUST, ACCT, EVSRC, etc.):
      - Get step list: [0, 1, 2, 3, ..., N]
      - Execute comparison at each step level
      - Step 0 = key fields only (identity match)
      - Higher steps = progressively stricter field comparisons
   c. Records failing comparison are written to diff files
3. Diff files feed into the fix processor chain
```

### 5.5 Step Semantics

| Step | Meaning |
|------|---------|
| 0 | Key field comparison (identity: customer ID, account ID) |
| 1–3 | Core attribute comparison (name, status, dates) |
| 4–7 | Extended attribute comparison (pricing, features, flags) |
| 8–11 | Full attribute comparison (all fields including advisory) |

---

## 6. Hold File Mechanism

Before any batch runs, the system checks for `.HOLD` files that act as **manual circuit breakers**:

| Hold File | Blocks |
|-----------|--------|
| `RECON.HOLD` | All batch types |
| `PREBILL.HOLD` | PREBILL only |
| `MINIBATCH.HOLD` | MINI only |
| `DELTAMINIBATCH.HOLD` | DELTAMINI only |
| `FULLBATCH.HOLD` | FULL only |
| `ODR.HOLD` | ODR only |

**Location:** `C:/RECON/HOLD/` (configurable via `recon.holdfile.path`)

Operations teams can drop or remove these files to control batch execution without redeploying the application. After processing, hold files are moved to the archive folder.
