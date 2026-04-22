# Section 6 — External Integrations

**Application:** JITR Loader 2.0  
**Navigation:** [← Messaging & Caching](./05_messaging_and_caching.md) | [Index](./README.md) | [Feature Flags →](./07_feature_flags.md)

---

## 6.1 JIS — JITR Integration Service (REST)

JIS is the primary read-side service for the Loader. The Loader calls JIS for all reference data lookups that go beyond what is cached in EhCache. JIS abstracts the RBM and UBSR read operations.

**Configuration:**
```properties
jis.host=<JIS hostname>
jis.port=<JIS port>
```

**HTTP Client:** Spring `RestTemplate` backed by Apache HttpClient 4.5.14.

### JIS Endpoint Catalog (30+ endpoints)

All JIS paths are defined as constants in `LoaderConstants.java`:

#### RBM Data Endpoints

| Constant | Path | Description |
|---|---|---|
| `RBM_PRICEPLAN_URL` | `/rbmPriceplan/` | Price plan lookup |
| `RBM_PRODUCT_URL` | `/rbmProduct/` | Product record |
| `RBM_PRODUCT_STATUS_URL` | `/rbmProductStatus/` | Product status |
| `RBM_PRODUCT_TARIFF_URL` | `/rbmProductTariff/` | Product tariff details |
| `RBM_PRODUCT_ADDON_URL` | `/rbmProductAddon/` | Product add-on rates |
| `RBM_ACCOUNTS_URL` | `/rbmAccounts/` | RBM account data |
| `RBM_CUST_MTN_URL` | `/rbmCustMtn/` | Customer MTN in RBM |
| `RBM_MTN_STATUS_URL` | `/rbmMtnStatus/` | MTN status in RBM |

#### Subscriber Plan & Feature Endpoints

| Constant | Path | Description |
|---|---|---|
| `SPLAN_URL` | `/splanData/` | Service plan (SPLAN) lookup |
| `SPO_URL` | `/spoData/` | Special offer (SPO) lookup |
| `SFO_OFFER_URL` | `/sfoOffer/` | Special Feature Offer details |
| `PPLAN_URL` | `/pplanData/` | Price plan reference data |

#### Subscriber Account Endpoints

| Constant | Path | Description |
|---|---|---|
| `TIMEZONE_URL` | `/timezoneData/` | Timezone resolution for a subscriber |
| `TRANSACTION_MAP_URL` | `/transactionMap/` | Transaction map reference |

#### SPO / SPLAN / SPOLN Operation Codes

`LoaderConstants` also defines operation codes used when building JIS requests for ILB processing:

| Constant Group | Values | Purpose |
|---|---|---|
| SPO operations | `SPO_ADD_SUBSCRIBER`, `SPO_REMOVE_SUBSCRIBER`, etc. | SPO add/remove actions |
| SPLAN operations | `SPLAN_ALL`, `SPLAN_ACTIVATE`, etc. | SPLAN change types |
| SPOLN operations | `SPOLN_ADD`, `SPOLN_REMOVE` | Global SFO to SPOLN operations |

---

## 6.2 DVS — Data Validation Service

**Configuration:**
```properties
dvs.url=<DVS endpoint URL>
```

DVS is called during event processing to validate customer data fields. It acts as a secondary validation layer beyond what the Loader checks internally.

---

## 6.3 DMD — Device Management Data API

DMD is called during device-related event processing. Audit records for DMD API interactions are written to the `DMD_AUDIT_INFO` table (`DmdAuditInfo` entity) in UBSR.

The `DmdDeviceDataRequest` DTO encapsulates the request structure sent to the DMD API.

---

## 6.4 AWS AppConfig — Runtime Feature Flags

**Class:** `FeatureFlagConfig.java`  
**Library:** `com.vzw.core:feature-flags-java:2.0-SNAPSHOT`

AWS AppConfig allows feature flags to be toggled at runtime without restarting or redeploying the application. This is used as a complement to the property-file-based flags (Section 7).

**Capabilities:**
- Pull feature flag values from a configured AWS AppConfig application/environment/profile
- Flags evaluated per-call — changes take effect on the next poll without restart
- Values are exposed at the `/versioncheck` health check endpoint

---

## 6.5 JIS Integration Pattern

Each JIS call follows this pattern:

```java
// Build URL
String url = "http://" + jisHost + ":" + jisPort + LoaderConstants.RBM_PRICEPLAN_URL + customerId;

// Make GET request
ResponseEntity<SomeDto> response = restTemplate.getForEntity(url, SomeDto.class);

// Handle response
if (response.getStatusCode() == HttpStatus.OK) {
    SomeDto data = response.getBody();
    // use data
}
```

Errors from JIS calls are caught and translated to error codes in the `LDR_D_E7xxx` range.

---

## 6.6 Email Alerts

The Loader sends email notifications on critical processing events and system-level failures.

**Configuration:**
```properties
LDR_WARNING_EMAIL_FROM=<sender address>
LDR_WARNING_EMAIL_TO=<recipient list>
LDR_WARNING_EMAIL_SUBJECT=<subject template>
```

---

## 6.7 Spring Cloud Config Server

The Loader supports centralized configuration via Spring Cloud Config, configured in `bootstrap.properties`:

```properties
spring.config.import=optional:configserver:
spring.cloud.config.fail-fast=true
spring.cloud.config.enabled=false
spring.profiles.active=native
```

Currently **disabled** (`enabled=false`). All configuration is read from local `application-ldr.properties`. When enabled, properties would be fetched from a remote Config Server before the application context starts.

---

## 6.8 Integration Summary

| Integration | Protocol | Direction | Required |
|---|---|---|---|
| IBM MQ (own QM) | JMS / TCP | Inbound + Outbound | Yes |
| IBM MQ (sibling QMs) | JMS / TCP | Outbound only | Yes |
| UBSR Oracle | JDBC | Outbound | Yes |
| RBM Oracle (8 nodes) | JDBC | Outbound | Yes |
| Cassandra | Native driver | Outbound | No (toggle) |
| JIS REST API | HTTP/REST | Outbound | Yes |
| DVS REST API | HTTP/REST | Outbound | Yes |
| DMD REST API | HTTP/REST | Outbound | Yes (device events) |
| AWS AppConfig | HTTPS | Outbound | No (supplements local flags) |
| Spring Cloud Config | HTTP/REST | Inbound | No (currently disabled) |
| SMTP Email | SMTP | Outbound | No (alerts only) |
