# Usage Broker (UB) Ecosystem — Technical Specification

## Part 3: Decoder Library — `jitr-ub-msg-decoder`

---

## 1. Library Identity

| Attribute           | Value                                          |
|---------------------|-------------------------------------------------|
| **Artifact ID**     | `jitr-ub-msg-decoder`                            |
| **Group ID**        | `com.verizon.jitr`                               |
| **Type**            | JAR Library (not a standalone application)        |
| **Consumers**       | `jitr-ub-msg` (Mediation Engine)                 |
| **Key Dependency**  | `jpos:1.9.0` (for BCD ↔ ASCII conversion)       |

---

## 2. Purpose

The Decoder Library solves the critical problem of **translating heterogeneous raw network usage records** into a uniform JSON representation. Network switches produce CDRs in various formats:

- **Binary BCD** (Binary Coded Decimal) — Lucent SMS, Motorola SMS
- **ASCII/CSV** — Motorola MMS
- **JSON streaming** — RCS
- **XML** — SMS (alternate format)

Each decoder reads the raw file format, extracts fields based on a layout map, converts binary fields to human-readable values, and produces a standardized JSON output for downstream processing.

---

## 3. Architecture

### 3.1 Class Hierarchy

```
UsageDecoder (Interface)
  │
  ├─ LucentSMSDecoder    ──── extends RawDecoder
  ├─ MotorolaSMSDecoder   ──── extends RawDecoder
  ├─ MotorolaMMSDecoder   ──── extends RawDecoder
  ├─ RCSDecoder           ──── extends RawDecoder
  └─ SMSDecoder           ──── (standalone, XML-based)
```

### 3.2 `UsageDecoder` Interface

```java
public interface UsageDecoder {
    String decode() throws Exception;    // Decode next record, return JSON string or null (EOF)
    void openFile() throws IOException;  // Open the input file for reading
    void closeFile() throws IOException; // Close file handles
    byte[] getRawRecord();               // Return raw bytes of last decoded record
}
```

### 3.3 `RawDecoder` — Binary Decoding Engine

The `RawDecoder` class provides two core decoding methods:

#### `decodeBinaryRecord(Map<String, Object> recordLayoutMap, byte[] inputRecord)`

Decodes **BCD-encoded binary records** using a JSON-defined field layout:

1. Iterates through the `recordLayoutMap` (field name → `{length, dataType, radix, paddedWith, external}`).
2. For each field:
   - Reads `length` bytes from the current offset using `ISOUtil.bcd2str()`.
   - Removes padding characters (`0` or `C`).
   - Applies radix conversion for integer fields.
   - Handles nested `external` JSON structures (e.g., `min_mdn`, `addrType`).
3. After fixed fields, processes **flexible Lucent modules** (module code `197`):
   - Reads module descriptor and converts EBCDIC → ASCII.
   - Maps descriptor codes to field names (e.g., `00010` → `addressInfo`, `00200` → `externalIdentifierInfo`).

#### `decodeASCIIRecord(Map<String, Object> recordLayoutMap, byte[] inputRecord, String delimiter)`

Decodes **delimited ASCII records** (used by MMS):

1. Splits input by delimiter (`,` for MMS).
2. Matches first field (record type) against layout map keys.
3. Maps positional indices to field names.

---

## 4. Decoder Implementations

### 4.1 `LucentSMSDecoder` — Binary BCD

| Attribute        | Value                                    |
|------------------|------------------------------------------|
| **Input Format** | Binary BCD with variable-length records  |
| **File Access**  | `RandomAccessFile` + `FileChannel`       |
| **Layout File**  | `lucentSMS.json` (classpath resource)    |
| **Starting Offset** | Byte 4 (skips file header)           |

**Decoding Flow:**

1. Opens file via `RandomAccessFile` in read mode.
2. Reads 2 bytes at current offset → parses record length (hex).
3. Reads 3 bytes at offset+5 → extracts structure code (sCode).
4. Validates sCode against known valid codes:
   - `01339` / `41339` — SMS records (143 / 200 bytes)
   - `09036` — BeginRecording (31 bytes, skipped)
   - `09038` — CLDSHeader (58 bytes, skipped)
   - `09039` — CLDSTrailer (53 bytes, skipped)
   - `09037` — EndRecording (41 bytes, skipped)
   - `09042` — AMAAudit (62 bytes, skipped)
5. For valid SMS records: calls `decodeBinaryRecord()` with Lucent layout map.
6. Enriches result:
   - Extracts `SwitchName` from filename (position `[5]`).
   - Normalizes `MsgDTM` (message date/time) from `deliveredDate` + `deliveredTime`.
   - Generates **GRI** (Global Record Identifier): `{ubPrefix}_{auditSeq}_{recordSeq}_{splitCount}_{cloneCount}`.

**LSM Date Normalization Logic:**

The Lucent SMS date format requires complex year derivation:
- Record contains only month+day+time (no year).
- Year is derived from the filename's file-stored timestamp.
- Edge cases handled:
  - Record month = Oct/Nov/Dec but file month = Jan/Feb/Mar → year - 1
  - Record month = Jan but file month = Dec → year + 1

### 4.2 `MotorolaSMSDecoder` — Binary BCD

| Attribute        | Value                                    |
|------------------|------------------------------------------|
| **Input Format** | Binary BCD, fixed 228-byte records       |
| **File Access**  | `RandomAccessFile` + `FileChannel`       |
| **Layout File**  | `motorolaSMS.json` (classpath resource)  |

**Decoding Flow:**

1. Reads 4 bytes → parses record length (hex).
2. Skips administrative records (70 bytes).
3. Validates record is exactly 228 bytes.
4. Calls `decodeBinaryRecord()` with Motorola SMS layout map.
5. Enriches:
   - Extracts `SwitchName` from filename.
   - Maps `serviceCodeStr` `22B8` → `serviceCode` `8888`.
   - Normalizes `MsgDTM` from `creationTime` + filename year.
   - **Validates record age** — rejects records older than `validDays` parameter.

### 4.3 `MotorolaMMSDecoder` — ASCII CSV

| Attribute        | Value                                    |
|------------------|------------------------------------------|
| **Input Format** | Comma-delimited ASCII (CSV)              |
| **File Access**  | `BufferedReader` + `FileReader`          |
| **Layout File**  | `motorolaMms.json` (classpath resource)  |

**Decoding Flow:**

1. Opens file with `BufferedReader`.
2. Reads line-by-line using `readLine()`.
3. Strips `~` characters from raw input.
4. Calls `decodeASCIIRecord()` with `,` delimiter.
5. Determines record type:
   - **MO1170** (Mobile Originated): Extracts `recipientAddresses` → creates consolidated single `recipientAddress` field. Generates GRI.
   - **MT1170** (Mobile Terminated): Passes through with GRI.
   - Other types: Filtered out (increments `filteredRecordCnt`).
6. Normalizes `MsgDTM` from `recordTimeStamp` (format: `HH:mm:ss:SSS MM/dd/yy`).

### 4.4 `RCSDecoder` — JSON Streaming

| Attribute        | Value                                    |
|------------------|------------------------------------------|
| **Input Format** | JSON array (`ASRecordS[]`)               |
| **File Access**  | Gson `JsonReader` (streaming)            |
| **Layout File**  | None (direct JSON parsing)               |

**Decoding Flow:**

1. Opens file with Gson `JsonReader` + custom `RcsRecordDeserializer`.
2. Reads JSON header: expects `{"ASRecordS": [...]}`.
3. Iterates array elements one-by-one (streaming, memory-efficient).
4. For each element:
   - Strips escaped quotes (RBMVD-15840 workaround).
   - Parses `serviceTimestamp` (ISO 8601: `yyyy-MM-dd'T'HH:mm:ssX`).
   - Generates `MsgDTM` in standard format.
   - Generates GRI.

**RCS MDN Parsing Logic** (in `RcsRecordDeserializer`):

Extracts 10-digit MDN from `subscriptionID`:
- `subscriptionIDType = 2`: Parse SIP URI — `sip:+{digits}@{domain}` → last 10 digits.
- `subscriptionIDType = 0`: Parse TEL URI — `1{10digits}` → extract 10 digits.
- Other types: Error.

### 4.5 `SMSDecoder` — XML DOM

| Attribute        | Value                                    |
|------------------|------------------------------------------|
| **Input Format** | XML with `<smsRecord>` elements          |
| **File Access**  | DOM `DocumentBuilder`                    |
| **Layout File**  | `SMS_CDR_2.2.xsd` (schema reference)    |

**Decoding Flow:**

1. Parses entire XML file into DOM using `DocumentBuilderFactory`.
2. Extracts all `<smsRecord>` nodes via `getElementsByTagName()`.
3. Iterates nodes, converting each to XML string using `Transformer`.
4. Returns XML string per record (not JSON — downstream handles conversion).

---

## 5. GRI (Global Record Identifier) Format

Every decoded record is assigned a unique GRI for traceability:

```
{ubPrefixCode}_{auditSequenceValue}_{recordSequenceNumber}_{splitCount}_{cloneCount}
```

Example: `UB_SDC_12345_1_0_0`

| Component              | Source                                    |
|------------------------|-------------------------------------------|
| `ubPrefixCode`         | Data center prefix (e.g., `UB_SDC`)       |
| `auditSequenceValue`   | From Oracle audit sequence                |
| `recordSequenceNumber` | Incrementing counter per file             |
| `splitCount`           | 0 for original, incremented for splits    |
| `cloneCount`           | 0 for original, incremented for clones    |

---

## 6. Layout Configuration Files

Decoder field mappings are defined in JSON resource files:

### `lucentSMS.json` Structure Example:
```json
{
  "serviceCode": {
    "length": 3,
    "dataType": "Integer",
    "radix": 16,
    "paddedWith": "C"
  },
  "deliveredDate": {
    "length": 3,
    "dataType": "String",
    "paddedWith": "0"
  },
  "min_mdn": {
    "addrType": { ... },
    "callingNumber": { "length": 5 },
    "calledNumber": { "length": 5 }
  }
}
```

Each field descriptor includes:
- `length` — Number of bytes to read
- `dataType` — `Integer` or `String`
- `radix` — Base for integer conversion (typically 16 for hex)
- `paddedWith` — Character to strip (`0`, `C`)
- `external` — Nested sub-structure for compound fields

### Lucent Flexible Module Codes:

| Code     | Field Name              | Description                                     |
|----------|-------------------------|-------------------------------------------------|
| `00010`  | `addressInfo`           | Email address of sender (MT) or recipient (MO) |
| `00110`  | `addressInfo`           | IP address of sender/recipient                  |
| `00200`  | `externalIdentifierInfo`| External ID (MSISDNless Premium)                |
| `00210`  | `mccMncInfo`            | MCC/MNC (CdrPlmnGta Premium)                    |
| `00211`  | `gtaInfo`               | GTA (CdrPlmnGta Premium)                        |
| `00201`  | `paniInfo`              | PANI (RoPaniHdr Premium)                        |

---

## 7. MMS Field Subset (Output Keys)

The MMS decoder outputs a normalized subset of fields:

```
attachmentTypes, auditID, contentType, copyDivert_billingID,
copyDivert_rcptIndicator, deliveryReportRequested, expirationTime,
gmtOffset, GRI, languageID, messageClass, messageSize,
messageStorageDuration, mmscID, mmStatusCode, MsgDTM,
numberDRMParts, numberDRMPartsBlocked, originalAddress,
originatorBillingID, originatorMIN, originatorMMSRelay,
originatorPrepaidClassIndicator, originatorServiceClass,
prepaidCost, prepaidCurrency, prepaidStatusCode, priority,
recipientAddress, recipientIP, recipientMIN,
recipientPrepaidClassIndicator, recipientServiceClass,
recordTimeStamp, recordType, replyCharging, replyChargingSize,
replyDeadline, sequenceNumber, subject, submissionTime,
SwitchName, transactionID, originatorAddress,
originalRecipientBillingIDs
```

---

*Continue to [Part 4: jitr-ub-msg (Core Mediation Engine)](04_Core_Mediation_jitr_ub_msg.md)*
