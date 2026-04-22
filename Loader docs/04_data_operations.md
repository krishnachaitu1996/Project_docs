# Section 4 — Data Operations (DataOps)

**Application:** JITR Loader 2.0  
**Navigation:** [← Event Types](./03_event_type_catalog.md) | [Index](./README.md) | [Messaging & Caching →](./05_messaging_and_caching.md)

---

## 4.1 Database Topology Overview

The Loader writes to three database backends:

```
┌────────────────────────────────────────────────────┐
│                    UBSR Oracle                      │
│  (Single instance — schema UBSRV2)                 │
│  100+ subscriber tables                             │
│  ~70 Spring Data JPA repositories                   │
│  HikariCP connection pool                           │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│          RBM Oracle (8 sharded nodes)              │
│  Schema: RB_CUSTOM                                  │
│  Routing: MultitenantDataSource + TenantContext     │
│  Sharding key: PVDOMAIN_GROUP_4.DOMAIN_GROUP_ID     │
│  Tenant keys: "rbmNode1" ... "rbmNode8"             │
│  Separate HikariCP pool per node                    │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│               Cassandra Cluster                     │
│  Toggle: VISION_2_0_CASSANDRA_ENABLE = ON/OFF       │
│  Driver: DataStax Java Driver 3.7.2                 │
│  Purpose: Stores original XML for reflow recovery   │
│  Fallback: Oracle JITR_LDR_TRX table (when OFF)    │
└────────────────────────────────────────────────────┘
```

---

## 4.2 UBSR Oracle — Primary Subscriber Database

The UBSR (UBSRV2 schema) Oracle database is the primary persistence store for all subscriber state and processing audit records.

### Table Categories

#### Processing Audit Tables (granted in `grants.sql`)

| Table | Purpose |
|---|---|
| `JITR_LDR_VB_SUCCESS` | Records for every successfully processed transaction |
| `JITR_LDR_VB_ERRORS` | Error records for failed-but-retryable transactions |
| `JITR_LDR_VB_UNPROCESSED_ERRORS` | Terminal error records (retries exhausted) |
| `JITR_LDR_TRX` | Transaction log; fallback XML storage for reflow |
| `JITR_REF_RECON_ON_DEMAND` | On-demand reconciliation records (ODR) |
| `SUB_ADDR_REPOS` | Subscriber address repository |
| `SUB_XLT_BL_RBM_BILLING` | UBSR↔RBM billing cross-reference |

#### Customer & Account Tables

| Table | Entity DTO | Purpose |
|---|---|---|
| `SUB_CUSTOMER` | `SubCustomer` | Master customer record |
| `SUB_CUST_ACCT` | `SubCustAcct` | Customer account |
| `SUB_CUST_ACCT_MDN` | `SubCustAcctMdn` | MDN-to-account mapping |
| `SUB_CUST_ACCT_LN_PPLAN` | `SubCustAcctLnPplan` | Line price plan |
| `SUB_CUST_ACCT_LN_SPO` | `SubCustAcctLnSpo` | Line special offers |
| `SUB_CUST_ACCT_LOS_SFO` | `SubCustAcctLosSfo` | Line-of-service SFO |
| `SUB_CUST_ACCT_LOS_SPO` | `SubCustAcctLosSpo` | Line-of-service SPO |
| `SUB_CUST_ACCT_LOS_PPLAN` | `SubCustAcctLosPplan` | Line-of-service price plan |
| `SUB_CUST_BL_CYCLE` | `SubCustBlCycle` | Bill cycle records |
| `SUB_CUST_ECPD_PROFILE` | `SubCustEcpdProfile` | ECPD profile |
| `SUB_CUST_NON_JITR` | `SubCustNonJITR` | Non-JITR customer marker |
| `SUB_CUST_RETURN_TO_VISION` | `SubCustReturnToVision` | Return-to-Vision migration marker |

#### Line & Service Tables

| Table | Entity DTO | Purpose |
|---|---|---|
| `SUB_ACCT_SVC_PROD` | `SubAcctSvcProd` | Account service products |
| `SUB_ACCT_SVC_PROD_ATTR` | `SubAcctSvcProdAttr` | Service product attributes |
| `SUB_ACCT_SPLAN` | `SubAcctSplan` | Account service plans |
| `SUB_ACCT_SPLAN_INACTIVE` | `SubAcctSplanInactive` | Inactive service plans |
| `SUB_BL_ACCT_LN_STAT` | `SubBlAcctLnStat` | Billing account line status |
| `SUB_LN_OF_SVC_CUST_BA` | `SubLnOfSvcCustBa` | Line-of-service to billing account |
| `SUB_LN_PRIM_ID_MDN` | `SubLnPrimIdMdn` | Primary line ID MDN |
| `SUB_LN_SVC_PROD_USG_SEG_BA` | `SubLnSvcProdUsgSegBa` | Line usage segmentation |
| `SUB_SF_MDN` | `SubSfMdn` | Service feature MDN |
| `SUB_CUST_ACCT_MC` | `SubCustAcctMc` | Account market cluster |
| `SUB_CUST_ACCT_MC_LIST` | `SubCustAcctMcList` | Account MC list |

#### Device & SIM Tables

| Table | Entity DTO | Purpose |
|---|---|---|
| `SUB_CUST_DVC_EQP_TRANS` | `SubCustDvcEqpTrans` | Customer device equipment transactions |
| `SUB_DVC_SIM_EQP_ASSN` | `SubDvcSimEqpAssn` | Device SIM equipment assignment |

#### Request Queue Tables

| Table | Entity DTO | Purpose |
|---|---|---|
| `SUB_RERATE_REQUEST` | `SubRerateRequest` | Rerate requests pending processing |
| `SUB_PRORATE_REQUEST` | `SubProrateRequest` | Prorate requests |
| `SUB_SU_ALLOCATION` | `SubSuAllocation` | Shared usage allocation |
| `SUB_PENDING_CYCLE_CHANGE` | `SubPendingCycleChange` | Pending bill cycle changes |

#### Reference & Lookup Tables (Cached)

| Table | Entity DTO | Cached As |
|---|---|---|
| `REF_VIS_XLT_RATING_TRANS_MAP` | `RefVisXltRatingTransMap` | TRANSMAP cache |
| `REF_VIS_RATING_PROD_HIER` | `RefVisRatingProdHier` | PRODHIER cache |
| `REF_VIS_RATING_TARIFF` | (tariff ref) | TARIFFID cache |
| `REF_VISION_INSTANCE_LKUP` | `RefVisionInstanceLkup` | REF_VISION_INSTANCE_LKUP cache |
| `REF_VIS_SPLAN` | `RefVisSplan` | From JIS/repository |
| `REF_VIS_SF_OFFER` | `RefVisSfOffer` | SFO offer details |
| `REF_VIS_PPLAN` | `RefVisPplan` | Price plan reference |
| `REF_CONFIG_PARAMETERS` | `RefConfigParameters` | System configuration |
| `REF_VIS_JITR_UNSUPPORT_OFFR` | `RefVisJitrUnsupportOffr` | Unsupported offer list |

#### Routing & Migration Tables

| Table | Entity DTO | Purpose |
|---|---|---|
| `MTDT_JITR_ROUTING` | `MtdtJitrRouting` | Customer→instance routing |
| `MTDT_JITR_MIGRATION` | `MtdtJitrMigration` | Migration status |
| `MTDT_JITR_MIGRATION_HISTORY` | `MtdtJitrMigrationHistory` | Migration history |

#### Soft-Delete Pattern (`*Delrows` tables)

The Loader uses a soft-delete pattern extensively. Many tables have a companion `*Delrows` variant:

| Active Table | Delete-Marker Table |
|---|---|
| `SUB_ACCT_SPLAN` | `SUB_ACCT_SPLAN_DELROWS` |
| `SUB_ACCT_SVC_PROD` | `SUB_ACCT_SVC_PROD_DELROWS` |
| `SUB_CUST_ACCT_MDN` | `SUB_CUST_ACCT_MDN_DELROWS` |
| `SUB_CUST_BL_CYCLE` | `SUB_CUST_BL_CYCLE_DEL_ROWS` |
| `SUB_CUST_DVC_EQP_TRANS` | `SUB_CUST_DEV_EQP_TRANS_DELROWS` |
| `SUB_SF_MDN` | `SUB_SF_MDN_DELROWS` |
| `SUB_PREV_CUST_MDN` | `SUB_PREV_CUST_MDN_DELROWS` |
| `SUB_XLT_BL_RBM_BILLING` | `SUB_XLT_BL_RBM_BILLING_DELROWS` |
| ... | ... |

Instead of `DELETE FROM table WHERE key=?`, the Loader writes a marker row to the `*DELROWS` table indicating the record was logically removed.

---

## 4.3 RBM Oracle — Billing Attribute Database (8 Sharded Nodes)

The RBM (RB_CUSTOM schema) database holds Vision billing attributes. It is horizontally sharded across 8 Oracle nodes, with routing transparent to event handlers.

### RBM Tables (granted in `RBM_Grants.sql`)

| Table | Purpose |
|---|---|
| `VZACCTATTRDETAILS` | Account-level attribute details |
| `VZECPDATTRDETAILS` | ECPD attribute details |
| `VZCUSTOVERRIDEBILLDISCOUNT` | Customer override billing discounts |
| `VZCUSTMTN` | Customer MTN records |
| `VZMTNSTATUS` | MTN status records |
| `VZPRODUCTSTATUS` | Product status |
| `CustProductDetails` | Product detail records |
| `CustProductTariffDetails` | Product tariff details |
| `CustProdAddonRateDetails` | Product add-on rate details |
| `Account` | Account master |
| `Accountattributes` | Account attribute details |
| `PvdomainGroup4` | Domain-to-shard mapping (used for routing) |
| `CustEventSource` | Customer event source |
| `CustProductStatus` | Customer product status |

### Multi-Tenant Routing Mechanism

```
1. Event processing begins for customer X
   │
2. RbmDao.setTenantForCustomer(domainGroupId)
   │   Queries PvdomainGroup4 for customer's domain group
   │   Maps domainGroupId → shardIndex (rbmNode1 through rbmNode8)
   │   Calls: TenantContext.setCurrentTenant("rbmNode3")
   │
3. MultitenantDataSource.determineCurrentLookupKey()
   │   Returns TenantContext.getCurrentTenant()
   │   Spring routes all JPA calls to rbmNode3 DataSource
   │
4. SQL executes on rbmNode3 Oracle connection
   │
5. TenantContext.clearTenant()
   └─ ThreadLocal cleared for next request
```

**Class responsibilities:**

| Class | Responsibility |
|---|---|
| `MultitenantDataSource` | Extends `AbstractRoutingDataSource`; selects DataSource based on ThreadLocal |
| `TenantContext` | ThreadLocal `String` holder — `setCurrentTenant()`, `getCurrentTenant()`, `clearTenant()` |
| `RbmDao` | Computes shard key from `domainGroupId`; sets tenant context; executes RBM queries via `NamedParameterJdbcTemplate` |

---

## 4.4 ATTRIBUTE_CONFIG Table

The `UBSRV2.ATTRIBUTE_CONFIG` table, seeded by `attribute_config_insert.sql` and `R2.4_insert.sql`, maps logical attribute names to database table names. This drives dynamic datapop field mapping without code changes.

Example entries from seed data:
```sql
INSERT INTO ATTRIBUTE_CONFIG (ATTR_NAME, TABLE_NAME) VALUES ('CUST_ACCT_SPLAN', 'SUB_ACCT_SPLAN');
INSERT INTO ATTRIBUTE_CONFIG (ATTR_NAME, TABLE_NAME) VALUES ('CUST_ACCT_MDN', 'SUB_CUST_ACCT_MDN');
```

---

## 4.5 UBSR Repositories (~70+)

All Spring Data JPA repositories for UBSR are in `dao.ubsr.repository`. Each entity has a corresponding repository:

**Core repositories:**
- `SubCustomerRepository`
- `SubCustAcctRepository`
- `SubCustAcctMdnRepository`
- `SubAcctSvcProdRepository`
- `SubAcctSplanRepository`
- `JitrLdrErrorsRepository`
- `JitrLdrSuccessRepository`
- `JitrLdrUnprocessedErrorsRepository`
- `MtdtJitrRoutingRepository`
- `RefVisionInstanceLkupRepository`
- `RefVisRatingProdHierRepository`
- `RefVisXltRatingTransMapRepository`

**Delrows repositories (soft-delete markers):**
- `SubAcctSplanDelrowsRepository`
- `SubAcctSvcProdDelrowsRepository`
- `SubCustAcctMdnDelrowsRepository`
- `SubCustBillCycleDelRowsRepository`
- `SubCustDevEqpTransDelrowsRepository`
- `SubSfMdnDelrowsRepository`
- `SubPrevCustMdnDelrowsRepository`
- `SubXltBlRbmDelrowsRepository`
- ... and more

**Reference/lookup repositories:**
- `RefVisSplanRepository`
- `RefVisPplanRepository`
- `RefVisSfOfferRepository`
- `RefConfigParametersRepository`
- `RefVisRateCampaignInfoRepository`
- `RefVisJitrUnsupportOffrRepository`
- `RefVisNonGeoRepository`

**ILB/audit repositories:**
- `JitrLdrIlbauditRepository`
- `JitrIlbProcessRepository`
- `BlCycPhaseAuditRepository`
- `DmdAuditInfoRepository`
- `VisionCyclePhaseActivityRepository`

---

## 4.6 Cassandra Configuration

Cassandra is used as a high-performance store for original XML message payloads to support the reflow pipeline.

**Configuration properties:**
```properties
cassandra.contactPoints=<host1,host2,...>
cassandra.port=9042
cassandra.keyspace=<keyspace_name>
```

**Toggle:** `VISION_2_0_CASSANDRA_ENABLE=ON` enables Cassandra writes. When `OFF`, the Oracle `JITR_LDR_TRX` table is used as the fallback XML store.

**Usage pattern:**  
When an event fails and requires reflow, `ReflowScheduler` calls the Cassandra DAO to retrieve the original XML payload using the transaction ID as the lookup key, rather than re-fetching from Oracle.

---

## 4.7 Connection Pool Configuration

HikariCP is configured for all Oracle DataSources.

### UBSR Pool
```properties
spring.datasource.hikari.maximum-pool-size=100
spring.datasource.hikari.minimum-idle=20
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
```

### RBM Pools
Each of the 8 RBM nodes (`rbm.datasource.node1` through `rbm.datasource.node8`) has its own independent HikariCP pool configured with similar settings. They share no connections across nodes.
