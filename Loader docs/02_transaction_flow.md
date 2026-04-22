# Section 2 — Transaction Flow Analysis

**Application:** JITR Loader 2.0  
**Navigation:** [← Architecture](./01_system_architecture.md) | [Index](./README.md) | [Event Types →](./03_event_type_catalog.md)

---

## 2.1 End-to-End Message Processing

The following is the complete flow from message arrival to final database write for every event processed by the Loader.

```
IBM MQ Queue
     │
     ▼
MQListener.onMessage(TextMessage)
     │   @JmsListener on ${LDR_LOCAL_MQ_QUEUENAME}
     │   Container factory: dtlJmsListenerContainerFactory
     │
     ▼
LoaderEventProcessor.processEvent(String xmlPayload)
     │
     ├─ Step 1: Parse raw message
     │     xmlPayload.split("@@@")
     │     → segment[0] = XML body
     │     → segment[1] = message timestamp
     │     → segment[2] = recycle count (retry number)
     │
     ├─ Step 2: JAXB Unmarshal
     │     JAXBHelper unmarshals XML string → Root object
     │     Root is defined by InputSchema.xsd
     │
     ├─ Step 3: Event Identification
     │     EventProcessHelper.getEDRCPFMessage(root)
     │     → instanceof checks on Root's child element
     │     → Populates EDRCPFMessage with:
     │           customerId, acctNo, mdn, eventType,
     │           datapopType (if DATAPOP), trTime, lineOfServiceP1/P2
     │
     ├─ Step 4: Vision 2.0 Customer Check
     │     CacheService.findByBillerId(billerId)
     │     → Looks up REF_VISION_INSTANCE_LKUP cache
     │     → Sets edrCpfMessage.isVisionCustomer flag
     │     → Flag may be overridden by V2DT39120="Y" (force all as Vision 2.0)
     │
     ├─ Step 5: Routing & Timezone Resolution
     │     EventProcessHelper.getTimeZoneAndRoutingInfo(edrCpfMessage)
     │     → Queries MTDT_JITR_ROUTING for customer's home instance
     │     → If customer NOT on this instance → SendMsgToCrossInstanceMQ
     │     → Sets domainId, domainGroupId, billerId, timezone, ubInstance
     │
     ├─ Step 6: Event Handler Lookup
     │     ILoaderEventService service = factory.getEvent(eventName, datapopType)
     │     → Returns the specific handler bean (e.g., ActivateEvent, CustomerEvent)
     │     → Returns null if event type is unsupported → error LDR_D_E7039
     │
     ├─ Step 7: Lock Name Construction
     │     getLockName(edrCpfMessage)
     │     → Builds from: eventType + customerId + accountNo + mdn [+ extras]
     │     → Extras: sfoId (for CHANGE_SFO_FEATURES), deviceId (for CUST_DVC_PROV_INFO)
     │     → Falls back to NOCUST_{tranId} if no customer ID present
     │
     ├─ Step 8: Distributed Lock Acquisition
     │     distributedService.acquireLock(lockName)
     │     → tryLock() with 1000ms timeout, retries up to max attempts
     │     → Blocks until lock is acquired
     │
     ├─ Step 9: Pre-Processing Check
     │     EventProcessHelper.getTimeZoneAndRoutingInfo(edrCpfMessage)
     │     → Validates UBSR errors list is empty before processing
     │     → Checks ORDataPopList membership (Vision 2.0 check for OR datapops)
     │
     ├─ Step 10: Event Processing
     │     service.validateAndProcessEvent(edrCpfMessage)
     │     → Each handler validates its specific fields
     │     → Writes to UBSR Oracle (subscriber tables)
     │     → Writes to RBM Oracle via MultitenantDataSource (billing tables)
     │     → Returns updated EDRCPFMessage with results
     │
     ├─ Step 11: Post-Processing (handlePostProcess)
     │     ├─ Success path:
     │     │     → Write to JITR_LDR_VB_SUCCESS
     │     ├─ Error path — reprocess:
     │     │     → recycleCount < max retries
     │     │     → SendMsgToReprocessingMQ.sendMsgToReprocessingMQ()
     │     ├─ Error path — unprocessed:
     │     │     → recycleCount >= max retries
     │     │     → Write to JITR_LDR_VB_UNPROCESSED_ERRORS
     │     └─ ODR path:
     │           → edrCpfMessage.isODRRequired = true
     │           → Write to JITR_REF_RECON_ON_DEMAND
     │
     ├─ Step 12: ILB Processing (CPF events only)
     │     ILBInterface.invokeIlb(custIdNo, acctNo, mdn)
     │     → Build overlap records
     │     → Process SFO overlaps
     │     → SPLAN to PayGo transitions
     │     → Global SFO to SPOLN
     │
     ├─ Step 13: Billing Vision Processing
     │     (if Vision 2.0 customer and vision events enabled)
     │     BillingEventProcessHelper processes billing-specific writes
     │
     └─ Step 14: ELK Logging & Lock Release
           distributedService.releaseLock(lockName)
           ELK log entry with:
             action | startTime | endTime | durationMs | retryCount | status | errorCodes
```

---

## 2.2 Message Format

Inbound messages arrive as plain text with three `@@@`-delimited segments:

```
<Root xmlns="urn:JITS:...">
  <ACTIVATE>
    <TRKEY>
      <TRFULFILLMENTTIME>2024-01-15T10:30:00</TRFULFILLMENTTIME>
      <TRMDN>2015551234</TRMDN>
    </TRKEY>
    <CUSTOMERID>12345</CUSTOMERID>
    <ACCOUNTNUMBER>67890</ACCOUNTNUMBER>
    ...
  </ACTIVATE>
</Root>@@@2024-01-15T10:30:00@@@0
```

| Segment | Content |
|---|---|
| `[0]` | XML body conforming to `InputSchema.xsd` |
| `[1]` | Timestamp of original message creation |
| `[2]` | Recycle count — `0` on first attempt; incremented on each reprocess |

When reprocessing, the Loader reconstructs this format: `xml@@@timestamp@@@recycleCount`.

---

## 2.3 EDRCPFMessage — The Processing Context Object

`EDRCPFMessage` is the central data transfer object that carries all information about an event through the entire processing pipeline. Key fields:

| Field | Type | Description |
|---|---|---|
| `eventType` | String | CPF event name (ACTIVATE, REASSIGN, etc.) |
| `datapopType` | String | DataPop table name (e.g., CUST_DVC_PROV_INFO) |
| `cpfTranID` | String | Unique CPF transaction identifier |
| `customerIdNo` | String | Customer identifier |
| `acctNo` | String | Account number |
| `mdn` | String | Mobile directory number |
| `lineOfServiceP1/P2` | String | Line-of-service identifiers |
| `trTime` | String | Transaction fulfillment time |
| `billerId` | String | Biller/instance ID (VISB/VISN/VISW/VISE) |
| `domainId` | String | Customer domain ID |
| `domainGroupId` | String | RBM shard routing key |
| `ubInstance` | String | UB instance from vision lookup |
| `custJiTRInstance` | String | Customer's home JITR instance |
| `isVisionCustomer` | Boolean | Whether customer is Vision 2.0 |
| `isJiTRCust` | Boolean | Whether customer is in MTDT_JITR_ROUTING |
| `custOnHomeJitrInstance` | Boolean | Whether this instance is the customer's home |
| `reprocessCount` | int | Number of retry attempts so far |
| `isReprocessRequired` | Boolean | Event should be retried |
| `isGeneralError` | Boolean | Unrecoverable/parse error |
| `isODRRequired` | Boolean | On-demand reconciliation needed |
| `isTransactionSuccessful` | Boolean | Processing succeeded |
| `ubsrErrors` | List\<String\> | List of error codes from UBSR processing |
| `originalData` | String | Raw original XML payload |
| Various message lists | List\<XxxType\> | Typed JAXB payload per event type |

---

## 2.4 Reflow (Automatic Retry) Pipeline

Failed transactions that cannot be immediately retried via the MQ are stored and replayed by the Reflow Scheduler.

```
Application runs
     │
     Every 20 minutes (cron: 0 */20 * * * *)
     │
     ▼
ReflowScheduler.reflowEvents()
     │
     ├─ Check LDR_REFLOW_SCHEDULER_ENABLE = ON
     │
     ├─ Query JITR_LDR_TRX_REFLOW table
     │     → Up to LDR_REFLOW_TXN_THRESHOLD records (default: 250)
     │     → Ordered by priority
     │
     ├─ For each failed transaction:
     │     ├─ Retrieve original XML:
     │     │     → Cassandra (preferred, if VISION_2_0_CASSANDRA_ENABLE = ON)
     │     │     → Fallback: JITR_LDR_TRX Oracle table
     │     │
     │     ├─ Duplicate detection via ReflowService
     │     │
     │     └─ Re-submit to LoaderEventProcessor.processEvent()
     │
     └─ Concurrent execution via ExecutorService thread pool
           (configurable pool size)
```

**Per-datapop retry count overrides:**  
`EventProcessHelper.reprocessRetryList` is a `HashMap<String, String>` mapping specific datapop type names to custom max retry counts. When a datapop type is found in this map, its custom limit is used instead of the global `reprocessRetryCount` property.

---

## 2.5 Cross-Instance Routing

When a message arrives at an instance that does not own the customer:

```
Message arrives at RT instance (VISN)
Customer's home instance = BS (VISB)
     │
     ▼
EventProcessHelper.getTimeZoneAndRoutingInfo()
     → MTDT_JITR_ROUTING lookup → custJiTRInstance = "BS"
     → JITR_INSTANCE (this node) = "RT"
     → custOnHomeJitrInstance = false
     │
     ▼
SendMsgToReprocessingMQ.sendMsgToCrossInstanceMQ()
     → ldrJmsTemplateBS (the BS instance JmsTemplate)
     → Message placed on BS queue
     → Processing completes on BS instance
```

---

## 2.6 Reprocessing MQ Routing Decision

```
Event fails with error
     │
     ├─ reprocessCount < MAX_RETRIES?
     │     ├─ YES → SendMsgToReprocessingMQ.sendMsgToReprocessingMQ()
     │     │         → Message format: xml@@@timestamp@@@(reprocessCount+1)
     │     │         → Placed on LDR_REPROCESSING_MQ_QUEUENAME
     │     │         → Picked up again by MQListener
     │     │
     │     └─ NO → Write JITR_LDR_VB_UNPROCESSED_ERRORS
     │               → Available for ReflowScheduler (periodic retry)
     │
     └─ generalError = true?
           → Skip retry queue
           → Write JITR_LDR_VB_UNPROCESSED_ERRORS immediately
```

---

## 2.7 ROSIOT (IOT) Event Filtering

The Loader receives some subscriber events from both Vision (via MQ) and the IOT Kafka pipeline. To prevent duplicate processing, when `ROSIOTFLAG=true` any event containing `DB_USERID=ROSIOT` is rejected at parse time:

- Sets `generalError = true`
- Adds error code `LDR_D_E7194`
- Applies to: **Suspend**, **Reconnect**, and **DataPop** events

No processing occurs; the event is recorded as a filtered/unprocessed transaction.

---

## 2.8 Batch Listener Flow

`MQBatchListener` operates on the `${LDR_BATCHLOCAL_LISTENER_MQ}` queue and follows an identical flow to `MQListener`. The `LDR_BATCH_NODE=Y/N` property governs whether this instance processes batch-routed events directly or forwards them.
