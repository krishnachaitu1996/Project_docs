# Section 3 — Event Type Catalog

**Application:** JITR Loader 2.0  
**Navigation:** [← Transaction Flow](./02_transaction_flow.md) | [Index](./README.md) | [Data Operations →](./04_data_operations.md)

---

## 3.1 Overview

The Loader processes three categories of events:

| Category | How Identified | Handler Location |
|---|---|---|
| **CPF Events** | XML element type (e.g., `<ACTIVATE>`, `<SUSPEND>`) | `loader.events.cpf` |
| **DataPop Events** | `<DATAPOP><BODY><TABLENAME>` field value | `loader.events.datapop` |
| **BillingVision Events** | DataPop with `TABLE_NAME` pointing to a Vision 2.0 billing table | `loader.events.datapop.billingVision` |

All events are identified by `EventProcessHelper.getEDRCPFMessage()` using `instanceof` checks on the JAXB-unmarshalled `Root` object.

---

## 3.2 CPF Events (16 XSD-Defined Root Types)

These events represent subscriber lifecycle changes triggered by CRM/provisioning systems. Each maps to a single XML element inside `<Root>`, defined in `InputSchema.xsd`.

| # | XML Element | XSD Type | Process Name Constant | Handler Bean | Description |
|---|---|---|---|---|---|
| 1 | `<ACTIVATE>` | `ActivateType` | `PROCESS_NAME_ACTIVATE` | `ActivateEvent` | New line activation |
| 2 | `<CHANGE_DATA_FEATURES>` | `ChangeDataFeaturesType` | `PROCESS_NAME_CHANGEDATAFEATURES` | `ChangeEvent` | Data feature changes (add/remove) |
| 3 | `<CHANGE_SFO_FEATURES>` | `ChangeSfoFeaturesType` | `PROCESS_NAME_CHANGESFOFEATURES` | `ChangeSfoEvent` | Special Feature Offer changes (SFO-level lock) |
| 4 | `<REASSIGN>` | `ReassignType` | `PROCESS_NAME_REASSIGN` | `ReassignEvent` | Line reassignment between accounts |
| 5 | `<TRANSFER>` | `TransferType` | `PROCESS_NAME_TRANSFER` | `TransferEvent` | Customer/account transfer (carries `PRICEPLANCONTRACTTERMID`) |
| 6 | `<CHANGE_BILL_DAY>` | `ChangeBillDayType` | `PROCESS_NAME_CHANGEBILLDAY` | ChangeBillDay handler | Bill cycle day change |
| 7 | `<RECONNECT>` | `ReconnectType` | `PROCESS_NAME_RECONNECT` | `ReconnectEvent` | Service reconnection after suspension |
| 8 | `<SUSPEND>` | `SuspendType` | `PROCESS_NAME_SUSPEND` | `SuspendEvent` | Service suspension |
| 9 | `<DEACTIVATE>` | `DeactivateType` | `PROCESS_NAME_DEACTIVATE` | `DeactivateEvent` | Line deactivation |
| 10 | `<DO_NOT_SOLICIT>` | `DoNotSolicitType` | `PROCESS_NAME_DO_NOT_SOLICIT` | Dedicated handler | Marketing opt-out flag |
| 11 | `<INQUIRY>` | — | `PROCESS_NAME_INQUIRY` | Dedicated handler | Read-only inquiry |
| 12 | `<SIM_OVERTHEAIR_DETECTION>` | `SimOverTheAirDetectionType` | `PROCESS_NAME_SIM_OVERTHEAIR` | Dedicated handler | SIM OTA detection event |
| 13 | `<REFRESH>` | `RefreshType` | `PROCESS_NAME_REFRESH` | `RefreshEvent` | Data refresh/re-sync |
| 14 | `<DATAPOP>` | `DataPop` | `PROCESS_NAME_DATAPOP` | *See Section 3.3/3.4* | Bulk data population |
| 15 | `<ACCOUNT_ORDER_ACTIVITY>` | `AccountOrderActivityType` | `PROCESS_NAME_ACCOUNT_ORDER_ACTIVITY` | Dedicated handler | Account order activity tracking |
| 16 | `<ACCOUNT_ORDER_SNAPSHOT>` | `AccountOrderSnapshotType` | `PROCESS_NAME_ACCOUNT_ORDER_SNAPSHOT` | Dedicated handler | Account order snapshot |

### CPF Event Common Fields

All CPF events include:
- **`TRKEY`** — Transaction key block containing `TRFULFILLMENTTIME` and `TRMDN`
- **`CUSTOMERID`** — Customer identifier (used for locking and routing)
- **`ACCOUNTNUMBER`** — Account number
- **`LINEOFSERVICEIDPART1` / `LINEOFSERVICEIDPART2`** — Line-of-service identifiers

> **Note on leading zeros (V2DT41462):** When feature flag `V2DT41462=Y`, leading zeros are stripped from `LINEOFSERVICEIDPART1` and `LINEOFSERVICEIDPART2` for consistent `LINE_OF_SERVICE` computation. Applies to ACTIVATE, REASSIGN, CHANGE_DATA_FEATURES, CHANGE_SFO_FEATURES, and TRANSFER.

---

## 3.3 DataPop Events — Subscriber Data (~22 handlers)

DataPop events carry database change notifications. The `TABLE_NAME` field inside `<BODY>` identifies which handler processes the event. The `OPERATION` field indicates INSERT, UPDATE, or DELETE.

### DataPop Body Structure

```xml
<DATAPOP>
  <TRPOPKEY>
    <TRFULFILLMENTTIME>...</TRFULFILLMENTTIME>
    <TRMDN>2015551234</TRMDN>
    <TRCUSTOMERID>12345</TRCUSTOMERID>
    <TRACCOUNTNUMBER>67890</TRACCOUNTNUMBER>
  </TRPOPKEY>
  <BODY>
    <TABLENAME>CUST_DVC_PROV_INFO</TABLENAME>
    <OPERATION>INSERT</OPERATION>
    <KEYS>
      <KEY name="LN_OF_SVC_ID_NO_P1">value</KEY>
    </KEYS>
    <DB_FIELDS>
      <FIELD name="DEVICE_ID">IMEI123</FIELD>
    </DB_FIELDS>
  </BODY>
</DATAPOP>
```

### Standard DataPop Handlers

| TABLE_NAME | Handler Class | Description |
|---|---|---|
| Account-related | `AccountEvent` | Customer account master updates |
| Account MDN price plan | `AccountMdnPplanEvent` | Account-level MDN pricing assignments |
| Account service product | `AccountServiceProdEvent` | Account-associated service products |
| Account SPLAN | `AccountSplanEvent` | Account service plan records |
| Account SPLAN inactive | `AccountSplanInactiveEvent` | Inactive service plan cleanup |
| Bill cycle | `BillCycleEvent` | Bill cycle date changes |
| Billing account promo | `BillingAccountPromoEvent` | Account-level promotional assignments |
| Billing account svc prod | `BillingAccountServiceProdEvent` | Billing service product records |
| Customer | `CustomerEvent` | Customer master record updates |
| Customer account MDN | `CustAcctMDNEvent` | MDN-to-account assignment changes |
| Customer line price plan | `CustLnPplanEvent` | Line-level price plan changes |
| Customer MTN promo | `CustMtnPromoEvent` | MTN-level promotional data |
| ECPD ID profile | `EcpdIdProfileEvent` | ECPD (Enterprise Customer Profile Data) updates |
| Line service product | `LineServiceProdEvent` | Line-level service product records |
| Line price guarantee | `LnPriceGuaranteeEvent` | Price lock/guarantee records |
| MTDT migration update | `MtdtMigrationUpdateEvent` | Migration metadata updates |
| MTDT routing update | `MtdtRoutingUpdateEvent` | Customer routing record updates |
| Previous customer MTN | `PrevCustMTNEvent` | Historical MDN records |
| Sub SU allocation | `SubSuAllocationEvent` | Shared usage (SU) allocation records |
| Text MC | `TextMCEvent` | Text messaging market cluster |
| Text MC list | `TextMCListEvent` | Market cluster list updates |
| Usage segmentation | `UsageSegmentationEvent` | Usage segmentation rule updates |

### Special DataPop Key Extraction (V2DT41569)

When `V2DT41569=Y`, the following fields are extracted from `KEYS` first (falling back to `DB_FIELDS` if absent):

| Field | Message Source | EDRCPFMessage Field |
|---|---|---|
| `LN_OF_SVC_ID_NO_P1` | KEYS → DB_FIELDS | `lineOfServiceP1` |
| `LN_OF_SVC_ID_NO_P2` | KEYS → DB_FIELDS | `lineOfServiceP2` |
| `ADDR_ID_NO` | KEYS → DB_FIELDS | `addrIdNo` |
| `CBR_PERSON_ID_NO` | KEYS → DB_FIELDS | `cbrPersonIdNo` |
| `ECPD_PROFILE_ID` | KEYS → DB_FIELDS | `ecpdId` |

### Specific DataPop Behaviors

| TABLE_NAME | Special Handling |
|---|---|
| `CUST_DVC_PROV_INFO` | Extracts `DEVICE_ID` from DB_FIELDS; includes device ID in lock name |
| `CUST_LN_TX_GEO_CD` | Extracts `LN_OF_SVC_ID_NO_P1/P2` from KEYS block |
| `CUST_DVC_EQP_TRANS` | MDN-level locking (when MDN_LEVEL_LOCK=Y) |

---

## 3.4 BillingVision DataPop Events (~40 handlers)

These handle Vision 2.0 billing-specific datapop events. They write to billing subsystem tables and are only processed when the customer is identified as a Vision 2.0 customer.

| Handler Class | Billing Area | Description |
|---|---|---|
| `BillingCustMtnEvent` | Customer MTN | Customer mobile number billing records |
| `BillingEcpdProfileEvent` | ECPD | ECPD billing profile records |
| `BillingEcpdRateExemptEvent` | ECPD | Rate exemption records |
| `BillingEcpdTaxExemptEvent` | ECPD | Tax exemption records |
| `BillingEcpdCustPromoOvrEvent` | ECPD | Customer promo override |
| `BillingEcpdProfilePtfRuleEvent` | ECPD | Platform rule assignments |
| `BillingLineServiceProdEvent` | Line billing | Line-level billing service products |
| `BillingLineServiceProdAttrEvent` | Line billing | Line service product attributes |
| `BillingBlAcctSvcProdAttrEvent` | Account billing | Account-level product attributes |
| `BillingAccountSplanEvent` | SPLAN | Billing service plan records |
| `BillingAccountSplanInactiveEvent` | SPLAN | Inactive billing service plans |
| `BillingAccountMdnPplanEvent` | Price plan | MDN-level pricing in billing system |
| `BillingUsageSegmentationEvent` | Usage | Billing usage segmentation |
| `BillingCustMtnPromoEvent` | Promotions | Customer MTN promotional data |
| `BillingCustLnPplanEvent` | Price plan | Customer line price plan |
| `BillingCustCustTypeEvent` | Customer type | Customer type classification |
| `AddressRepoEvent` | Address | Address repository records |
| `CustAddressEvent` | Address | Customer address records |
| `BillingSecurityDepositEvent` | Security deposit | Security deposit records |
| `BillingSecDepAcctTypEvent` | Security deposit | Security deposit account type |
| `BillingVolteBsnsLnGrpXrefEvent` | VoLTE | VoLTE business line group cross-reference |
| `BillingLnSvcBenefitPairStat` | Benefits | Service benefit pair status |
| `BillingBASpoParticipateLNEvent` | SPO | Billing account SPO participation lines |
| `BillingAccountAddressEvent` | Address | Billing account address |
| `BaExemptTmplEvent` | Exemptions | Billing account exemption templates |
| `BaSpclBlFormatEvent` | Billing format | Special billing format |
| `BaBtMedSuppressEvent` | Suppression | Bill media suppression |
| `BlAcctBlTypEvent` | Account type | Billing account type |
| `BlAcctPmtStatEvent` | Payment | Billing account payment status |
| `BlAcctSpclMsgEvent` | Messages | Special billing messages |
| `BlAcctPmtStatEvent` | Payment | Payment status records |
| `BlEisOrdMTNAdminEvent` | EIS orders | EIS order MTN admin |
| `BlEisOrdMtnUbiEvent` | EIS orders | EIS order MTN UBI |
| `BlSlcBlAcctEvent` | SLC | SLC billing account |
| `SlcBlAcctPPOvrCompEvent` | SLC | SLC price plan override component |
| `CustDefinedInfoEvent` | Custom | Customer-defined info fields |
| `CustLnDefiendInfo` | Custom | Line-level defined info |
| `CustMtnLnTxGeoCdEvent` | Geo | MTN line transaction geo code |

---

## 3.5 Locking Granularity by Event Type

| Event Type | Lock Level | Lock Key Example |
|---|---|---|
| ACTIVATE | Customer + Account + MDN | `ACTIVATE_12345_67890_2015551234` |
| CHANGE_DATA_FEATURES | Customer + Account + MDN | `CHANGE_DATA_FEATURES_12345_67890_2015551234` |
| CHANGE_SFO_FEATURES | Customer + Account + MDN + SFO_ID | `CHANGE_SFO_FEATURES_12345_67890_2015551234_500` |
| SUSPEND | Customer + Account + MDN | `SUSPEND_12345_67890_2015551234` |
| RECONNECT | Customer + Account + MDN | `RECONNECT_12345_67890_2015551234` |
| DEACTIVATE | Customer + Account + MDN | `DEACTIVATE_12345_67890_2015551234` |
| DataPop (general) | Customer + Account + MDN | `ACCOUNT_12345_67890_2015551234` |
| CUST_DVC_PROV_INFO | Customer + Account + MDN + Device ID | `CUST_DVC_PROV_INFO_12345_67890_2015551234_IMEI123` |
| CUST_DVC_EQP_TRANS | Customer + Account + MDN | `CUST_DVC_EQP_TRANS_12345_67890_2015551234` |
| MTN-type DataPops | Customer + Account only | `MTN_TYPE_12345_67890` |
| No customer ID | Transaction ID | `NOCUST_TXN123456` |
| Null MDN + V2DT15112=Y | Transaction ID (if in list) | `ACCOUNT_12345_67890_TXN123456` |

---

## 3.6 OR DataPop List

The `ORDataPopList` property defines DataPop types that require an additional Vision 2.0 customer check before processing. These datapops are only processed if either:
- `visionFlagEnable=ON`, or  
- The customer is identified as a Vision 2.0 customer

If neither condition is met, the event is silently skipped with a debug log entry: `"OR Datapop {type} received for non vision customer"`.
