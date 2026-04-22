# Section 8 — Environment & Configuration Reference

**Application:** JITR Loader 2.0  
**Navigation:** [← Feature Flags](./07_feature_flags.md) | [Index](./README.md) | [Error Handling →](./09_error_handling.md)

---

## 8.1 Configuration File Inventory

| File | Location | Purpose |
|---|---|---|
| `application-ldr.properties` | `src/main/resources/` | Primary application configuration (~600 lines) |
| `bootstrap.properties` | `src/main/resources/` | Spring Cloud Config bootstrap; Jasypt password |
| `ehcache.xml` | `src/main/resources/` | EhCache cache definitions |
| `bpmn.xml` | `src/main/resources/` | JGroups UDP protocol stack configuration |
| `log4j2.xml` | `src/main/resources/` | Log4j2 appenders, loggers, async configuration |
| `InputSchema.xsd` | `src/main/resources/` | XML schema for inbound CPF/DataPop messages |
| `InputSchema.xjb` | `src/main/resources/` | JAXB binding customization for XSD |
| `messages.properties` | `src/main/resources/` | Message resource bundle |

> **Runtime note:** Files in `PROPS/` are environment-specific overrides used during deployment. The `src/main/resources/` files are defaults packaged into the JAR.

---

## 8.2 Server Configuration

```properties
# HTTP server
server.port=14175
server.servlet.context-path=/loader

# Application identity
spring.application.name=jitrLoader-app
```

---

## 8.3 MQ Connection Configuration

Each of the five queue manager connections (own + 4 siblings) follows this property pattern:

```properties
# Primary / own instance
LDR_MQ_HOSTNAME=<host>
LDR_MQ_PORT=<port>
LDR_MQ_QMANAGER=<queue manager name>
LDR_MQ_CHANNEL=<MQ channel>
LDR_LOCAL_MQ_QUEUENAME=UBD31.VISN.LOCAL.RCV_PUB.QL
LDR_REPROCESSING_MQ_QUEUENAME=<retry queue name>
LDR_BATCHLOCAL_LISTENER_MQ=<batch queue name>
LDR_BATCHLOCAL_MQ_QUEUENAME=<batch local queue>

# RO instance (VISE)
LDR_MQ_HOSTNAME_RO=<host>
LDR_MQ_PORT_RO=<port>
LDR_MQ_QMANAGER_RO=<QM name>
LDR_MQ_CHANNEL_RO=<channel>
LDR_MQ_QUEUENAME_RO=<queue name>

# RT instance (VISN) — similar pattern with _RT suffix
# BS instance (VISB) — similar pattern with _BS suffix
# RS instance (VISW) — similar pattern with _RS suffix
# Prod parallel — similar pattern with _PRODPARALLEL suffix
```

Biller-specific outbound queues:
```properties
LDR_MQ_QUEUENAME_VISB=<VISB queue>
LDR_MQ_QUEUENAME_VISE=<VISE queue>
LDR_MQ_QUEUENAME_VISN=<VISN queue>
LDR_MQ_QUEUENAME_VISW=<VISW queue>
```

---

## 8.4 Oracle Database Configuration

### UBSR (Single Node)
```properties
spring.datasource.url=jdbc:oracle:thin:@//<host>:<port>/<service_name>
spring.datasource.username=ENC(<encrypted>)
spring.datasource.password=ENC(<encrypted>)
spring.datasource.driver-class-name=oracle.jdbc.OracleDriver

# HikariCP
spring.datasource.hikari.maximum-pool-size=100
spring.datasource.hikari.minimum-idle=20
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
```

### RBM (8 Nodes)
```properties
rbm.datasource.node1.url=jdbc:oracle:thin:@//<host1>:<port>/<service>
rbm.datasource.node1.username=ENC(<encrypted>)
rbm.datasource.node1.password=ENC(<encrypted>)

rbm.datasource.node2.url=jdbc:oracle:thin:@//<host2>:<port>/<service>
...
rbm.datasource.node8.url=jdbc:oracle:thin:@//<host8>:<port>/<service>
```

Each node has identical HikariCP settings and its own isolated connection pool.

---

## 8.5 Cassandra Configuration

```properties
cassandra.contactPoints=<host1,host2,host3>
cassandra.port=9042
cassandra.keyspace=<keyspace_name>
cassandra.username=ENC(<encrypted>)
cassandra.password=ENC(<encrypted>)

# Toggle
VISION_2_0_CASSANDRA_ENABLE=ON
```

---

## 8.6 External Service Configuration

```properties
# JIS — JITR Integration Service
jis.host=<JIS hostname>
jis.port=<JIS port>

# DVS — Data Validation Service
dvs.url=<DVS URL>
```

---

## 8.7 Scheduler Configuration

```properties
# Reflow scheduler
LDR_REFLOW_SCHEDULER_ENABLE=ON
LDR_REFLOW_SCHEDULER_CRON=0 */20 * * * *
LDR_REFLOW_TXN_THRESHOLD=250

# ILB scheduler
LDR_ILB_SCHEDULER_ENABLE=ON
LDR_ILB_SCHEDULER_CRON=<cron expression>

# RBA checking
LDR_RBA_CHECKING_PERIOD=<period>
```

---

## 8.8 Instance Identity Configuration

```properties
# This instance's identity
JITR_INSTANCE=RT
UB_PREFIX=<prefix, e.g. STC>
LDR_BATCH_NODE=N
```

---

## 8.9 Processing Behavior Configuration

```properties
# Retry
LDR_REPROCESS_RETRY_COUNT=<max retry attempts>

# Locking
MDN_LEVEL_LOCK=Y

# Vision 2.0
visionFlagEnable=ON
V2DT37304=Y
V2DT39120=N
V2DT41237=Y
V2DT41462=Y
V2DT41569=Y
V2DT41649=Y
V2DT15112=N
RBMVD18943=Y
ROSIOTFLAG=false

# Lists (comma-separated)
ORDataPopList=DATAPOP_TYPE1,DATAPOP_TYPE2,...
vision2EnabledEventsList=ACTIVATE,SUSPEND,DEACTIVATE,...
transactionLevelLockDataPopList=SOME_DATAPOP_TYPE,...
```

---

## 8.10 Email Alert Configuration

```properties
LDR_WARNING_EMAIL_FROM=loader-noreply@verizon.com
LDR_WARNING_EMAIL_TO=ops-team@verizon.com
LDR_WARNING_EMAIL_SUBJECT=JITR Loader Alert - {instance}
```

---

## 8.11 Encryption Configuration (Jasypt)

Sensitive values (passwords, credentials) are encrypted using Jasypt and stored in properties files as `ENC(ciphertext)`.

**In `bootstrap.properties`:**
```properties
jasypt.encryptor.password=<master key — provided at startup via env variable or JVM arg>
jasypt.encryptor.algorithm=PBEWithMD5AndDES
```

The master key is **never stored in a file**. It is provided at JVM startup via:
```
-Djasypt.encryptor.password=<key>
```
or as an environment variable.

---

## 8.12 Spring Cloud Config (`bootstrap.properties`)

```properties
spring.config.import=optional:configserver:
spring.cloud.config.fail-fast=true
spring.cloud.config.enabled=false
spring.profiles.active=native
```

Currently disabled. When re-enabled, the Config Server URL and credentials would be added here.

---

## 8.13 REST API Endpoints

All endpoints are under the context path `/loader`.

| HTTP Method | Full Path | Controller | Purpose |
|---|---|---|---|
| `GET` | `/loader/api/versionCheck/` | `VersionCheckController` | Returns application version, instance name, and key flag states |
| `GET` | `/loader/versioncheck` | `LoaderHealthCheckController` | Full health check: JVM info, JAR manifest, all feature flag values |
| `POST` | `/loader/checkLock/lock/{name}` | `LockController` | Manually acquire a named distributed lock (operational use) |
| `POST` | `/loader/checkLock/unlock/{name}` | `LockController` | Manually release a named distributed lock (operational use) |

### Version Check Response Fields (`/versioncheck`)
- Application version string (from `LoaderConstants.VERSION_ID`)
- Running instance (`JITR_INSTANCE`)
- JVM version and memory stats
- JAR manifest attributes
- All `@Value`-injected feature flag values
- AWS AppConfig flag values

---

## 8.14 JGroups Cluster Configuration (`bpmn.xml`)

Despite the `.xml` extension and filename suggesting BPMN, this file is a **JGroups protocol stack configuration**.

Key parameters:
- **Transport:** UDP multicast
- **Discovery:** PING protocol
- **Locking:** CENTRAL_LOCK protocol (final stack layer)
- **Fragmentation:** FRAG2 for large messages
- **Reliable delivery:** NAKACK, UNICAST

All Loader instances on the same network segment must use the same UDP multicast group address and port to form a cluster for distributed locking.

---

## 8.15 Logging Configuration (`log4j2.xml`)

### Appenders

| Appender Name | Type | Output Location | Purpose |
|---|---|---|---|
| `rollingLogger` | RollingFile | `${LDR_LOG_PATH}/loader.log` | Main application log |
| `archive` | RollingFile | `${LDR_LOG_PATH}/archive/` | Archived log |
| `ignorequeue` | RollingFile | `${LDR_LOG_PATH}/ignore.log` | Filtered/ignored events |
| `ELK` | Async (LMAX Disruptor) | ELK endpoint | Structured JSON for log aggregation |
| `BCP` | RollingFile | `${LDR_LOG_PATH}/bcp.log` | Bill cycle processing |
| `ILB` | RollingFile | `${LDR_LOG_PATH}/ilb.log` | ILB module |
| `SQL` | RollingFile | `${LDR_LOG_PATH}/sql.log` | SQL query logging |
| `Hikari` | RollingFile | `${LDR_LOG_PATH}/hikari.log` | HikariCP pool activity |

### Log Pattern

```
%d{HH:mm:ss.SSS} [%t] %-5level %logger{36}
  [%X{EventName}][%X{TransId}][%X{CustomerId}][%X{AccountNumber}][%X{mdn}]
  - %msg%n
```

The bracketed MDC (Mapped Diagnostic Context) fields are set per-message in `LoaderEventProcessor.processEvent()` using Log4j2's `ThreadContext`:

| ThreadContext Key | Value |
|---|---|
| `EventName` | CPF event name or datapop type |
| `TransId` | `cpfTranID` from EDRCPFMessage |
| `CustomerId` | `customerIdNo` |
| `AccountNumber` | `acctNo` |
| `mdn` | Mobile directory number |
