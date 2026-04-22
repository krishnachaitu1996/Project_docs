# Section 5 — Messaging & Caching

**Application:** JITR Loader 2.0  
**Navigation:** [← Data Operations](./04_data_operations.md) | [Index](./README.md) | [External Integrations →](./06_external_integrations.md)

---

## 5.1 IBM MQ Architecture

All asynchronous message exchange uses **IBM MQ** via Spring JMS. Each Loader instance maintains connections to up to 7 different queue managers — its own for inbound, and one per sibling instance for cross-instance routing.

```
                     ┌─────────────────────────────────────────────────────┐
                     │                    RT Instance (VISN)               │
                     │                                                     │
  Vision CPF ──MQ──▶ │  MQListener                  MQBatchListener        │
                     │  (LDR_LOCAL_MQ_QUEUENAME)    (LDR_BATCHLOCAL_MQ)   │
                     │           │                          │              │
                     │           └──────────┬───────────────┘              │
                     │                      │                              │
                     │           LoaderEventProcessor                      │
                     │                      │                              │
                     │     ┌────────────────┼──────────────────────┐       │
                     │     │                │                      │       │
                     │  ldrJmsTemplate   ldrJmsTemplateBS        ldrJmst   │
                     │  ForRetry          (→ BS instance)        ForProd   │
                     │  (local retry)    ldrJmsTemplateRO        Parallel  │
                     │                   ldrJmsTemplateRS        (B2B)     │
                     └─────────────────────────────────────────────────────┘
```

---

## 5.2 Connection Factory Configuration

All connection factories are defined in `JmsConfiguration.java` using `MQConnectionFactory` from the IBM MQ client library (`com.ibm.mq.allclient:9.1.0.0`).

**Common settings for all factories:**
- Transport type: `WMQConstants.WMQ_CM_CLIENT` (network client mode)
- Client reconnect options: IBM MQ reconnect enabled

| Bean Name | Queue Manager Property | Connected To | Purpose |
|---|---|---|---|
| `dtlConnectionFactory` (primary) | `${LDR_MQ_QMANAGER}` | This instance's QM | Inbound listeners, local retry |
| `roConnectionFactory` | `${LDR_MQ_QMANAGER_RO}` | RO instance QM | Cross-routing to VISE |
| `rtConnectionFactory` | `${LDR_MQ_QMANAGER_RT}` | RT instance QM | Cross-routing to VISN |
| `bsConnectionFactory` | `${LDR_MQ_QMANAGER_BS}` | BS instance QM | Cross-routing to VISB |
| `rsConnectionFactory` | `${LDR_MQ_QMANAGER_RS}` | RS instance QM | Cross-routing to VISW |
| `retryConnectionFactory` | `${LDR_MQ_QMANAGER}` | This instance's QM | Reprocessing queue |
| `prodParallelConnectionFactory` | `${LDR_MQ_QMANAGER_PRODPARALLEL}` | Parallel QM | B2B parallel billing |

---

## 5.3 JMS Listener Container Factories

Spring JMS listener containers manage the polling threads for each queue.

| Factory Bean | Connection Factory | Used By |
|---|---|---|
| `dtlJmsListenerContainerFactory` | `dtlConnectionFactory` | `MQListener`, `MQBatchListener` |
| (per-instance factories) | Instance-specific CFs | Cross-instance consumers (if configured) |

---

## 5.4 JmsTemplate Beans (Outbound Senders)

| Bean Name | Connection Factory | Destination Queue | Used For |
|---|---|---|---|
| `ldrJmsTemplate` | `dtlConnectionFactory` | `${LDR_LOCAL_MQ_QUEUENAME}` | General sends |
| `ldrJmsTemplateRO` | `roConnectionFactory` | `${LDR_MQ_QUEUENAME_RO}` | Route to RO/VISE instance |
| `ldrJmsTemplateRT` | `rtConnectionFactory` | `${LDR_MQ_QUEUENAME_RT}` | Route to RT/VISN instance |
| `ldrJmsTemplateBS` | `bsConnectionFactory` | `${LDR_MQ_QUEUENAME_BS}` | Route to BS/VISB instance |
| `ldrJmsTemplateRS` | `rsConnectionFactory` | `${LDR_MQ_QUEUENAME_RS}` | Route to RS/VISW instance |
| `ldrJmstTemplateForRetry` | `retryConnectionFactory` | `${LDR_REPROCESSING_MQ_QUEUENAME}` | Retry failed messages |
| `ldrJmstTemplateForProdParallel` | `prodParallelConnectionFactory` | Parallel queue | B2B parallel prod |

---

## 5.5 Queue Inventory

| Property | Example Value | Purpose |
|---|---|---|
| `LDR_LOCAL_MQ_QUEUENAME` | `UBD31.VISN.LOCAL.RCV_PUB.QL` | Primary inbound queue for this instance |
| `LDR_BATCHLOCAL_LISTENER_MQ` | (batch variant) | Batch event inbound queue |
| `LDR_REPROCESSING_MQ_QUEUENAME` | (retry queue) | Failed message retry queue |
| `LDR_MQ_QUEUENAME_RO` | (RO queue name) | Outbound to RO/VISE instance |
| `LDR_MQ_QUEUENAME_RT` | (RT queue name) | Outbound to RT/VISN instance |
| `LDR_MQ_QUEUENAME_BS` | (BS queue name) | Outbound to BS/VISB instance |
| `LDR_MQ_QUEUENAME_RS` | (RS queue name) | Outbound to RS/VISW instance |
| `LDR_BATCHLOCAL_MQ_QUEUENAME` | (batch local) | Batch local routing queue |

Biller-specific queue names are also configured for direct routing:

| Property Pattern | Description |
|---|---|
| `LDR_MQ_QUEUENAME_VISB` | Queue for VISB biller |
| `LDR_MQ_QUEUENAME_VISE` | Queue for VISE biller |
| `LDR_MQ_QUEUENAME_VISN` | Queue for VISN biller |
| `LDR_MQ_QUEUENAME_VISW` | Queue for VISW biller |

---

## 5.6 Message Routing in `SendMsgToReprocessingMQ`

`SendMsgToReprocessingMQ` centralizes all outbound MQ routing decisions.

### Method Summary

| Method | JmsTemplate Used | Destination | When Called |
|---|---|---|---|
| `sendMsgToReprocessingMQ()` | `ldrJmstTemplateForRetry` | Reprocessing queue | Event failed, retries remaining |
| `sendMsgToCrossInstanceMQ()` | Per-instance template (RO/RT/BS/RS) | Target instance queue | Customer's home instance ≠ this instance |
| `sendMsgToBatchLocalMQ()` | `ldrJmsTemplate` | Batch local queue | Batch-type events needing batch node |

### Message Format on Queue

```
{xmlBody}@@@{timestamp}@@@{recycleCount}
```

Example for a first retry:
```
<Root>...</Root>@@@2024-01-15T10:30:00@@@1
```

---

## 5.7 EhCache Configuration

In-memory reference data caching reduces Oracle round-trips for frequently-read, rarely-changing reference tables.

**Config file:** `ehcache.xml`  
**Manager class:** `CacheService.java`

### Cache Definitions

| Cache Name | Max Entries | Overflow to Disk | TTL | Source Table | Lookup Axis |
|---|---|---|---|---|---|
| `TRANSMAP` | 10,000 | No | Eternal (refresh-controlled) | `REF_VIS_XLT_RATING_TRANS_MAP` | blIdValue; composite(blIdTyp, blIdValue, rbmIdTyp) |
| `PRODHIER` | 50,000 | No | Eternal | `REF_VIS_RATING_PROD_HIER` | tariffId |
| `TARIFFID` | 10,000 | No | Eternal | (tariff reference table) | tariffId |
| `REF_VISION_INSTANCE_LKUP` | 10 | No | Eternal | `REF_VISION_INSTANCE_LKUP` | billerId |

> `PRODHIER` is the largest cache at 50,000 entries because the product hierarchy table has many rows covering all rating configurations.

### Cache Lifecycle

```
Application startup
     │
     ▼
CacheService.refresh()
     ├─ Load all TRANSMAP records → EhCache "TRANSMAP"
     ├─ Load all PRODHIER records → EhCache "PRODHIER"
     ├─ Load all TARIFFID records → EhCache "TARIFFID"
     └─ Load all INSTANCE_LKUP records → EhCache "REF_VISION_INSTANCE_LKUP"
     
Runtime lookups (cache hits served from memory; no DB call)
     
Manual refresh available via API (triggers CacheService.refresh() again)
```

### CacheService Lookup Methods

| Method | Cache | Input | Returns |
|---|---|---|---|
| `findByBlIdValue(blIdValue)` | TRANSMAP | blIdValue string | `List<RefVisXltRatingTransMap>` |
| `findByCompositeKey(blIdTyp, blIdValue, rbmIdTyp)` | TRANSMAP | Three-part key | `RefVisXltRatingTransMap` |
| `findByTariffId(tariffId)` | PRODHIER | tariffId string | `RefVisRatingProdHier` |
| `findByBillerId(billerId)` | REF_VISION_INSTANCE_LKUP | billerId (e.g. VISN) | `RefVisionInstanceLkup` |
