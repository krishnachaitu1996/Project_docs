# Section 10 — Semi-Technical Overview

**Application:** JITR Loader 2.0  
**Navigation:** [← Error Handling](./09_error_handling.md) | [Index](./README.md)

**Audience:** This document is written for technical and semi-technical readers who need to understand what the JITR Loader does, why it exists, and how it works — without needing to read source code.

---

## 10.1 What Is the JITR Loader?

The **JITR Loader** (Just-In-Time Rating Loader) is a background processing service that keeps Verizon's billing and rating systems up to date whenever something changes on a subscriber's account.

When a customer activates a new phone, changes their plan, suspends service, or hundreds of other possible changes, that information needs to reach the billing databases quickly and accurately. The Loader is the system that does exactly that — it receives those change notifications and ensures the right information ends up in the right databases.

---

## 10.2 Why Does It Exist?

Verizon's customer provisioning systems (the systems that track what customers have and what changes to their accounts) are separate from the billing systems (the systems that calculate charges and produce bills). The JITR Loader is the bridge between them.

Without the Loader:
- Billing records would be out of sync with what was actually provisioned
- Customers could be billed incorrectly
- Rating lookups for calls and data usage would fail or produce wrong results

The Loader's job is to ensure that every account change is reflected in the billing databases as quickly as possible — ideally within seconds of the change occurring.

---

## 10.3 How It Fits in the Bigger Picture

```
Customer or CSR makes a change
          │
          ▼
   Provisioning System (Vision/CPF)
          │
          │  Sends XML notification message
          ▼
       IBM MQ Queue
    (message waiting room)
          │
          ▼
     JITR Loader ◄─────── This is us
          │
          │  Reads the message
          │  Validates it
          │  Updates databases
          │
          ├──────────────────► UBSR Oracle Database
          │                    (subscriber records)
          │
          ├──────────────────► RBM Oracle Databases (×8)
          │                    (billing attributes)
          │
          └──────────────────► Cassandra
                               (error message storage)
```

---

## 10.4 The Four Instances

The Loader does not run as a single server. It runs as **four separate copies**, each responsible for a specific group of customers:

| Instance | Customers It Handles |
|---|---|
| **BS** (VISB) | Billing System B customers |
| **RT** (VISN) | Rating North customers |
| **RS** (VISW) | Rating West customers |
| **RO** (VISE) | Rating East customers |

Each instance processes messages for its own group. If a message arrives at the wrong instance — because, for example, a routing change hasn't propagated yet — the instance automatically forwards the message to the correct instance. Nothing is lost.

---

## 10.5 What Kinds of Changes Does It Handle?

The Loader handles two broad categories of changes:

### 1. Subscriber Events (The "What Happened" Messages)

These describe a specific action that occurred on a line or account:

| Event | What It Means |
|---|---|
| **ACTIVATE** | A new phone line was turned on |
| **DEACTIVATE** | A phone line was turned off |
| **SUSPEND** | Service was temporarily paused |
| **RECONNECT** | Suspended service was restored |
| **REASSIGN** | A line was moved to a different account |
| **TRANSFER** | Ownership transferred to another person |
| **CHANGE_DATA_FEATURES** | Data plan features were added or removed |
| **CHANGE_SFO_FEATURES** | Special Feature Offers were changed |
| **CHANGE_BILL_DAY** | The billing cycle date changed |
| **REFRESH** | Account data was re-synchronized |

### 2. Data Population Events ("DataPop")

These are database change notifications — when a record in the provisioning database changes, a DataPop message tells the Loader which table changed, what the key was, and what the new values are.

There are around **60 types** of DataPop events covering everything from equipment transactions to service plan changes to address updates.

---

## 10.6 Step by Step: What Happens to a Message

Here is what happens every time a subscriber change occurs:

1. **Message arrives** on the IBM MQ queue. The message is in XML format and contains full details of the change.

2. **Message is parsed**. The Loader reads the XML and extracts key information: who the customer is, what account it affects, what phone number is involved, and what type of change occurred.

3. **Customer is looked up**. The Loader checks which of the four instances "owns" this customer. If it's the wrong instance, the message is forwarded.

4. **Lock is acquired**. Before touching any data, the Loader broadcasts a lock to all other running instances: "I'm working on customer 12345, nobody else touch them until I'm done." This prevents two instances from writing conflicting data simultaneously.

5. **Data is updated**. The appropriate database records are updated. Different types of changes update different tables — an activation updates subscriber tables, a plan change updates billing attribute tables, etc.

6. **Post-processing**. The result is recorded:
   - If successful → Success record written
   - If it failed but can be retried → Message sent back to the queue to try again
   - If it permanently failed → Error record saved for review

7. **Lock is released**. Other instances are free to process work for this customer now.

8. **Log entry written**. A structured log entry goes to the ELK (Elasticsearch) platform for monitoring and alerting.

---

## 10.7 What Happens When Something Goes Wrong?

The Loader is designed with retry logic at multiple levels:

### Automatic Retries (In-Flight)

If processing fails due to a transient issue (e.g., database temporarily busy), the message is placed back on the MQ queue with a retry counter incremented. The Loader will try again, up to a configurable maximum number of attempts.

### Reflow Scheduler (Background Recovery)

Every 20 minutes, a background job called the **Reflow Scheduler** runs. It:
- Looks for transactions that failed and couldn't be retried immediately
- Retrieves the original message from storage (Cassandra or Oracle)
- Re-submits them for processing

This means that even if a batch of transactions fails (e.g., database outage), they won't be permanently lost. They will be retried when the system recovers.

### Permanent Failure

If a message has been retried the maximum number of times and still fails, it is moved to the "unprocessed errors" table. Operations teams can review these, and the Reflow Scheduler can reprocess them once the underlying issue is resolved.

---

## 10.8 How Is Concurrent Correctness Guaranteed?

A key challenge: four Loader instances are all running simultaneously and all receiving messages. If two instances get different messages about the same customer at the same time, how do you prevent them from writing conflicting data?

**Answer: Distributed Locking via JGroups**

Before processing any event, each Loader instance acquires a cluster-wide lock using a component called **JGroups**. Lock names include the customer ID, account number, and phone number:

```
ACTIVATE_12345_67890_2015551234
```

Only the instance that holds this lock processes messages for that customer at that moment. All other instances that receive a conflicting message wait their turn. After the first instance finishes and releases the lock, the next message is processed.

This is similar to a "talking stick" in a meeting — only the person holding the stick can speak.

---

## 10.9 What Is Vision 2.0?

Vision 2.0 is Verizon's billing modernization initiative. The Loader was updated to support Vision 2.0 alongside the existing BAU (Business As Usual) billing flow.

When Vision 2.0 processing is enabled:
- The Loader uses a newer set of billing tables and processing logic
- An extra set of ~40 "BillingVision" event handlers writes to Vision 2.0 billing tables
- The customer is checked against a lookup table to determine if they are a Vision 2.0 customer

Feature flags (toggles) control which customers and which event types route through the Vision 2.0 path versus the legacy path. This allows a gradual migration with the ability to roll back instantly.

---

## 10.10 What Is ILB?

**ILB** (Intelligent Lookback) is a sub-module within the Loader that runs after CPF events are processed.

When a plan change occurs (adding or removing a Special Feature Offer, service plan, or special offer), there may be **overlapping records** — multiple billing records that conflict with each other because one plan started before another ended. ILB detects these overlaps and writes correction records to the RBM billing databases.

Think of it like a bookkeeper who checks the ledger after each transaction to ensure no entries contradict each other.

---

## 10.11 Technology Summary (Plain Language)

| Component | What It Is | Why It's There |
|---|---|---|
| **IBM MQ** | A message queuing system | Decouples Vision from the Loader; messages don't get lost even if the Loader is momentarily down |
| **Oracle Database (UBSR)** | A relational database | Stores subscriber records and processing history |
| **Oracle Database (RBM ×8)** | Eight relational database shards | Stores billing attributes; spread across 8 servers for performance |
| **Cassandra** | A distributed NoSQL database | Stores original message XML for reliable retry/reflow |
| **JGroups** | A cluster coordination library | Enables distributed locking across multiple Loader instances |
| **EhCache** | An in-memory cache | Avoids repeated slow database lookups for rarely-changing reference data |
| **JAXB** | An XML parsing library | Converts the incoming XML messages into Java objects the code can work with |
| **JIS** | An internal REST API | Provides reference data lookups (plans, products, pricing) |
| **AWS AppConfig** | Amazon feature flag service | Allows toggling features on/off instantly without redeployment |
| **ELK Stack** | Elasticsearch/Logstash/Kibana | Collects structured logs for monitoring, alerting, and debugging |

---

## 10.12 Operational Quick Reference

| I want to... | How |
|---|---|
| Check if the Loader is running | `GET /loader/versioncheck` |
| See current feature flag status | `GET /loader/versioncheck` |
| Check application version | `GET /loader/api/versionCheck/` |
| Investigate a failed transaction | Search `JITR_LDR_VB_ERRORS` or `JITR_LDR_VB_UNPROCESSED_ERRORS` by customer ID or transaction ID |
| Force-release a stuck distributed lock | `POST /loader/checkLock/unlock/{lockName}` |
| Check if a specific lock is held | `POST /loader/checkLock/lock/{lockName}` (returns current state) |
| Re-process failed transactions | Done automatically every 20 minutes; or trigger the Reflow Scheduler |
| Enable/disable Vision 2.0 | Set `visionFlagEnable=ON/OFF` in properties (requires restart) or toggle via AWS AppConfig |
