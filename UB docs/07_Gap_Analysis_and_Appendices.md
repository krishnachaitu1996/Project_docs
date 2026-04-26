# Usage Broker (UB) Ecosystem — Technical Specification

## Part 7: Gap Analysis, Cross-Cutting Concerns & Appendices

---

## 1. Gap Analysis

The following gaps have been identified through source code analysis across all nine repositories. These are areas where the current implementation either lacks standard best practices, has incomplete coverage, or presents operational risk.

---

### 1.1 Reliability & Fault Tolerance Gaps

| Gap ID  | Area                        | Finding                                                                                     | Risk Level | Recommendation                                                      |
|---------|-----------------------------|---------------------------------------------------------------------------------------------|------------|----------------------------------------------------------------------|
| GAP-001 | Dead Letter Queue (DLQ)     | No DLQ configured for any Kafka topic. Failed messages are written to local files.          | **HIGH**   | Implement Kafka DLQ topics for each consumer. Local file retry is fragile across restarts and not visible to monitoring. |
| GAP-002 | Idempotent Processing       | No deduplication at the Kafka consumer level. `enable.idempotence=true` only protects producer-side duplicates. | **HIGH**   | Implement consumer-side idempotency using record GRI or audit sequence ID stored in a fast-lookup store (Redis/Cassandra). |
| GAP-003 | Circuit Breaker             | No circuit breaker pattern for Oracle DB calls, Cassandra queries, or Kafka publishes.       | **MEDIUM** | Integrate Resilience4j circuit breakers on DB and Kafka operations to prevent cascading failures during outages. |
| GAP-004 | Backpressure                | Thread pools (`FixedThreadPool`) have no queue depth limits. OOM is caught reactively.       | **MEDIUM** | Implement bounded work queues with rejection policies. Add Kafka consumer `pause()`/`resume()` backpressure. |
| GAP-005 | Graceful Shutdown           | `@PreDestroy` present but no `ShutdownHook` coordination for in-flight Kafka batches or file operations. | **MEDIUM** | Implement `SmartLifecycle` with ordered shutdown — drain Kafka buffers → close file writers → flush audit. |

### 1.2 Data Integrity Gaps

| Gap ID  | Area                        | Finding                                                                                     | Risk Level | Recommendation                                                      |
|---------|-----------------------------|---------------------------------------------------------------------------------------------|------------|----------------------------------------------------------------------|
| GAP-006 | Exactly-Once Semantics      | Kafka producers use idempotence but not transactions. File audit and Kafka publish are not atomic. | **HIGH** | For critical paths, use Kafka transactions (`transactional.id`) to couple audit messages with data messages. |
| GAP-007 | Re-rating Synchronization   | No mechanism to ensure JITR re-rating output is correlated back to the originating ECS error record. | **MEDIUM** | Implement end-to-end correlation IDs propagated through Kafka headers. |
| GAP-008 | Duplicate File Detection     | Duplicate check relies on Oracle MZAUD query. If DB is slow/down, files may be double-processed. | **MEDIUM** | Add a local Bloom filter or in-memory cache as a first-pass duplicate check before DB lookup. |
| GAP-009 | MMS Clone Count Accuracy    | MMS multi-recipient clone/split logic modifies `auditID` in-place with string manipulation. Off-by-one risks on edge cases. | **LOW** | Add unit test coverage for edge cases: 0 recipients, 1 recipient, max recipients, empty recipientAddress. |

### 1.3 Observability Gaps

| Gap ID  | Area                        | Finding                                                                                     | Risk Level | Recommendation                                                      |
|---------|-----------------------------|---------------------------------------------------------------------------------------------|------------|----------------------------------------------------------------------|
| GAP-010 | Distributed Tracing         | No OpenTelemetry/Zipkin/Jaeger integration. File → Kafka → Process → Output has no trace correlation. | **HIGH** | Integrate OpenTelemetry with Kafka header propagation for end-to-end trace visibility. |
| GAP-011 | Metrics Export               | No Micrometer/Prometheus metrics beyond Spring Actuator defaults.                            | **MEDIUM** | Export custom metrics: records/sec, batch duration, error rate, queue depth, Kafka lag per consumer group. |
| GAP-012 | Alerting Rules               | ELK/Kibana integration exists, but no documented alerting thresholds or SLO definitions.     | **MEDIUM** | Define SLOs (e.g., p99 file processing < 5s, error rate < 1%) and configure Kibana alerts. |
| GAP-013 | Health Check Depth           | Actuator health endpoint exists but does not check Kafka broker, Cassandra, or Oracle deep health. | **LOW** | Implement custom `HealthIndicator` for each external dependency. |

### 1.4 Security Gaps

| Gap ID  | Area                        | Finding                                                                                     | Risk Level | Recommendation                                                      |
|---------|-----------------------------|---------------------------------------------------------------------------------------------|------------|----------------------------------------------------------------------|
| GAP-014 | Kafka Authentication        | No SASL/SSL configuration visible in properties (bootstrap servers use plaintext ports).     | **HIGH**   | Enable SASL_SSL with SCRAM or mTLS for Kafka broker connections in production. |
| GAP-015 | Actuator Exposure            | Actuator endpoints (suspend/resume/status) have no authentication or RBAC.                  | **MEDIUM** | Secure actuator endpoints with Spring Security + role-based access. |
| GAP-016 | Cassandra Authentication    | CassandraConfiguration does not configure credentials or SSL.                               | **MEDIUM** | Enable Cassandra native authentication and TLS for client-to-node communication. |

### 1.5 Operational Gaps

| Gap ID  | Area                        | Finding                                                                                     | Risk Level | Recommendation                                                      |
|---------|-----------------------------|---------------------------------------------------------------------------------------------|------------|----------------------------------------------------------------------|
| GAP-017 | Configuration Management    | Properties use Maven placeholder tokens (`@...@`). No Spring Cloud Config or centralized config. | **LOW** | Consider Spring Cloud Config Server for dynamic configuration without redeployment. |
| GAP-018 | Database Migration           | No Flyway/Liquibase for schema versioning. SQL scripts in `sql/` directory are manual.       | **LOW** | Adopt Flyway for version-controlled database migrations. |
| GAP-019 | Container Readiness          | No Kubernetes readiness/liveness probe differentiation (only basic health endpoint).         | **LOW** | Implement startup, liveness, and readiness probes with dependency checks. |
| GAP-020 | File Retry Storm             | `KafkaRetryFailureMechanism` retries all failed messages on each cron cycle without rate limiting. | **MEDIUM** | Add exponential backoff and max-retry-count to prevent retry storms. |

---

## 2. Cross-Cutting Concerns

### 2.1 Technology Stack Summary

| Component          | Technology                   | Version       |
|--------------------|------------------------------|---------------|
| Language           | Java                         | 17            |
| Framework          | Spring Boot                  | 3.1.1         |
| Cloud              | Spring Cloud                 | 2022.0.3      |
| Messaging          | Apache Kafka                 | 3.5.1         |
| RDBMS              | Oracle                       | 19c           |
| NoSQL              | Apache Cassandra             | 4.x (OSS)    |
| Stream Processing  | Apache Pulsar (5G path)      | —             |
| Connection Pool    | HikariCP                     | (Boot default)|
| Serialization      | Jackson / Gson               | 2.15.2 / —   |
| Encryption         | Jasypt + BouncyCastle        | 3.0.5 / 1.76  |
| Logging            | Log4j2 + LMAX Disruptor      | 2.19.0        |
| XML Processing     | XStream (5G)                 | 1.4.x         |
| Compression        | Snappy (Kafka), Zstandard (5G), GZip (archive) | — |
| Scheduling         | Spring @Scheduled, Quartz (5G audit) | —      |
| Service Discovery  | Netflix Eureka               | (Cloud 2022)  |
| Build              | Maven                        | 3.x           |
| Testing            | JUnit 5, JMockit             | 1.34          |
| Code Coverage      | JaCoCo                       | —             |

### 2.2 Port Allocation Map

| Application                           | Port   | Management Port |
|---------------------------------------|--------|-----------------|
| `jitr-ub-msg-dist` (File Distributor) | 14190  | —               |
| `jitr-ub-msg` (Mediation Engine)      | 14060  | —               |
| `jitr-data-dsl` (Data Stream Layer)   | Config | —               |
| `jitr-ub-data-d5g-filestreamer-app`   | 14250  | Config          |
| `jitr-ub-data-d5g-app`               | 14251  | Config          |
| `jitr-ub-data-d5g-output-app`        | 14252  | Config          |
| `jitr-ub-data-audit-aggregator`      | 14256  | Config          |

### 2.3 Database Schema Map

| Schema       | Database | Used By                                  | Purpose                   |
|--------------|----------|------------------------------------------|---------------------------|
| **MZAUD**    | Oracle   | `jitr-ub-msg`, `jitr-data-dsl`, `jitr-ub-msg-dist` | File/record-level audit |
| **MZADMIN**  | Oracle   | `jitr-ub-msg`, `jitr-data-dsl`, `d5g-app`, `d5g-output-app` | Config & reference |
| **REED**     | Oracle   | `jitr-data-dsl`                          | Cell-site / geo data      |
| **AUD_5G**   | Oracle   | `d5g-filestreamer`, `audit-aggregator`   | 5G pipeline audit         |
| **PROCCTRL** | Oracle   | `audit-aggregator`                       | Quartz / process control  |
| **Cassandra**| Cassandra| `jitr-ub-ecs-cdr-generator`, `jitr-ub-msg` | Error record storage   |

### 2.4 Kafka Topic Architecture

| Topic Category          | Producer                    | Consumer                     | Content                        |
|-------------------------|-----------------------------|------------------------------|--------------------------------|
| Raw Usage (MSG)         | `jitr-ub-msg-dist`          | `jitr-data-dsl`              | Batched raw records (~delimiter)|
| Audit (MSG)             | `jitr-ub-msg-dist`          | Audit consumers              | File-level audit JSON          |
| ECS Error               | `jitr-ub-msg` (via ecs-dist)| ECS downstream consumers     | Binary CDR + metadata          |
| ECS Handshake           | `jitr-ub-msg` (via ecs-dist)| ECS downstream consumers     | INITIAL/COMMIT/ROLLBACK        |
| RBM 3G/4G/Recycle       | Upstream producers           | `jitr-data-dsl`              | Pipe-delimited data records    |
| D5G Input               | `d5g-filestreamer`/`d5g-pulsar` | `d5g-app`               | 5G XML/JSON records            |
| D5G JITR (RT/RO/RS/BS)  | `d5g-app`                   | JITR rating instances        | Rated 5G records               |
| D5G Auxiliary            | `d5g-app`                   | `d5g-output-app`             | DROPPED/LRA/BCE/RSS/AFB/NONBILLING |
| D5G Audit               | All 5G modules               | `audit-aggregator`           | Audit events                   |

---

## 3. Near Real-Time Alerting Logic

### 3.1 Current Alerting Mechanisms

| Mechanism                     | Trigger                              | Action                              |
|-------------------------------|--------------------------------------|-------------------------------------|
| OOM Error Detection           | `OutOfMemoryError` caught            | Suspend all processing; log heap stats |
| ECS Error Threshold           | Error count ≥ `maxKafkaErrorThresholdVal` (10) | Abort file; cancel batch  |
| Invalid Config on Startup     | Missing/invalid required properties  | Application shutdown (`context.close()`) |
| Kafka Send Failure            | `ProducerRecord` send fails          | Write to local file for retry       |
| File Processing Exception     | Any uncaught exception in `processFile()` | Move file to error dir; log      |
| Cassandra Fetch Failure       | Query exception in ECS CDR reader    | Single retry then propagate error  |

### 3.2 Recommended Alerting Enhancements

| Alert                                   | Threshold                            | Channel              |
|-----------------------------------------|--------------------------------------|----------------------|
| Consumer Lag > N                        | 10,000 records behind                | PagerDuty / Slack    |
| Error Rate > X%                         | > 5% of records per file             | Kibana Alert → Email |
| File Age in Staging > Y minutes         | > 120 min (indicates stuck file)     | Kibana Alert         |
| OOM Suspension Event                    | Any occurrence                       | PagerDuty (P1)       |
| DB Connection Pool Exhaustion           | Active connections = max pool size   | Prometheus Alert     |
| Kafka Retry File Count > N             | > 100 pending retry files            | Kibana Alert         |
| Cassandra Latency > P99                 | > 500ms at P99                       | Prometheus Alert     |

---

## 4. Appendices

### Appendix A: Repository Map

| Repository                   | Type         | Standalone? | Depends On                           |
|------------------------------|-------------|-------------|--------------------------------------|
| `ub-jitr-domain-parent`     | Parent POM   | N/A         | —                                    |
| `jitr-ub-msg-dist`          | Application  | Yes         | `jitr-kafka-library`, `ub-jitr-domain-parent` |
| `jitr-ub-msg-decoder`       | Library      | No          | `jpos`, `ub-jitr-domain-parent`     |
| `jitr-ub-msg`               | Application  | Yes         | `jitr-ub-msg-decoder`, `jitr-ub-ecs-distributor`, `jitr-ub-ecs-cdr-generator`, `jitr-kafka-library` |
| `jitr-ub-ecs-distributor`   | Library      | No          | `ub-jitr-domain-parent`             |
| `jitr-ub-ecs-cdr-generator` | Library      | No          | `ub-jitr-domain-parent`             |
| `jitr-kafka-library`        | Library      | No          | `ub-jitr-domain-parent`             |
| `jitr-data-dsl`             | Application  | Yes         | `ub-jitr-domain-parent`, `jitr-elk-library` |
| `jitr-ub-data`              | Multi-module | Yes (per module) | `ub-jitr-domain-parent`, `jitr-elk-library` |

### Appendix B: Configuration Property Placeholder Tokens

Properties use Maven resource filtering with token patterns `@TOKEN_NAME@` replaced at build time per environment:

| Token Pattern                               | Example Resolution                    |
|---------------------------------------------|---------------------------------------|
| `@UB.MSG.DIST.KAFKA.BOOTSTRAP.SERVERS@`    | `broker1:9092,broker2:9092`           |
| `@UMM.SCH.DSL.SERVER.PORT@`                | `14070`                               |
| `@JITR.UB.D5G.MAIN.INPUT.KAFKA.TOPIC@`     | `UB_D5G_INPUT_TDC`                   |
| `@JITR.UB.D5G.FS.INPUT.FOLDER.LIST@`       | `/data/dm/5g/input`                   |
| `@AUDIT.AGGREGATOR.SOURCE.TOPIC@`           | `UB_D5G_AUDIT_TDC`                   |

### Appendix C: Glossary

| Term       | Definition                                                                 |
|------------|----------------------------------------------------------------------------|
| **BCD**    | Binary Coded Decimal — telecom encoding format for CDRs                   |
| **CDR**    | Call Detail Record — raw usage event from a network switch                |
| **DM**     | Data Manager — upstream system depositing files for UB processing         |
| **DSL**    | Data Stream Layer — the data/voice processing pipeline                    |
| **ECS**    | Error Correction Subsystem — closed-loop error recovery mechanism         |
| **GRI**    | Global Record Identifier — unique traceability ID per record              |
| **JITR**   | Just-In-Time Rating — real-time billing system consuming UB output        |
| **MDN**    | Mobile Directory Number — subscriber phone number (10-digit)              |
| **MZADMIN**| Oracle schema for admin/config/reference data                             |
| **MZAUD**  | Oracle schema for audit records                                           |
| **RBM**    | Rate Base Module — output format for data pipeline                        |
| **REED**   | Oracle schema for cell-site and geolocation reference data                |
| **REVO**   | Revenue Operations — billing error/adjustment system                      |
| **UDR**    | Usage Detail Record — normalized usage record after mediation             |
| **UMM**    | Usage Mediation & Management — the overarching mediation platform         |

### Appendix D: Document Index

| Part | Document                                        | Scope                              |
|------|-------------------------------------------------|------------------------------------|
| 1    | `01_Executive_Overview_and_Architecture.md`     | System overview, architecture, tech stack |
| 2    | `02_File_Distributor_jitr_ub_msg_dist.md`       | File ingestion & Kafka publishing   |
| 3    | `03_Decoder_Library_jitr_ub_msg_decoder.md`     | Raw CDR decoding (BCD/ASCII/JSON/XML) |
| 4    | `04_Core_Mediation_jitr_ub_msg.md`              | Decode → Validate → Enrich → Route |
| 5    | `05_ECS_Subsystem.md`                           | Error handling, Cassandra, Kafka retry |
| 6    | `06_Data_Processing_DSL_and_5G.md`              | Data/Voice DSL, 5G multi-module     |
| 7    | `07_Gap_Analysis_and_Appendices.md` (this doc)  | Gaps, cross-cutting, glossary       |

---

*End of Technical Specification — Usage Broker (UB) Ecosystem*
