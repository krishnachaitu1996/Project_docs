# JITR Recon — External Integration & Dependency Map

## 1. External Integration Points

### 1.1 JIS — Jacket Information System (Primary Integration)

JIS is the **most critical external dependency**. Every billing fix in the system is applied by making an HTTP call to JIS, which then updates the RBM database.

```
┌────────────────────┐         HTTP POST          ┌────────────────────┐
│  JITR Recon        │ ──────────────────────────▶ │   JIS Server       │
│  (Fix Processor)   │ ◀────────────────────────── │   (RBM Gateway)    │
│                    │      XML Response           │                    │
│  20 parallel       │                             │  Port: 14305       │
│  threads           │                             │  Host: tpaldrbmva  │
│  50 conn pool      │                             │  012.ebiz.verizon  │
└────────────────────┘                             └────────────────────┘
```

**Integration Architecture:**
- **Protocol:** HTTP/HTTPS (REST with XML payloads)
- **Authentication:** HTTP Basic (Base64-encoded `jis.username:jis.psd`)
- **Connection Pool:** 50 connections per system (PooledHttpClients.xml)
- **Timeout:** 100,000,000 ms (effectively unlimited — relies on application-level timeout)
- **Retry:** 1 retry on failure, no retry on already-sent requests
- **Parallelism:** 20 JIS threads (`jis.threads=20`)
- **Timeout Hold:** If >5 consecutive timeouts (`jis.timeout.hold.threshold.limit=5`), triggers hold alert

**Key Classes:**
| Class | Role |
|-------|------|
| `JisServiceHandler` | Builds JIS requests from SubXltBlRbmRecord, orchestrates the call |
| `MakeJisCall` | HTTP client — 50+ static endpoint methods for different operations |
| `JisCallService` | Core HTTP request/response handling |
| `PooledHttpClient` | Apache HttpClient wrapper with connection pooling |
| `PooledHttpClientFactory` | Factory pattern for creating pooled clients |
| `JisCreateXmlHelper` | XML request document builder |
| `ProxyHandlerService` | Proxy configuration for environments behind corporate proxy |

**JIS Operations (50+ endpoint types):**
| Category | Operations |
|----------|-----------|
| Customer | Create, Update, Delete customer |
| Account | Create, Update, Terminate account |
| SPLAN | Create, Update, Delete service plan |
| EventSource | Create, Update, Delete event source |
| Product | Create, Update, Delete service product |
| SFO/SPOAC | Special feature operations |
| CPARD | Charge plan adjustment record operations |
| PPLAN | Package plan operations |
| Address | Update customer/account addresses |

**PooledHttpClients.xml Configuration:**
```xml
<systems>
  <system id="DEFAULT" connectionsPerHost="50">
    <pool timeout="100000000" socketTimeout="100000000"
          staleCheck="true" retryCount="1" sentRetryEnabled="false"/>
  </system>
  <system id="DEV" connectionsPerHost="50">...</system>
  <system id="TEST" connectionsPerHost="50">...</system>
</systems>
```

**Vision 2.0 JIS:**
- `JisServiceHandlerVision2` — enhanced handler with subscription-level operations
- `ParallelJISUtilVision2` — batched parallel JIS calls for V2 entity types

---

### 1.2 Spring Cloud Config Server

Centralized configuration management for all runtime properties.

```
┌────────────────────┐         HTTP GET            ┌────────────────────┐
│  JITR Recon        │ ──────────────────────────▶ │  Config Server     │
│  (bootstrap)       │ ◀────────────────────────── │  Port: 8888        │
│                    │      Properties              │  Profile: native   │
└────────────────────┘                             └────────────────────┘
```

**Configuration:**
```properties
# bootstrap.properties
spring.cloud.config.uri=http://tpaldrbmva016.ebiz.verizon.com:8888
spring.profiles.active=native
spring.cloud.config.fail-fast=true
management.endpoints.web.exposure.include=refresh
```

**Behavior:**
- On startup, the app fetches externalized properties from the Config Server
- `spring.profiles.active=native` means Config Server reads properties from its local filesystem
- `fail-fast=true` — app will fail to start if Config Server is unreachable
- `/actuator/refresh` endpoint can reload properties at runtime (enabled)
- **Production:** URI is tokenized as `@RECON.CLOUD.CONFIG.URI@/config-server` (replaced at deployment)

---

### 1.3 AWS AppConfig (Feature Flags)

```
┌────────────────────┐         HTTP GET            ┌────────────────────┐
│  JITR Recon        │ ──────────────────────────▶ │  AWS AppConfig     │
│  (FeatureFlagMgr)  │ ◀────────────────────────── │  (feature store)   │
│                    │      Flag Evaluation          │                    │
└────────────────────┘                             └────────────────────┘
```

**SDK:** `com.vzw.core:feature-flags-java:2.0-SNAPSHOT` (Verizon internal library)

**Configuration (FeatureFlagConfig.java):**
```java
@Bean
public FeatureFlagManager featureFlagManager() {
    return new FeatureFlagManager.Builder(serverUrl, serverPort, environment).build();
}
```

**Usage Pattern:**
```java
boolean enabled = featureFlagManager.getFlag(applicationName, configProfile, flagName);
```

**Defined Flags:**
| Flag | Profile | Purpose |
|------|---------|---------|
| `vision2AAFileCollectorFFlag` | `vision2BillingAACfgProfile` | Controls Vision 2.0 AA file collector daily job |

**Deployment Strategy:** `AppConfig.Linear50PercentEvery30Seconds` — gradual rollout

---

### 1.4 DVS — Digital Video Services (JAXB Model Integration)

DVS is not called as a live remote service. Instead, JAXB-generated model classes represent the DVS data schema, used for **encoding Vision data** into DVS-compatible formats for ODR processing.

**50+ JAXB Model Classes** (`com.verizon.jitr.recon.dvs`):

| Category | Classes |
|----------|---------|
| Customer-level | `CustomerInfo`, `CustomerBillCycleInfo`, `CustomerLineCountInfo` |
| Account-level | `AccountInfo`, `AccountInfoList`, `AccountListUnderCustomer`, `AccountMCDetails` |
| Device-level | `DeviceSimEquipmentAssn`, `DvcSimEqpAssn` |
| Line-level | `LineOfService`, `MdnInfo`, `MdnInfoList`, `LnSvcProd` |
| Special Features | `SpecialFeatureMtnInfo`, `SpecialFeatureMtnList` |
| Plans | `PplanMtnInfo`, `PromoCustMtnInfo`, `SharePlanInactive` |
| Service wrapper | `Service`, `ServiceHeader`, `ServiceBody`, `ServiceResponse` |

**50+ Encoder Classes** serialize internal data to DVS format for ODR table output.

---

### 1.5 ODR — On-Demand Rating Subsystem

ODR is an **internal subsystem** (not a remote service) that processes single-customer rating requests.

```
MZADMIN.REF_RECON_ONDEMAND   →   ODRProcessor   →   JIS API
      (request queue)              (processing)       (apply fixes)
```

**Key Components:**
| Class | Purpose |
|-------|---------|
| `ODRProcessor` | Main batch processing of ODR requests |
| `OdrHelper` | Helper for regular on-demand requests |
| `OdrDvsHelper` | DVS data encoding for ODR |
| `OdrUtil` | File operations and data utilities |
| `OdrQueries` | SQL for `REF_RECON_ONDEMAND` table operations |
| `OdrConstants` | ODR-specific constants |

**40+ ODR Encoder Classes** (in `odr/bo/`):
- `CustomerInfoEncoder`, `CustAcctEncoder`, `CustAcctMdnEncoder`
- `LnSvcProdEncoder`, `AcctSplanEncoder`, `BlAcctLnStatEncoder`
- `PplanMtnEncoder`, `SubSfMdnEncoder`, `DvcSimEqpAssnEncoder`
- And 30+ more for each billing entity type

---

### 1.6 Email (SMTP)

```
┌────────────────────┐         SMTP                ┌────────────────────┐
│  JITR Recon        │ ──────────────────────────▶ │  Verizon SMTP      │
│  (EmailUtils)      │                              │  smtp.verizon.com  │
└────────────────────┘                             └────────────────────┘
```

**Configuration:**
```properties
email.host=smtp.verizon.com        # (or vzsmtp.verizon.com)
email.from=ReconAlert@Verizon.com
email.fromcontrol=ReconControls@Verizon.com
email.to=ReconEmailAlertDistroNew@one.verizon.com
```

**Alert Types:**
| Trigger | Subject Template |
|---------|-----------------|
| Threshold breach | `SUB00038` — Delete hold evaluation results |
| Empty extract | `APP00005` — Empty file alerts |
| JIS timeout | `SUB00028` — JIS timeout alerts |
| Processing error | `SYS00001` — System error |
| Run completion | Run summary email |
| Disk space low | Disk space warning |

**Text/SMS Alerts:** Sent to mobile numbers via email-to-SMS gateway (e.g., `5083090984@vzwpix.com`)

---

### 1.7 SSH/SCP (File Transfer)

**Library:** JSch 0.1.55

Used for remote file operations (transferring splitter output, large customer files). Configuration is property-driven.

---

## 2. Third-Party Library Dependency Map

### 2.1 Spring Ecosystem

| Library | Version | Purpose |
|---------|---------|---------|
| Spring Boot Starter | 3.3.5 | Application framework |
| Spring Boot Web | 3.3.5 | Embedded Tomcat, REST controllers |
| Spring Boot JDBC | 3.3.5 | JdbcTemplate, transaction management |
| Spring Boot Validation | 3.3.5 | Bean validation (Hibernate Validator) |
| Spring Boot Log4j2 | 3.3.5 | Logging framework bridge |
| Spring Boot JSON | 3.3.5 | Jackson auto-configuration |
| Spring Boot Web Services | 3.3.5 | Web service support |
| Spring Boot Configuration Processor | 3.3.5 | Metadata for `@ConfigurationProperties` |
| Spring Cloud Config Client | 2022.0.3 | Centralized config fetching |
| Spring Cloud Bootstrap | 2022.0.3 | Bootstrap context for config |
| Spring Security Crypto | — | Password encoding utilities |

### 2.2 Data & Database

| Library | Version | Purpose |
|---------|---------|---------|
| Oracle JDBC (ojdbc11) | — | Oracle database driver |
| HikariCP | (Spring Boot managed) | High-performance connection pooling |
| Commons DBCP | 1.4 | Legacy connection pool (fallback) |

### 2.3 Integration & Messaging

| Library | Version | Purpose |
|---------|---------|---------|
| Apache Camel Core | 3.21.5 | ETL pipeline routing engine |
| Camel Bean Starter | 3.21.5 | Bean component for Camel |
| Camel CSV | 3.21.5 | CSV file processing in Camel routes |
| Camel Metrics | 3.21.5 | Processing metrics in Camel |
| Camel Base64 | 3.21.5 | Base64 encoding in Camel |
| Camel Spring | 3.21.5 | Spring integration for Camel |
| Quartz | 2.3.2 | Cron scheduling engine |
| Apache HttpClient | 4.5.14 | HTTP client for JIS API calls |
| Geronimo Commons HttpClient | 3.1_2 | Legacy HTTP client (SOAP support) |

### 2.4 Security & Encryption

| Library | Version | Purpose |
|---------|---------|---------|
| Jasypt Spring Boot | 3.0.5 | Encrypted property values (`ENC(...)`) |
| BouncyCastle | 1.78.1 (jdk18on) | Cryptographic provider for Jasypt |
| Spring Security Crypto | — | Password encryption utilities |

### 2.5 XML & JSON Processing

| Library | Version | Purpose |
|---------|---------|---------|
| JAXB (Jakarta XML Bind) | — | XML→Java marshalling for DVS models |
| JAXB Implementation | — | Runtime JAXB implementation |
| JDOM2 | — | XML document manipulation |
| Jaxen | — | XPath expression evaluation |
| Woodstox | 6.5.1 | High-performance XML parser |
| Jackson Core/Databind/Annotations | (managed) | JSON serialization |
| Jackson XML | (managed) | XML serialization via Jackson |
| Jackson JSR-310 | (managed) | Java 8 date/time support |
| org.json | 20231013 | Simple JSON operations |
| json-simple | 1.1.1 | Lightweight JSON parsing |
| SnakeYAML | 2.2 | YAML parsing (feature flags) |

### 2.6 Utilities

| Library | Version | Purpose |
|---------|---------|---------|
| Apache Commons Lang3 | (managed) | String utils, reflection, builders |
| Apache Commons Net | 3.10.0 | FTP/network utilities |
| Apache Commons IO | 2.17.0 | File I/O utilities |
| Apache Commons Codec | (managed) | Base64, hex encoding |
| Apache Commons Configuration | 1.9 | Configuration file parsing |
| Google Guava | 33.0.0-jre | Collections, caching, utilities |
| LMAX Disruptor | 3.3.4 | High-performance async logging |

### 2.7 SSH & Email

| Library | Version | Purpose |
|---------|---------|---------|
| JSch | 0.1.55 | SSH/SCP file transfers |
| Jakarta Mail | 2.0.1 | Email sending (alerts) |

### 2.8 Web & API

| Library | Version | Purpose |
|---------|---------|---------|
| Spring Boot Tomcat | 3.3.3 | Embedded servlet container |
| Undertow | 2.3.15 | Alternative servlet container (available) |
| Springfox Swagger 2/UI | 3.0.0 | API documentation |
| Swagger Annotations | 1.5.14 | API annotations |
| ClassGraph | 4.8.176 | Classpath scanning (Swagger support) |

### 2.9 Internal Verizon Libraries

| Library | Version | Purpose |
|---------|---------|---------|
| `com.vzw.core:feature-flags-java` | 2.0-SNAPSHOT | AWS AppConfig feature flag SDK |

### 2.10 Build Plugins

| Plugin | Purpose |
|--------|---------|
| `spring-boot-maven-plugin` | Repackages JAR as executable |
| `maven-compiler-plugin` | Java 17 compilation with debug symbols |
| `buildnumber-maven-plugin` | Git revision number in build metadata |
| `git-commit-id-plugin` | Writes git info to `gitrecon.properties` |
| `maven-surefire-plugin` | Test execution |

---

## 3. Integration Dependency Diagram

```
                          ┌──────────────────┐
                          │   AWS AppConfig   │
                          │  (Feature Flags)  │
                          └────────┬─────────┘
                                   │
┌──────────────────┐    ┌──────────┴──────────┐    ┌──────────────────┐
│  Spring Cloud    │    │                     │    │   JIS Server     │
│  Config Server   │───▶│    JITR RECON APP   │───▶│  (RBM Gateway)   │
│  (Port 8888)     │    │                     │    │  (Port 14305)    │
└──────────────────┘    │  Spring Boot 3.3.5  │    └──────────────────┘
                        │  + Apache Camel     │
                        │  + Quartz           │    ┌──────────────────┐
┌──────────────────┐    │  + HikariCP         │───▶│  Verizon SMTP    │
│  Oracle Database │    │                     │    │  (Email Alerts)  │
│  ├─ UBSR         │◀──▶│                     │    └──────────────────┘
│  ├─ RBM (6 nodes)│    │                     │
│  ├─ MzAdmin      │    │                     │    ┌──────────────────┐
│  ├─ Audit        │    │                     │───▶│  Remote Servers  │
│  └─ REED         │    │                     │    │  (SSH/SCP via    │
└──────────────────┘    └─────────────────────┘    │   JSch)          │
                                                    └──────────────────┘
```
