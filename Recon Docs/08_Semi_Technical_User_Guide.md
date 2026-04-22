# JITR Recon — Semi-Technical User Guide

## How the Billing Reconciliation System Works

*A guide for stakeholders, product owners, and operations teams — no code required.*

---

## 1. What Is Billing Reconciliation?

Verizon's wireless billing operates across **two major systems**:

| System | Full Name | Role |
|--------|-----------|------|
| **UBSR** | Unified Billing System Records | The **source of truth** — stores what customers actually have (accounts, plans, devices, features) |
| **RBM** | Rating Billing Mart | The **rating engine** — calculates charges based on what it believes the customer has |

**The problem:** These two systems can drift apart. A customer may have changed their plan in UBSR, but RBM may not have received the update. When this happens, customers get billed incorrectly — either overcharged or undercharged.

**JITR Recon's job:** Compare UBSR and RBM side-by-side, find every difference, and automatically fix RBM so it matches UBSR. This happens **before bills are generated**, ensuring accurate charges.

Think of it like a bank reconciliation: you compare your checkbook (UBSR) against the bank statement (RBM) and correct any differences.

---

## 2. The Different Batch Modes (When Each Runs)

The system doesn't reconcile everything at once. Instead, it runs in **different modes** depending on timing and scope:

### Pre-Bill Reconciliation
- **When:** Before each bill cycle closes
- **What:** Checks all customers due to bill that cycle
- **Why:** Ensures accuracy right before charges are calculated
- **Frequency:** Polls every 2 minutes during operating hours

### Mini Batch
- **When:** Ongoing, throughout the day
- **What:** Checks only recently-changed customer records
- **Why:** Catches changes quickly between full runs
- **Frequency:** Every 2 minutes

### Delta Mini Batch
- **When:** Between mini batches
- **What:** Processes only the incremental changes since the last mini batch
- **Why:** More efficient than re-running the full mini batch

### Full Batch
- **When:** Typically during off-peak hours
- **What:** Checks **every customer** in a billing zone — the most comprehensive run
- **Why:** Catches anything that the smaller batches may have missed
- **Frequency:** Every 2 minutes during operating window

### On-Demand Rating (ODR)
- **When:** Evenings (10 PM – 11 PM)
- **What:** Processes individual customer requests from a queue
- **Why:** Used for customer-specific investigations or special corrections

### DMD (Data Management)
- **When:** As needed, 1 AM – 11 PM
- **What:** Data management/demand-driven processing

### Special Reconciliation
- **When:** Ad-hoc, manually triggered
- **What:** Custom processing for special scenarios (like B2B corrections or restaging)

---

## 3. What Happens During a Reconciliation Run

Here's the end-to-end flow in business terms:

### Step 1: Safety Checks
Before any processing begins, the system checks:
- **Is there a "Hold" in place?** — Operations can place a hold file (like a stop sign) to prevent processing
- **Is the feature enabled?** — Feature flags can remotely enable or disable specific batch types
- **Is there enough disk space?** — Ensures the system won't crash mid-run

### Step 2: Gathering the Data
The system extracts current data from both databases:
- From **UBSR**: "What does the source of truth say this customer has?"
- From **RBM**: "What does the rating system think this customer has?"

This involves running pre-built database queries that pull records for customers, accounts, service plans, products, event sources, and other billing entities.

### Step 3: Comparing the Data
The system places UBSR and RBM data side-by-side and classifies every record:
- **Match** — Both agree → no action needed
- **In UBSR only** — Customer has something in UBSR that RBM doesn't know about → needs to be **added** to RBM
- **In RBM only** — RBM has something that no longer exists in UBSR → needs to be **removed** from RBM
- **Different values** — Both have the record, but the details don't match → needs to be **updated** in RBM

### Step 4: Safety Gates (Threshold Checks)
Before applying fixes, the system checks: *"Does this look right?"*

For example, if the system finds it needs to **delete 50,000 customer records**, that's suspicious — it could indicate a system error rather than legitimate changes. The system checks these counts against pre-set thresholds:
- If counts are within normal range → proceed
- If counts exceed thresholds → **stop processing**, alert the operations team

### Step 5: Applying the Fixes
For each discrepancy found, the system:
1. Determines the correct fix (create, update, or delete)
2. Sends a request to the **JIS system** (the gateway to RBM) to make the change
3. Records what was done for audit purposes

There are **50+ different types of fixes** covering:
- Customer information (name, address, status)
- Account details (billing address, contacts)
- Service plans and pricing
- Product features (special features, voice, data, messaging)
- Device and equipment records
- Promotional offers and discounts

### Step 6: Post-Processing
After fixes are applied:
- **Proration** calculations are run for partial-cycle charges
- **Rerate** requests are submitted if pricing needs recalculation
- **Audit records** are written (who, what, when, how many)
- **Email alerts** are sent with a summary of the run
- Input files are **archived** for historical reference

---

## 4. The Fix Process: Finding and Correcting Discrepancies

### How Discrepancies Are Categorized

| Category | Example | Action |
|----------|---------|--------|
| **UBSR-to-RBM** | Customer changed plan in UBSR, RBM still shows old plan | Update RBM via JIS |
| **UBSR-to-UBSR** | Internal UBSR inconsistency (orphaned records) | Clean up UBSR data |
| **RBM-to-RBM** | RBM-internal data consistency issues | Fix RBM directly |
| **Vision-to-UBSR** | Vision system data out of sync with UBSR | Sync Vision data |

### Fix Priority Order

Fixes are applied in a specific order to maintain data integrity:
1. **Customer** fixes first (parent entity)
2. **Account** fixes second (child of customer)
3. **Service Plan** fixes (tied to accounts)
4. **Event Source** fixes (tied to service plans)
5. **Product** and **Pricing** fixes (tied to service plans)
6. ... and so on for 50+ entity types

This ordering prevents situations where, for example, a product fix is applied to an account that hasn't been created yet.

---

## 5. The Hold Evaluation: Protecting Against Bad Data

The Hold system is a **safety net** that prevents the system from making harmful changes.

### What Triggers a Hold?
- **Threshold Breach:** If the number of deletes or inserts exceeds configured limits
- **JIS Timeouts:** If the JIS system (RBM gateway) stops responding
- **Manual Hold:** Operations team drops a `.HOLD` file in the hold directory

### What Happens During a Hold?
1. Processing **stops immediately**
2. An **email and SMS alert** goes to the operations team
3. A `.HOLD` file is created to prevent further runs
4. Operations investigates the cause
5. Once resolved, the hold file is removed and processing resumes

### Why Is This Important?
Without threshold checks, a database glitch could cause the system to delete millions of valid billing records, resulting in widespread billing errors. The hold system ensures human review before large-scale changes proceed.

---

## 6. The Vision 2.0 Migration

### What Is Vision 2.0?
Vision 2.0 is an upgraded version of the billing comparison system. The JITR Recon application supports **both** the legacy system and Vision 2.0 simultaneously.

### What Changes with Vision 2.0?
| Aspect | Legacy | Vision 2.0 |
|--------|--------|------------|
| **Entity Model** | Flat customer/account structure | Richer subscription-based model |
| **File Types** | 17–25 standard files per zone | Enhanced files with additional attributes |
| **Processing** | Sequential pipeline | Parallel processing with template patterns |
| **MDN Handling** | MDN-required | Supports MDN-less devices (tablets, watches) |
| **Product Coverage** | Standard products | Extended product hierarchy |

### How Is It Controlled?
- **Feature flag:** `vision2.0.enabled=Y` activates Vision 2.0 processing
- **Phase gates:** `vision2.ph2`, `vision2.ph2_4`, `vision2.ph3_3` control progressive rollout
- **AWS AppConfig:** Remote feature flags for deployment-level control

The system can run legacy-only, Vision 2.0-only, or both simultaneously depending on configuration.

---

## 7. The Loader and Splitter: Processing at Scale

### The Loader
The **Loader** (`ReconAppContextLoader`) is responsible for initializing the application's processing engine. It:
- Loads configuration from multiple sources (config server, local files, database)
- Initializes in-memory caches with reference data (product catalogs, tariff tables, zone maps)
- Sets up database connections to all required systems
- Prepares the processing pipeline before the first batch runs

Think of it as the "warm-up" phase — similar to how a race car does warm-up laps before a race.

### The Splitter
The **Splitter** (`ReconSplitter`) handles the challenge of processing millions of records efficiently by **dividing the work across multiple servers**.

**How it works:**
1. A large extract file arrives with data for thousands of customers
2. The Splitter assigns each customer to a specific processing server using a mathematical formula (modulus hashing on customer ID)
3. This ensures:
   - All data for one customer always goes to the **same server** (consistency)
   - The work is **evenly distributed** across servers (load balancing)
   - Large enterprise (B2B) customers get dedicated processing capacity

**Example:**
```
Customer 12345 → assigned to Server A
Customer 12346 → assigned to Server B
Customer 12347 → assigned to Server A
Large B2B Enterprise Corp → assigned to dedicated B2B Server
```

After files are distributed, each server processes its share independently, and a completion marker is written when done.

### File-Based Processing
The entire system operates on **files**:
- **Input:** Vision extract files (fixed-width text, one record per line)
- **Intermediate:** Diff files (comparison results)
- **Output:** Archive files (processed data for audit)

Files are organized by **zone** (geographical billing region) and **bill cycle** (billing period, identified by a cycle number). A typical directory looks like:
```
C:/RECON/VISION/01_15/       ← Zone 01, Bill Cycle 15
C:/RECON/VISION/02_15/       ← Zone 02, Bill Cycle 15
```

---

## 8. Monitoring & Alerts

### How to Know if Something Goes Wrong

The system sends **automated alerts** when:
- A batch run fails or encounters errors
- Threshold limits are breached (too many deletes/inserts)
- The JIS system (RBM gateway) is not responding
- Disk space is running low
- Input files are missing or corrupted

**Alert channels:**
- **Email:** Sent to the Recon operations distribution list
- **SMS/Text:** Sent to on-call mobile numbers for critical alerts

### Health Check

A simple web endpoint is available to verify the application is running:
```
GET http://{server}:14030/jitr-recon/versioncheck
```
Returns: build version, JVM uptime, and basic health status.

---

## 9. Key Business Terms Glossary

| Term | Meaning |
|------|---------|
| **UBSR** | Unified Billing System Records — the source of truth for customer data |
| **RBM** | Rating Billing Mart — the system that calculates charges |
| **JIS** | Jacket Information System — the API gateway for updating RBM |
| **Vision** | The extract/compare framework used to produce reconciliation files |
| **Zone** | A geographical billing region (numbered, e.g., Zone 01, Zone 02) |
| **Bill Cycle** | A billing period identified by a numeric cycle ID (e.g., cycle 15) |
| **SPLAN** | Service Plan — a customer's wireless plan |
| **EVSRC** | Event Source — tracks billable events (calls, data usage) |
| **CPARD** | Charge Plan Adjustment Record — promotional discounts and adjustments |
| **SFO** | Special Feature Offer — add-on features (hotspot, international roaming) |
| **PPLAN** | Package Plan — bundled plan structures |
| **MDN** | Mobile Device Number — the phone number |
| **DGID** | Domain Group ID — groups related billing entities |
| **ODR** | On-Demand Rating — process a single customer's recon on request |
| **Prorate** | Calculate partial charges for mid-cycle plan changes |
| **Rerate** | Recalculate charges that were previously billed incorrectly |
| **REED** | Reconciliation Error Event Data — error tracking and recovery system |
| **Hold** | A safety mechanism that stops processing when anomalies are detected |
| **Splitter** | Distributes work across multiple servers for parallel processing |
| **B2B** | Business-to-Business — large enterprise customer accounts |
| **DVS** | Digital Video Services — device and equipment data system |
| **Restage** | Re-process a previously completed batch (for corrections) |
