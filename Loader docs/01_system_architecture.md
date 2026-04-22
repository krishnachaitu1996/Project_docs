# Section 1 — System Architecture Document

**Application:** JITR Loader 2.0  
**Navigation:** [← Index](./README.md) | [Transaction Flow →](./02_transaction_flow.md)

---

## 1.1 High-Level Overview

The **JITR Loader** (Just-In-Time Rating Loader) is a Spring Boot application that sits between Verizon's Customer Provisioning Feed (CPF) system and the downstream billing/rating databases. Its core responsibilities are:

1. **Receive** XML messages from IBM MQ queues describing subscriber lifecycle events (activations, deactivations, feature changes, etc.) and bulk data population ("datapop") updates.
2. **Parse, validate, and transform** those messages into database writes across two Oracle database tiers (UBSR and RBM) and optionally a Cassandra cluster.
3. **Coordinate** across multiple running instances using JGroups-based distributed locking so no two instances process the same customer concurrently.

```
┌────────────────┐       ┌──────────────────────────────────────────────────────┐
│  Vision / CPF  │──MQ──▶│                 JITR Loader Instance                 │
│   Upstream     │       │                                                      │
└────────────────┘       │  MQListener ──▶ LoaderEventProcessor ──▶ Factory     │
                         │       │              │                    │           │
                         │       │        JAXB Unmarshal        EventHandlers   │
                         │       │              │              (CPF / DataPop /  │
                         │       │        Distributed Lock      BillingVision)  │
                         │       │         (JGroups)                │           │
                         │       │              │           ┌───────┴───────┐   │
                         │       │              │           │               │   │
                         │       │              ▼           ▼               ▼   │
                         │       │         ┌────────┐  ┌────────┐  ┌──────────┐│
                         │       │         │UBSR DB │  │RBM DBs │  │Cassandra ││
                         │       │         │(Oracle)│  │(Oracle)│  │ Cluster  ││
                         │       │         │1 node  │  │8 nodes │  └──────────┘│
                         │       │         └────────┘  └────────┘              │
                         │       │                                              │
                         │       └──▶ JIS (REST) / DVS / DMD / ILB             │
                         └──────────────────────────────────────────────────────┘
```

---

## 1.2 Spring Boot Bootstrap

**Entry Point:** `LoaderApplication.java`  
**Package:** `com.vzw.jitrLdr`

| Annotation | Purpose |
|---|---|
| `@SpringBootApplication` | Auto-configuration (excludes `DataSourceAutoConfiguration` — manual DataSource setup) |
| `@EnableJms` | Activates JMS listener containers for IBM MQ |
| `@EnableCaching` | Turns on EhCache-backed method caching |
| `@EnableScheduling` | Enables `@Scheduled` methods (reflow, ILB, rerate batch) |
| `@ComponentScan` | Scans `com.vzw.jitrLdr` and `com.vzw.ilb` packages |
| `@EntityScan` | JPA entity discovery in `dao.rbm.dto` and `dao.ubsr.dto` |

Configuration is loaded from `application-ldr.properties` with encrypted secrets managed by **Jasypt** (`PBEWithMD5AndDES`). Spring Cloud Config bootstrap is defined in `bootstrap.properties` but is currently disabled (`spring.cloud.config.enabled=false`).

---

## 1.3 Multi-Instance Topology

The Loader is deployed as **four named instances**, one per Vision billing region:

| Instance Code | Biller ID | Vision Instance | Description |
|---|---|---|---|
| BS | VISB | VISB | Billing System B |
| RT | VISN | VISN | Rating System N |
| RS | VISW | VISW | Rating System W |
| RO | VISE | VISE | Rating System E |

Each instance has its own set of **MQ connection factories and JmsTemplates** defined in `JmsConfiguration.java`. When a message arrives for a customer that belongs to a *different* instance, the Loader routes it to that instance's MQ queue via `SendMsgToReprocessingMQ.sendMsgToCrossInstanceMQ()`.

A special **prodParallel** queue exists for B2B parallel billing.

The mapping from billing system to Vision instance is maintained in the `INSTANCE_MAP` in `LoaderEventProcessor`:

```java
// Biller ID → Vision Instance mapping
BS → VISB
RT → VISN
RS → VISW
RO → VISE
```

---

## 1.4 Distributed Locking (JGroups)

**Class:** `DistributedService.java`  
**Cluster Config:** `bpmn.xml` — despite the filename, this is a **JGroups UDP protocol stack**, not a BPMN workflow file.

### Protocol Stack

```
UDP → PING → MERGE3 → FD_SOCK → FD → VERIFY_SUSPECT
  → NAKACK → UNICAST → STABLE → GMS → UFC → MFC
  → FRAG2 → STATE_TRANSFER → CENTRAL_LOCK
```

The `CENTRAL_LOCK` protocol at the end of the stack is what enables distributed mutual exclusion across all running Loader instances on the same network.

### Lock Naming Convention

Lock names are built from event metadata so that only events affecting the same business entity contend with each other:

```
{EventType}_{CustomerId}_{AccountNumber}[_{MDN}[_{SFO_ID | DeviceId}]]
```

**Examples:**

| Event | Lock Name Pattern | Example |
|---|---|---|
| CPF event (standard) | `ACTIVATE_{custId}_{acctNo}_{mdn}` | `ACTIVATE_12345_67890_2015551234` |
| DataPop — device provisioning | `CUST_DVC_PROV_INFO_{custId}_{acctNo}_{mdn}_{deviceId}` | `CUST_DVC_PROV_INFO_12345_67890_2015551234_IMEI123` |
| Change SFO Features | `CHANGE_SFO_FEATURES_{custId}_{acctNo}_{mdn}_{sfoId}` | `CHANGE_SFO_FEATURES_12345_67890_2015551234_500` |
| No customer ID | `NOCUST_{cpfTranId}` | `NOCUST_TXN123456` |

The lock service calls `tryLock()` with a 1000ms timeout and retries up to a configurable number of attempts. Named locks like `"ILBProcessor"` are also used for batch operations (rerate processing).

MDN-level locking can be enabled or disabled via the `MDN_LEVEL_LOCK` flag in `application-ldr.properties`.

---

## 1.5 Sub-Modules

| Module | Package | Purpose |
|---|---|---|
| **ILB** (Intelligent Lookback) | `com.vzw.ilb` | Detects SFO/SPLAN/SPO overlaps when subscriber plans change; writes correction records to RBM databases |
| **BillingVision** | `loader.events.datapop.billingVision`, `loader.events.cpf.billingVision` | 40+ event handlers that write to Vision 2.0 billing tables |
| **Rerate** | `rerateProcess` | Batch rerate processing with distributed lock `"ILBProcessor"`; processes `SubRerateRequest` records |
| **BillCycle RBA** | `billCycleProcess` | Bill cycle change processing; writes output files to configured paths |
| **Cassandra** | `cassandra` | Stores original XML message payloads for reflow error recovery; toggled via `VISION_2_0_CASSANDRA_ENABLE` |

### ILB Sub-Module Detail

**Entry Point:** `ILBInterface.invokeIlb(custIdNo, acctNo, mdn)`

Processing steps:
1. Build overlap records — identify where current SFO/SPLAN/SPO assignments overlap
2. Process SFO overlaps
3. Process SPLAN to PayGo transitions
4. Process global SFO to SPOLN

Returns a `Set<SubXltBlRbm>` — the cross-reference records between UBSR and RBM billing.

---

## 1.6 Technology Stack

| Layer | Technology | Version |
|---|---|---|
| Runtime | Java | 17 |
| Framework | Spring Boot | 3.1.5 |
| Messaging | IBM MQ (com.ibm.mq.allclient) | 9.1.0.0 |
| JMS Integration | Spring JMS | (Spring Boot managed) |
| Primary Database | Oracle (ojdbc8) | 18.3 |
| ORM | Hibernate | 5.6.15 |
| JPA | Spring Data JPA | 2.7.18 |
| Cassandra | DataStax Java Driver | 3.7.2 |
| Cassandra Spring | Spring Data Cassandra | 2.1.2 |
| Distributed Locking | JGroups | 3.6.10 |
| Caching | EhCache | 2.10.9.2 |
| Logging | Log4j2 | 2.19.0 |
| Async Logging | LMAX Disruptor | (Log4j2 managed) |
| Encryption | Jasypt | (PBEWithMD5AndDES) |
| Config Server | Spring Cloud Config | (bootstrap) |
| Feature Flags | com.vzw.core:feature-flags-java | 2.0-SNAPSHOT |
| XML Parsing | JAXB | (Generated from XSD) |
| HTTP Client | Apache HttpClient | 4.5.14 |
| Build | Maven | — |
