# Section 9 — Error Handling & Observability

**Application:** JITR Loader 2.0  
**Navigation:** [← Environment Config](./08_environment_config.md) | [Index](./README.md) | [Semi-Technical Guide →](./10_semi_technical_guide.md)

---

## 9.1 Error Code System

**Class:** `LoaderErrorMap`  
**Source:** Static `HashMap<String, String>` populated from the `REF_ERROR_CODES` Oracle table.

The Loader uses a two-layer error code system:
1. A **symbolic name** (e.g., `E_LOADER_CPF_ORDER`) used in code
2. A **short code** (e.g., `LDR_D_E7000`) stored in the database and logs

```java
// Usage in code
edrCpfMessage.getUbsrErrors().add("LDR_D_E7039");  // direct short code
// or via map
edrCpfMessage.getUbsrErrors().add(
    LoaderErrorMap.getLoaderErrorMap().get("E_LOADER_CPF_ORDER")
);
```

---

## 9.2 Error Code Reference (170+ codes)

### LDR_D_E70xx — Core / CPF Processing

| Code | Symbolic Name | Description |
|---|---|---|
| `LDR_D_E7000` | `E_LOADER_CPF_ORDER` | Generic CPF order processing error |
| `LDR_D_E7039` | (direct) | Unsupported or unknown event type received |
| `LDR_D_E7121` | (direct) | Unhandled exception during `validateAndProcessEvent()` |
| `LDR_D_E7130` | (direct) | Persistence exception during `validateAndProcessEvent()` |
| `LDR_D_E7194` | (direct) | ROSIOT/Kafka duplicate — event filtered (DB_USERID=ROSIOT) |

### LDR_D_E7001–E7038 — Field Validation

Codes in this range cover missing or invalid mandatory fields:
- Missing customer ID
- Missing account number
- Missing MDN
- Invalid field values
- JAXB unmarshalling failures

### LDR_D_E7040–E7099 — Event-Specific Errors

Each CPF event type has a block of reserved error codes for its specific failure scenarios:
- ACTIVATE failures (e.g., duplicate activation, invalid line of service)
- DEACTIVATE failures
- CHANGE errors
- TRANSFER contract term errors
- SUSPEND / RECONNECT failures
- REFRESH failures

### LDR_D_E7100–E7130 — System / Infrastructure Errors

| Code | Description |
|---|---|
| `LDR_D_E7100`–`LDR_D_E7120` | JIS REST call failures |
| `LDR_D_E7121` | Unexpected exception during event processing |
| `LDR_D_E7130` | Database persistence exception |

### LDR_D_E7131–E7170 — DataPop-Specific Errors

DataPop processing failures with specifics per datapop type:
- Missing required datapop fields
- Invalid TABLE_NAME values
- DB write failures for specific subscriber tables

### LDR_D_E7171–E7200+ — BillingVision Errors

Vision 2.0 billing processing failures:
- ECPD profile not found
- Billing system write errors
- ILB processing errors
- Vision 2.0 customer lookup failures

---

## 9.3 Error Outcome Routing

When processing fails, the outcome is determined by the combination of error flags on `EDRCPFMessage`:

```
Processing outcome determination:

isTransactionSuccessful = true?
  └─ YES → Write JITR_LDR_VB_SUCCESS
            Log: SUCCESS

isBillingTransactionSuccessful = true?
  └─ YES → Write JITR_LDR_VB_SUCCESS (billing path)
            Log: SUCCESS

isGeneralError = true?
  └─ YES → Write JITR_LDR_VB_UNPROCESSED_ERRORS
            ELK status: UNPROCESSED
            (No retry — parse/routing/unknown event)

isReprocessRequired = true?
  ├─ recycleCount < MAX_RETRIES?
  │     └─ YES → SendMsgToReprocessingMQ()
  │               ELK status: REPROCESS
  └─ recycleCount >= MAX_RETRIES?
        └─ YES → Write JITR_LDR_VB_UNPROCESSED_ERRORS
                  ELK status: ERROR

isODRRequired = true (any of above)?
  └─ Write JITR_REF_RECON_ON_DEMAND (in addition to above)
```

---

## 9.4 Database Error Tables

### `JITR_LDR_VB_SUCCESS`
Written when `isTransactionSuccessful=true`. Columns include transaction ID, customer ID, account number, MDN, event type, processing timestamp, instance.

### `JITR_LDR_VB_ERRORS`
Written for retryable errors. Includes error codes, retry count, original XML reference.

### `JITR_LDR_VB_UNPROCESSED_ERRORS`
Terminal error table. Written when:
- `isGeneralError=true`
- Retry count exhausted
- Mapped to `JitrLdrUnprocessedErrors` entity

### `JITR_REF_RECON_ON_DEMAND`
On-Demand Reconciliation table. Written when `isODRRequired=true`. Flags that the customer's billing data needs manual or automated reconciliation. Mapped to `JitrRefReconOnDemand` entity.

---

## 9.5 Logging Architecture

### Log4j2 + LMAX Disruptor

The Loader uses Log4j2's **asynchronous logging** via the LMAX Disruptor library. All file appenders are wrapped in async appenders to prevent disk I/O from blocking event processing threads.

### Appender Summary

| Appender | Type | Content |
|---|---|---|
| `rollingLogger` | Rolling file (date + size) | All application logs — primary operational log |
| `archive` | Rolling file | Secondary archive copy |
| `ignorequeue` | Rolling file | Suppressed/filtered messages (e.g., ROSIOT events) |
| `ELK` | Async file → ELK pipeline | Structured JSON logs for Kibana dashboards |
| `BCP` | Rolling file | Bill cycle processing events only |
| `ILB` | Rolling file | ILB module events only |
| `SQL` | Rolling file | SQL query execution logs (Hibernate SQL) |
| `Hikari` | Rolling file | HikariCP connection pool diagnostics |

### Log Levels in Practice

| Logger | Typical Level | Notes |
|---|---|---|
| `com.vzw.jitrLdr` | INFO | Main application — INFO for key decisions, DEBUG for field values |
| `com.vzw.ilb` | INFO | ILB sub-module — routed to ILB appender |
| `org.hibernate.SQL` | DEBUG | SQL log — routed to SQL appender only |
| `com.zaxxer.hikari` | DEBUG | HikariCP — routed to Hikari appender only |

---

## 9.6 ELK Structured Logging

**Class:** `ELKLogger.java`

Each processed event generates a structured ELK log entry. The format:

```
{MessageAction} | {EventStartTime} | {EventEndTime} | {DurationMs} | {ReprocessCount} | {Status} | {ErrorCode1} | {ErrorCode2} | {ErrorCode3} | {ODRInd} | {CustBillerId}
```

### ELK Log Fields

| Field | Source | Example |
|---|---|---|
| `serviceName` | Constant | `jitrLoader` |
| `customer` | `edrCpfMessage.customerIdNo` | `12345` |
| `account` | `edrCpfMessage.acctNo` | `67890` |
| `mdn` | `edrCpfMessage.mdn` | `2015551234` |
| `correlationId` | Message correlation ID | UUID |
| `transactionId` | `edrCpfMessage.cpfTranID` | `CPF_TXN_123` |
| `processTime` | `endTime - startTime` | `245` (ms) |
| `statusCode` | Outcome | `SUCCESS` / `REPROCESS` / `ERROR` / `UNPROCESSED` |
| `statusMessage` | Same as statusCode | `SUCCESS` |
| `errorCode` | First error from `ubsrErrors` | `LDR_D_E7121` |
| `inputFileName` | Event type or datapop type | `ACTIVATE` / `CUSTOMER` |
| `instance` | `ubInstance` | `VISN` |
| `method` | `msgAction` | `INSERT` |

### ELK Status Values

| Status | Meaning |
|---|---|
| `SUCCESS` | Transaction fully processed |
| `REPROCESS` | Sent back to MQ for retry |
| `ERROR` | Retries exhausted, written to unprocessed errors |
| `UNPROCESSED` | General error, not retryable |

---

## 9.7 Reprocessing & Reflow Error Recovery

### Retry Count Override Map

`EventProcessHelper.reprocessRetryList` maps specific datapop types to custom maximum retry counts:

```properties
# Example: allow more retries for device provisioning
LDR_REPROCESS_RETRY_MAP=CUST_DVC_PROV_INFO:5,ACCOUNT:3
```

Resolution order:
1. Is `datapopType` in `reprocessRetryList`? → Use that count
2. Else → Use global `reprocessRetryCount`

### Reflow Error Map

`reprocessRetryList` in `EventProcessHelper` (loaded from `LDR_REFLOW_ERROR_MAP` property) also drives which error codes trigger reprocessing vs. immediate failure.

---

## 9.8 Exception Handling in `LoaderEventProcessor`

Two catch blocks in the processing pipeline:

```java
// Catch 1: Known LoaderExceptions (expected application exceptions)
} catch (LoaderException pexp) {
    edrCpfMessage.setODRRequired(Boolean.TRUE);
    edrCpfMessage.getUbsrErrors().add("LDR_D_E7130");
    return lockName;
}

// Catch 2: All other exceptions (unexpected / runtime)
} catch (Exception ex) {
    edrCpfMessage.setODRRequired(Boolean.TRUE);
    edrCpfMessage.getUbsrErrors().add("LDR_D_E7121");
    return lockName;
}
```

Both set `isODRRequired=true`, ensuring a reconciliation record is written for any unhandled failure.

---

## 9.9 Observability Summary

| Signal Type | Technology | Output |
|---|---|---|
| Application logs | Log4j2 + LMAX Disruptor | Rolling files per domain (main, ILB, BCP, SQL, Hikari) |
| Structured events | ELK appender → Elasticsearch | Kibana dashboards for transaction metrics |
| Health status | Spring MVC REST | `GET /versioncheck` |
| JMX metrics | JGroups MBeans (registered in `DistributedService`) | Cluster membership, lock state |
| Email alerts | JavaMail | Critical system-level failures |
| Per-transaction MDC | Log4j2 ThreadContext | EventName, TransId, CustomerId, AccountNumber, mdn in every log line |
