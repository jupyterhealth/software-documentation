---
title: Data Flow
description: Learn about JupyterHealth Exchange data flow
---

Understanding how health data moves through JupyterHealth Exchange—from its origin on a wearable device to its use in research analysis—reveals the platform's design philosophy: secure, consent-driven, and standards-based data sharing.

This document traces that journey step by step, explaining not just *what* happens but *why* each stage exists and how the pieces fit together.

## The Complete Journey

Health data flows through JupyterHealth Exchange across three major phases: **Acquisition**, **Exchange**, and **Analysis**. This sequence diagram shows **who** does **what** at each stage:

:::\{div}
:class: dark:hidden

```{mermaid}
%%{init: {'theme':'default'}}%%
sequenceDiagram
    participant R as Researcher
    participant P as Patient
    participant CH as CommonHealth App
    participant M as Manufacturer API
    participant JHE as JupyterHealth Exchange

    note over R,JHE: ACQUISITION PHASE
    R->>JHE: 1. Create patient & study
    JHE->>P: 2. Send invitation link
    P->>CH: 3. Open invitation
    CH->>JHE: 4. Authenticate patient
    JHE->>CH: 5. Return consent requirements
    P->>CH: 6. Grant consent
    CH->>JHE: 7. Upload consent
    P->>CH: 8. Connect manufacturer
    CH->>M: 9. Authenticate with manufacturer
    M->>M: 10. Device measures (e.g., glucose)
    CH->>M: 11. Sync data (proprietary format)
    CH->>CH: 12. Transform to IEEE 1752
    CH->>CH: 13. Wrap in FHIR Observation

    note over R,JHE: EXCHANGE PHASE
    CH->>JHE: 14. Upload FHIR Bundle
    JHE->>JHE: 15. Validate consent
    JHE->>JHE: 16. Store in database

    note over R,JHE: ANALYSIS PHASE
    R->>JHE: 17. Query FHIR API
    JHE->>R: 18. Return observations
    R->>R: 19. Analyze data
```

:::

:::\{div}
:class: hidden dark:block

```{mermaid}
%%{init: {'theme':'dark'}}%%
sequenceDiagram
    participant R as Researcher
    participant P as Patient
    participant CH as CommonHealth App
    participant M as Manufacturer API
    participant JHE as JupyterHealth Exchange

    note over R,JHE: ACQUISITION PHASE
    R->>JHE: 1. Create patient & study
    JHE->>P: 2. Send invitation link
    P->>CH: 3. Open invitation
    CH->>JHE: 4. Authenticate patient
    JHE->>CH: 5. Return consent requirements
    P->>CH: 6. Grant consent
    CH->>JHE: 7. Upload consent
    P->>CH: 8. Connect manufacturer
    CH->>M: 9. Authenticate with manufacturer
    M->>M: 10. Device measures (e.g., glucose)
    CH->>M: 11. Sync data (proprietary format)
    CH->>CH: 12. Transform to IEEE 1752
    CH->>CH: 13. Wrap in FHIR Observation

    note over R,JHE: EXCHANGE PHASE
    CH->>JHE: 14. Upload FHIR Bundle
    JHE->>JHE: 15. Validate consent
    JHE->>JHE: 16. Store in database

    note over R,JHE: ANALYSIS PHASE
    R->>JHE: 17. Query FHIR API
    JHE->>R: 18. Return observations
    R->>R: 19. Analyze data
```

:::

Let's examine each phase in detail.

______________________________________________________________________

# ACQUISITION PHASE

The acquisition phase encompasses everything from patient enrollment through data transformation, preparing standardized health measurements for exchange.

## Stage 1: Patient Enrollment

Before data can flow, patients must enroll and grant consent:

1. **Researcher creates patient record**: Adds patient to study in JHE
1. **Patient receives invitation**: Gets link to join study via email/SMS
1. **Patient opens CommonHealth**: Installs app if needed
1. **Patient reviews study**: Sees what data the study requests
1. **Patient grants consent**: Chooses which data types to share
1. **Patient connects manufacturer**: Links their device manufacturer account

**Result**: Patient is enrolled, consented, and ready to contribute data.

```{note}
For detailed consent mechanics, see [Understanding Consent Management](./consent-management.md).
```

## Stage 2: Device Measurement and Collection

### Device Measurements

Health data begins with a measurement:

- A glucose monitor reads blood sugar: **129 mg/dL**
- A blood pressure cuff measures: **122/77 mmHg**
- A smartwatch records heart rate: **72 bpm**

Each manufacturer stores data in its own proprietary format:

**iHealth format**:

```json
{
  "deviceId": "iHealth-BG5-ABC123",
  "timestamp": "2025-01-15T10:30:00-08:00",
  "reading": 129,
  "unit": "mg/dL",
  "mealContext": "before_breakfast"
}
```

**Dexcom format**:

```json
{
  "recordId": "dex_123456789",
  "egvs": [{
    "systemTime": "2025-01-15T18:30:00Z",
    "value": 129,
    "trend": "flat"
  }]
}
```

**The Problem**: Each manufacturer uses different field names, structures, and units. This fragmentation makes cross-device analysis difficult.

### CommonHealth Retrieves Data

The [CommonHealth Android App](https://play.google.com/store/apps/details?id=org.thecommonsproject.android.phr) retrieves device data from manufacturer APIs:

1. **Patient authorizes access**: Patient connects their manufacturer account to CommonHealth
1. **CommonHealth syncs data**: App periodically queries manufacturer's API for new measurements
1. **Data retrieved in proprietary format**: Each manufacturer returns data in their own structure

This approach works regardless of how data physically gets from the device to the manufacturer's cloud (Bluetooth, Wi-Fi, cellular, etc.).

## Stage 3: Transformation to IEEE 1752

### The Normalization Layer

This is where the CommonHealth application performs its crucial role: **format normalization**.

The app contains device-specific adapters (sometimes called "shims") that understand each manufacturer's format and know how to convert it to IEEE 1752 standard.

**For the iHealth glucose reading above, the adapter creates:**

```json
{
  "header": {
    "uuid": "abc-123-def-456",
    "schema_id": {
      "namespace": "omh",
      "name": "blood-glucose",
      "version": "4.0"
    },
    "source_creation_date_time": "2025-01-15T10:30:00-08:00",
    "modality": "sensed",
    "external_datasheets": [{
      "datasheet_type": "manufacturer",
      "datasheet_reference": "iHealth BG5"
    }]
  },
  "body": {
    "blood_glucose": {
      "value": 129,
      "unit": "mg/dL"
    },
    "effective_time_frame": {
      "date_time": "2025-01-15T10:30:00-08:00"
    },
    "temporal_relationship_to_meal": "fasting"
  }
}
```

**What Changed:**

- **Standardized structure**: Now follows IEEE 1752 header/body pattern
- **Consistent field names**: `blood_glucose` instead of `reading`
- **Rich metadata**: UUID, schema version, device provenance
- **Preserved context**: Meal relationship maintained
- **Timezone clarity**: Full ISO 8601 with offset

**Why This Matters**: Every glucose reading from every device now looks the same. A researcher's analysis code doesn't need to know whether data came from iHealth, Dexcom, or Abbott—it's all `omh:blood-glucose:4.0`.

### Benefits of Early Transformation

Why transform at the mobile app level rather than at the server?

**Pros**:

- **Offline capability**: Transformation works even without internet
- **Reduced server load**: Server doesn't need device-specific parsers
- **Data validation**: Catches errors before transmission
- **User feedback**: App can show standardized data to patient

**Cons**:

- **App complexity**: The CommonHealth application needs many shims
- **Update distribution**: New devices require app updates
- **Platform dependency**: Currently the CommonHealth application is Android only (web application currently in development)

## Stage 4: FHIR Envelope Creation

### Wrapping for Healthcare Interoperability

The CommonHealth app doesn't just send IEEE 1752 JSON to JupyterHealth Exchange, it wraps it in a FHIR Observation resource.

**The complete structure sent to JHE:**

```json
{
  "resourceType": "Observation",
  "status": "final",
  "code": {
    "coding": [{
      "system": "https://w3id.org/openmhealth",
      "code": "omh:blood-glucose:4.0",
      "display": "Blood Glucose"
    }]
  },
  "subject": {
    "reference": "Patient/12345"
  },
  "device": {
    "reference": "Device/70003"
  },
  "effectiveDateTime": "2025-01-15T10:30:00-08:00",
  "valueAttachment": {
    "contentType": "application/json",
    "data": "eyJoZWFkZXIiOnsiLi4uIn19"  // Base64-encoded IEEE 1752 JSON
  }
}
```

**Key Components**:

- **resourceType**: Declares this as a FHIR Observation
- **code.coding**: Identifies the data type using Open mHealth schema ID
- **subject**: References the patient this data belongs to
- **device**: References the source device
- **valueAttachment**: Contains the Base64-encoded IEEE 1752 data

### Why FHIR + IEEE 1752?

This hybrid approach serves multiple needs:

**FHIR Provides**:

- Healthcare system compatibility
- Standardized query patterns (`GET /Observation?patient=123`)
- Resource relationships (Patient → Observation → Device)
- EHR integration potential

**IEEE 1752 Provides**:

- Rich device data fidelity
- Wearable-specific field validation
- Community-driven schemas
- Preservation of all device context

**Together**: You can query for observations using FHIR APIs, but when you decode the data, you get full-fidelity device information.

______________________________________________________________________

# EXCHANGE PHASE

The exchange phase handles secure transmission, consent validation, and storage of health data within JupyterHealth Exchange.

## Stage 5: Secure Transmission

The FHIR Bundle containing our glucose observation is sent to JupyterHealth Exchange via HTTPS POST to the FHIR R5 endpoint. The request includes an OAuth bearer token for authentication and wraps the Observation resource in a FHIR Bundle of type "batch", allowing multiple resources to be sent in a single request.

**Security Layers**:

- **TLS 1.3**: Encryption in transit
- **OAuth 2.0 Bearer Token**: Proof of authorization
- **HTTPS only**: No unencrypted connections accepted
- **Rate limiting**: Prevents abuse

## Stage 6: Consent Validation

### The Gatekeeper

When JupyterHealth Exchange receives the FHIR Bundle, it immediately validates consent *before* accepting the data.

**Validation checks**:

1. **Is this patient enrolled in any active studies?**

   - Query: Does Patient 12345 have any StudyPatient records?

1. **Does the patient have active consent for this data type?**

   - Check: Blood glucose (`omh:blood-glucose:4.0`)
   - Query: Is there a ScopeConsent record for this patient + study + data type?

1. **Is the consent still valid?**

   - Check: Consent granted date ≤ now ≤ expiration date (if set)
   - Status: Has consent been revoked?

1. **Does the device match consented sources?**

   - Check: Is Device 70003 (iHealth BG5) an approved data source for this study?

**If all checks pass**: Data is accepted and stored.

**If any check fails**: HTTP 403 Forbidden response with specific error:

```json
{
  "resourceType": "OperationOutcome",
  "issue": [{
    "severity": "error",
    "code": "forbidden",
    "diagnostics": "Patient has not consented to share blood glucose data for this study"
  }]
}
```

**Why This Matters**: Consent validation at the API boundary ensures that *no data enters the system without explicit patient permission*. This isn't just good practice—it's a regulatory requirement under HIPAA and GDPR.

## Stage 7: Storage and Indexing

### Database Architecture

Once validated, JupyterHealth Exchange stores the observation in PostgreSQL:

The observation record includes a unique identifier, references to the patient and data type, a reference to the data source (device), the IEEE 1752 data stored as JSONB, a status field, and a timestamp.

**Key Design Decisions**:

**JSONB for Flexibility**:

- Stores IEEE 1752 data as native PostgreSQL JSONB
- Allows efficient querying: `WHERE value_attachment_data->'body'->'blood_glucose'->>'value' > '180'`
- Supports partial indexes on frequently queried fields

**Foreign Key Relationships**:

- `subject_patient_id` → Links to Patient
- `codeable_concept_id` → Links to data type taxonomy
- `data_source_id` → Links to Device/DataSource

**Audit Trail**:

- `last_updated` timestamp for every modification
- All access logged in separate audit table

### Indexing Strategy

JHE creates database indexes to accelerate common queries. Indexes are created on patient references, data types, and even within the JSONB data structure itself (such as glucose values and timestamps). These indexes allow researchers to quickly filter large datasets without scanning entire tables.

______________________________________________________________________

# ANALYSIS PHASE

The analysis phase provides researchers with secure API access to consented health data, enabling transformation of raw measurements into research insights.

## Stage 8: Data Access and Retrieval

### FHIR Query Patterns

Researchers access data through FHIR APIs using standard search parameters. They can filter observations by patient, data type (using the Open mHealth code), date ranges, sort by timestamp, and limit result counts. FHIR also supports advanced queries like finding observations for all patients in a study group.

### FHIR Response Format

JHE returns results as a FHIR Bundle containing a collection of Observation resources. Each Observation includes the data type code (Open mHealth schema), patient reference, device reference, timestamp, and the IEEE 1752 data encoded as Base64 in the `valueAttachment` field. Researchers decode this attachment to access the full structured health measurement.

## Stage 9: Analysis and Insight

### From Data to Knowledge

The final stage is where health data becomes health insight.

Researchers fetch observations from the FHIR API using authenticated requests, decode the Base64-encoded IEEE 1752 data from the FHIR `valueAttachment`, extract relevant fields (timestamp, glucose value, meal context), and perform statistical analysis using tools like pandas.

This analysis can calculate metrics like average glucose levels, count readings above clinical thresholds, and determine time-in-range percentages. This is the ultimate goal: turning raw device measurements into actionable clinical insights.

## Data Flow Variations

### Variation 1: EHR Data Upload

When data comes from an Electronic Health Record instead of a wearable:

1. **EHR FHIR API**: Hospital system exports via FHIR Bulk Data
1. **Already in FHIR**: Data is already FHIR Observations
1. **No IEEE 1752**: Clinical data uses native FHIR `valueQuantity`
1. **Same consent check**: Still validates patient consent
1. **Same storage**: Stored alongside wearable data

### Variation 2: Manual Data Entry

Patients can manually enter data through the web UI:

1. **Web form**: Patient logs into JHE portal
1. **Guided entry**: Form validates units and ranges
1. **IEEE 1752 generation**: UI creates IEEE 1752 structure
1. **FHIR wrapping**: Converts to Observation resource
1. **Standard flow**: Follows same consent/storage path

### Variation 3: Research Device Integration

For specialized research devices without commercial adapters:

1. **Custom adapter**: Research team builds device-specific code
1. **IEEE 1752 output**: Adapter outputs standardized format
1. **Direct API upload**: Uses JHE FHIR API
1. **Community contribution**: Adapter can be shared with others

## Why This Architecture?

### Design Principles Reflected

**Separation of Concerns**:

- Device integration (CommonHealth)
- Data standardization (IEEE 1752)
- Healthcare compatibility (FHIR)
- Consent management (JHE)
- Analysis (Researcher tools)

Each component has a clear job, making the system maintainable and extensible.

**Defense in Depth**:

- TLS encryption in transit
- OAuth authorization
- Consent validation
- Database access controls
- Audit logging

Multiple security layers protect patient data.

**Standards-Based Integration**:

- FHIR for healthcare systems
- IEEE 1752 for device data
- OAuth 2.0 for authorization
- REST for APIs

Using standards enables ecosystem growth.

## Learn More

**Related Documentation**:

- [Why FHIR? Interoperability Explained](./fhir-interoperability.md) - Deep dive on FHIR
- [IEEE 1752 Data Standards](./openmhealth-standards.md) - Device data schemas
- [Understanding Consent Management](./consent-management.md) - How consent works
- [API Reference](../reference/exchange-apis.md) - Technical API details

**External Resources**:

- [FHIR Observation Resource](https://www.hl7.org/fhir/observation.html)
- [IEEE 1752 Standard](https://standards.ieee.org/ieee/1752.1/6982/)
- [OAuth 2.0 Specification](https://oauth.net/2/)

## Summary

Data flow through JupyterHealth Exchange is organized into three phases:

### ACQUISITION PHASE

Collecting and standardizing health data from diverse sources.

- **Stage 1: Patient Enrollment** → Consent establishment
- **Stage 2: Device Measurement and Collection** → Device readings and retrieval
- **Stage 3: IEEE 1752 Transformation** → Format normalization
- **Stage 4: FHIR Envelope Creation** → Healthcare compatibility

**Output**: Standardized, FHIR-wrapped health observations ready for transmission.

### EXCHANGE PHASE

Securely transmitting and storing consented health data.

- **Stage 5: Secure Transmission** → HTTPS + OAuth protection
- **Stage 6: Consent Validation** → Permission enforcement
- **Stage 7: Storage and Indexing** → PostgreSQL persistence

**Output**: Validated, indexed health data available for authorized access.

### ANALYSIS PHASE

Enabling researchers to derive insights from consented data.

- **Stage 8: Data Access and Retrieval** → FHIR API queries
- **Stage 9: Analysis and Insight** → Research findings

**Output**: Clinical insights and research discoveries from aggregated health measurements.

______________________________________________________________________

Each phase serves a specific purpose, balancing the needs of:

- **Patients**: Privacy, control, and informed consent
- **Researchers**: Access, standardization, and analytical power
- **Healthcare Systems**: Interoperability, compliance, and auditability

The result: wearable data that's secure, consented, standardized, and ready for research—transforming individual measurements into collective knowledge.
