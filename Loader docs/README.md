# JITR Loader 2.0 — Documentation Index

**Application:** JITR Loader (Just-In-Time Rating Loader)  
**Version:** Loader2.0 ER 26.01  
**Artifact:** `jitrLoader-app`  
**Framework:** Spring Boot 3.1.5 / Java 17  

---

## Document Index

| # | Document | Audience | Description |
|---|---|---|---|
| 1 | [System Architecture](./01_system_architecture.md) | Engineers | Overall architecture, topology, and sub-modules |
| 2 | [Transaction Flow](./02_transaction_flow.md) | Engineers | End-to-end message processing pipeline |
| 3 | [Event Type Catalog](./03_event_type_catalog.md) | Engineers / Analysts | All supported event and datapop types |
| 4 | [Data Operations](./04_data_operations.md) | Engineers / DBAs | Database topology, entities, repositories |
| 5 | [Messaging & Caching](./05_messaging_and_caching.md) | Engineers | IBM MQ configuration and EhCache |
| 6 | [External Integrations](./06_external_integrations.md) | Engineers | JIS, DVS, DMD, AWS AppConfig, email |
| 7 | [Feature Flags & Toggles](./07_feature_flags.md) | Engineers / Operators | All feature flags and runtime toggles |
| 8 | [Environment & Configuration](./08_environment_config.md) | Engineers / Operators | Config files, properties, REST endpoints |
| 9 | [Error Handling & Observability](./09_error_handling.md) | Engineers / Operators | Error codes, logging, ELK, reprocessing |
| 10 | [Semi-Technical Overview](./10_semi_technical_guide.md) | All audiences | Plain-language guide to what the Loader does |

---

## Quick Reference

- **Server Port:** `14175`  
- **Context Path:** `/loader`  
- **Health Check:** `GET /versioncheck`  
- **Version Check:** `GET /api/versionCheck/`  
- **Lock Management:** `POST /checkLock/lock/{name}` · `POST /checkLock/unlock/{name}`  
- **Reflow Schedule:** Every 20 minutes (`0 */20 * * * *`)  
- **Instances:** BS (VISB), RT (VISN), RS (VISW), RO (VISE)  
