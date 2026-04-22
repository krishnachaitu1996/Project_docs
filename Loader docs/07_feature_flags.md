# Section 7 — Feature Flags & Toggles

**Application:** JITR Loader 2.0  
**Navigation:** [← External Integrations](./06_external_integrations.md) | [Index](./README.md) | [Environment Config →](./08_environment_config.md)

---

## 7.1 Overview

The Loader uses three mechanisms for runtime control:

| Mechanism | Source | Restart Required | Scope |
|---|---|---|---|
| **Property flags** | `application-ldr.properties` | Yes | Per-instance |
| **List-based flags** | `application-ldr.properties` | Yes | Per-instance |
| **AWS AppConfig** | Remote AWS service | No | Dynamic runtime |

All property-based flags are injected via Spring `@Value` annotations, primarily in `LoaderEventProcessor.java` and `EventProcessHelper.java`.

---

## 7.2 Master Enable Flags

| Property | Values | Default | Description |
|---|---|---|---|
| `visionFlagEnable` | `ON` / `OFF` | `ON` | Master Vision 2.0 processing switch. When `OFF`, OR DataPop types are skipped for non-Vision 2.0 customers |
| `LDR_REFLOW_SCHEDULER_ENABLE` | `ON` / `OFF` | `ON` | Enable/disable the ReflowScheduler entirely |
| `LDR_ILB_SCHEDULER_ENABLE` | `ON` / `OFF` | `ON` | Enable/disable ILB batch scheduler |
| `VISION_2_0_CASSANDRA_ENABLE` | `ON` / `OFF` | `ON` | Use Cassandra for reflow XML storage; falls back to Oracle when `OFF` |

---

## 7.3 Vision 2.0 Processing Flags (V2DT Series)

These flags control specific Vision 2.0 feature behaviors, typically tied to delivery task (DT) work items.

| Flag Property | Values | Description |
|---|---|---|
| `V2DT37304` | `Y` / `N` | Vision 2.0 DT-37304: specific processing behavior change |
| `V2DT39120` | `Y` / `N` | When `Y`, forces **all** customers to be treated as Vision 2.0 regardless of lookup result. Used to globally enable Vision 2.0 processing. |
| `V2DT41237` | `Y` / `N` | DT-41237: additional Vision 2.0 processing gate |
| `V2DT41462` | `Y` / `N` | When `Y`, removes leading zeros from `LINEOFSERVICEIDPART1` and `LINEOFSERVICEIDPART2` for consistent `LINE_OF_SERVICE` computation. Applies to ACTIVATE, REASSIGN, CHANGE_DATA_FEATURES, CHANGE_SFO_FEATURES, TRANSFER. |
| `V2DT41569` | `Y` / `N` | When `Y`, extracts `LN_OF_SVC_ID_NO_P1/P2`, `ADDR_ID_NO`, `CBR_PERSON_ID_NO`, `ECPD_PROFILE_ID` from DataPop KEYS block (with DB_FIELDS fallback) |
| `V2DT41649` | `Y` / `N` | DT-41649: additional Vision 2.0 processing gate |
| `V2DT15112` | `Y` / `N` | When `Y` and MDN is null, acquires transaction-level locks for DataPop types in `transactionLevelLockDataPopList` |

---

## 7.4 Locking Behavior Flags

| Property | Values | Default | Description |
|---|---|---|---|
| `MDN_LEVEL_LOCK` | `Y` / `N` | `Y` | When `Y`, includes MDN in the distributed lock name for finer-grained locking. When `N`, lock is only customer + account level. |
| `V2DT15112` | `Y` / `N` | `N` | When MDN is null and this flag is `Y`, falls back to transaction-level locking for DataPop types in `transactionLevelLockDataPopList` |

---

## 7.5 IOT/Kafka Filtering Flags

| Property | Values | Default | Description |
|---|---|---|---|
| `ROSIOTFLAG` | `true` / `false` | `false` | When `true`, events with `DB_USERID=ROSIOT` are filtered at parse time (error `LDR_D_E7194`) to prevent duplicate processing of transactions already handled via Kafka. Applies to Suspend, Reconnect, and DataPop events. |

---

## 7.6 Instance & Routing Flags

| Property | Values | Description |
|---|---|---|
| `LDR_BATCH_NODE` | `Y` / `N` | Whether this instance processes batch-routed events directly |
| `JITR_INSTANCE` | `BS` / `RT` / `RS` / `RO` | This instance's identifier, used to determine if a customer is on their home instance |
| `UB_PREFIX` | `STC` / other | Used to select B2B parallel biller ID when routing B2B customers |

---

## 7.7 RBM Flags

| Property | Values | Description |
|---|---|---|
| `RBMVD18943` | `Y` / `N` | RBM-specific processing fix for VDMC-18943 |

---

## 7.8 List-Based Configuration Flags

These properties hold comma-separated or delimited lists of values that drive conditional processing.

| Property | Type | Description |
|---|---|---|
| `ORDataPopList` | Comma-separated DataPop TABLE_NAMEs | DataPop types that require Vision 2.0 customer check before processing |
| `transactionLevelLockDataPopList` | Comma-separated DataPop/event names | Events that use transaction-level locking when MDN is null (requires `V2DT15112=Y`) |
| `vision2EnabledEventsList` | Comma-separated event names | Event types enabled for Vision 2.0 billing processing |

---

## 7.9 AWS AppConfig Integration

Beyond static property flags, the `FeatureFlagConfig` class integrates with AWS AppConfig for **live, no-restart flag changes**.

**How it works:**
1. At startup, `FeatureFlagConfig` connects to the configured AWS AppConfig application/environment
2. On each periodic poll, it fetches the current flag state
3. Flag values override or augment property-file values
4. Current state is exposed in the `/versioncheck` health endpoint

**Use case:** Rolling out a new feature to production with the ability to instantly disable it if issues arise, without redeploying the application.

---

## 7.10 Runtime Flag Inspection

### Via Health Endpoint
```
GET /versioncheck
```
Returns a JSON/text response including:
- Application version
- Running instance identifier
- JVM information
- JAR manifest details
- Current feature flag states (both property-based and AWS AppConfig)

### Via Version Endpoint
```
GET /api/versionCheck/
```
Returns summary view of version, instance, and key flag values.

---

## 7.11 Decision Tree: Which Flag Controls What

```
Incoming message
     │
     ├─ IOT filter: ROSIOTFLAG=true + DB_USERID=ROSIOT?
     │     └─ YES → Drop message (LDR_D_E7194)
     │
     ├─ Cross-instance routing
     │     └─ Customer's JITR_INSTANCE ≠ this JITR_INSTANCE?
     │           └─ YES → Route to sibling MQ (no flags, always routes)
     │
     ├─ OR DataPop types (ORDataPopList member)?
     │     ├─ visionFlagEnable=ON? → Process
     │     └─ isVisionCustomer=true? → Process
     │           (else → skip silently)
     │
     ├─ Non-OR DataPop / CPF events?
     │     └─ Always process (Vision customer check does not gate these)
     │
     ├─ Leading zero removal for LOS fields?
     │     └─ V2DT41462=Y? → Strip leading zeros from P1/P2
     │
     ├─ Extended key extraction for DataPops?
     │     └─ V2DT41569=Y? → Extract addr/ecpd/cbr IDs from KEYS/DB_FIELDS
     │
     └─ Lock granularity:
           ├─ MDN_LEVEL_LOCK=Y → include MDN in lock name
           ├─ CHANGE_SFO_FEATURES → also include SFO_ID
           ├─ CUST_DVC_PROV_INFO → also include DEVICE_ID
           └─ V2DT15112=Y + null MDN → transaction-level lock (for listed types)
```
