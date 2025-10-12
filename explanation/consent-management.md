---
title: Consent Management
description: Learn how JupyterHealth Exchange implements granular patient consent for research data sharing
---

# Understanding Consent Management

Patient consent is the cornerstone of ethical health data research. JupyterHealth Exchange implements a sophisticated consent management system that gives patients fine-grained control over what data they share, with whom, and for what purposes.

This document explains the consent model, how it works in practice, and why it's designed the way it is.

## Why Granular Consent Matters

Traditional consent models often use an "all or nothing" approach: patients either share all their health data with a research study or none at all. This creates problems:

- **Privacy concerns**: Patients may be comfortable sharing step counts but not glucose measurements
- **Reduced participation**: Overly broad consent requirements discourage enrollment
- **Regulatory challenges**: GDPR and HIPAA emphasize data minimization and purpose limitation
- **Ethical obligations**: Patients deserve meaningful control over their personal health information

JupyterHealth Exchange addresses these challenges through **scope-based consent**, where patients grant or deny permission for specific data types independently.

## The Consent Hierarchy

Consent in JHE operates through a three-level hierarchy that connects patients to studies through specific data permissions:

:::{div}
:class: dark:hidden

```{mermaid}
%%{init: {'theme':'default'}}%%
graph TB
    Patient[Patient] --> StudyPatient[Study Enrollment]
    Study[Study] --> StudyPatient
    StudyPatient --> Consent1["Consent: Blood Glucose ✓"]
    StudyPatient --> Consent2["Consent: Heart Rate ✓"]
    StudyPatient --> Consent3["Consent: Sleep Data ✗"]

    Study --> ScopeRequest1[Requests: Blood Glucose]
    Study --> ScopeRequest2[Requests: Heart Rate]
    Study --> ScopeRequest3[Requests: Sleep Data]

    style Consent1 fill:#d4edda
    style Consent2 fill:#d4edda
    style Consent3 fill:#f8d7da
```

:::

:::{div}
:class: hidden dark:block

```{mermaid}
%%{init: {'theme':'dark', 'themeVariables': {'lineColor':'#ffffff', 'primaryColor':'#1f2937', 'primaryTextColor':'#fff', 'primaryBorderColor':'#4b5563'}, 'flowchart': {'useMaxWidth': true}}}%%
graph TB
    Patient[Patient] --> StudyPatient[Study Enrollment]
    Study[Study] --> StudyPatient
    StudyPatient --> Consent1["Consent: Blood Glucose ✓"]
    StudyPatient --> Consent2["Consent: Heart Rate ✓"]
    StudyPatient --> Consent3["Consent: Sleep Data ✗"]

    Study --> ScopeRequest1[Requests: Blood Glucose]
    Study --> ScopeRequest2[Requests: Heart Rate]
    Study --> ScopeRequest3[Requests: Sleep Data]

    style Consent1 fill:#d4edda,stroke:#333,stroke-width:2px,color:#000
    style Consent2 fill:#d4edda,stroke:#333,stroke-width:2px,color:#000
    style Consent3 fill:#f8d7da,stroke:#333,stroke-width:2px,color:#000

    classDef arrowClass color:#fff
    linkStyle default stroke:#ffffff,stroke-width:2px
```

:::

### Level 1: Organization Membership

Before any consent can be granted, a patient must be associated with an organization. For example, Alice might belong to "Academic Medical Center".

This relationship establishes the administrative boundary for data sharing. Patients can belong to multiple organizations.

### Level 2: Study Enrollment

Within an organization, patients can be enrolled in specific research studies. For example, Alice might enroll in a "Diabetes Management Study" conducted by her medical center.

Study enrollment alone does **not** grant data access. It simply indicates the patient is participating in the study.

### Level 3: Scope-Specific Consent

For each enrolled study, patients grant consent for individual data types (scopes). For example, Alice might consent to share blood glucose readings with the diabetes study, but decline to share her sleep data. Each consent decision is recorded with a timestamp, creating an audit trail.

## Scope Codes: What Patients Consent To

Scopes represent specific types of health data. JHE uses **CodeableConcept** to define scopes using standardized coding systems:

```json
{
  "coding_system": "https://w3id.org/openmhealth",
  "coding_code": "omh:blood-glucose:3.0",
  "text": "Blood glucose"
}
```

### Common Scope Examples

| Scope Text | Coding System | Code | Description |
|------------|--------------|------|-------------|
| Blood glucose | Open mHealth | `omh:blood-glucose:3.0` | Glucose meter readings |
| Heart rate | Open mHealth | `omh:heart-rate:2.0` | HR from wearables/monitors |
| Blood pressure | Open mHealth | `omh:blood-pressure:4.0` | Systolic/diastolic readings |
| Physical activity | Open mHealth | `omh:physical-activity:2.1` | Exercise and movement data |
| Step count | Open mHealth | `omh:step-count:3.0` | Daily step totals |
| Body weight | Open mHealth | `omh:body-weight:2.0` | Scale measurements |
| Sleep duration | Open mHealth | `omh:sleep-duration:2.0` | Hours slept per night |

### Study Scope Requests

When researchers create a study, they define which scopes the study needs. For example, a diabetes study might request three data types: blood glucose measurements, physical activity data, and body weight observations.

Patients see these requests and decide which to approve.

## The Consent Workflow

Here's what happens when a patient enrolls in a study:

:::{div}
:class: dark:hidden

```{mermaid}
%%{init: {'theme':'default'}}%%
sequenceDiagram
    participant P as Patient
    participant CH as CommonHealth App
    participant JHE as JupyterHealth Exchange
    participant R as Researcher

    R->>JHE: 1. Create study with scope requests
    R->>JHE: 2. Enroll patient in study
    JHE->>P: 3. Send invitation link
    P->>CH: 4. Open invitation
    CH->>JHE: 5. Authenticate patient
    JHE->>CH: 6. Return pending consent requests
    Note over CH: Study: Diabetes Management<br/>Requests:<br/>✓ Blood glucose<br/>✓ Physical activity<br/>✓ Body weight
    P->>CH: 7. Review and grant/deny each scope
    CH->>JHE: 8. Upload consent decisions
    JHE->>JHE: 9. Store StudyPatientScopeConsent records
    JHE->>R: 10. Notify: Patient consented
```

:::

:::{div}
:class: hidden dark:block

```{mermaid}
%%{init: {'theme':'dark'}}%%
sequenceDiagram
    participant P as Patient
    participant CH as CommonHealth App
    participant JHE as JupyterHealth Exchange
    participant R as Researcher

    R->>JHE: 1. Create study with scope requests
    R->>JHE: 2. Enroll patient in study
    JHE->>P: 3. Send invitation link
    P->>CH: 4. Open invitation
    CH->>JHE: 5. Authenticate patient
    JHE->>CH: 6. Return pending consent requests
    Note over CH: Study: Diabetes Management<br/>Requests:<br/>✓ Blood glucose<br/>✓ Physical activity<br/>✓ Body weight
    P->>CH: 7. Review and grant/deny each scope
    CH->>JHE: 8. Upload consent decisions
    JHE->>JHE: 9. Store StudyPatientScopeConsent records
    JHE->>R: 10. Notify: Patient consented
```

:::

### Pending vs. Active Consent

JHE distinguishes between:

- **Pending consent**: Study scope requests that the patient hasn't yet responded to
- **Active consent**: Scope permissions that have been explicitly granted or denied

This distinction helps patient interfaces show:
- "You have 3 pending consent requests" (action required)
- "You are sharing 5 data types with this study" (informational)

The API endpoint `/api/Patient/{id}/consents` returns both:

```json
{
  "studies_pending_consent": [
    {
      "study": {"id": 42, "name": "Heart Health Study"},
      "pending_scope_consents": [
        {"code": {"text": "Heart rate"}, "consented": null}
      ]
    }
  ],
  "studies": [
    {
      "study": {"id": 15, "name": "Diabetes Management Study"},
      "scope_consents": [
        {"code": {"text": "Blood glucose"}, "consented": true, "consented_time": "2025-01-15T10:30:00Z"},
        {"code": {"text": "Sleep data"}, "consented": false, "consented_time": "2025-01-15T10:30:00Z"}
      ]
    }
  ]
}
```

## Consent Enforcement

Every time data enters or leaves JHE, consent is validated. This happens at multiple enforcement points:

### 1. Data Upload Validation

When the CommonHealth app uploads an observation, JHE checks:

The system checks whether the patient is enrolled in any study that requests that data type and has active consent. If the patient hasn't consented to share blood glucose data with any study, the observation is rejected.

### 2. Researcher Data Access

When a researcher queries patient data, JHE verifies:

The system verifies the practitioner is authorized for the patient's organization, then checks whether the patient has active consent to share the requested data type with a study in that organization. Only observations matching consented scopes are returned in query results.

### 3. Consolidated Patient Scopes

For efficient access control, JHE can query all scopes a patient has consented to across all studies:

The system can efficiently query all data types a patient has consented to share across all their study enrollments. This consolidated view is used when:
- Displaying patient dashboards ("You are sharing 7 data types")
- Validating bulk data uploads
- Filtering FHIR API responses

## Modifying Consent

Patients can change their consent decisions at any time through the CommonHealth app, and researchers (with appropriate permissions) can modify consent on behalf of patients through the JHE web UI or API.

### Patient-Initiated Changes via CommonHealth App

When patients want to grant or revoke consent, they use the CommonHealth Android app:

**User Experience**:
1. Patient opens the app and navigates to study consent requests
2. App displays a consent screen showing:
   - Study name and organization
   - List of requested data types (blood glucose, heart rate, etc.) with icons
   - Accept or Decline buttons
3. Patient taps their choice

**Behind the Scenes (API Call)**:
The app then makes a POST request to JHE:

```http
POST /api/v1/patients/{patient_id}/consents
Content-Type: application/json
Authorization: Bearer {patient_access_token}

{
  "study_scope_consents": [{
    "study_id": 15,
    "scope_consents": [{
      "coding_system": "https://w3id.org/openmhealth",
      "coding_code": "omh:blood-glucose:3.0",
      "consented": true  // or false if declining
    }]
  }]
}
```

The `consented_time` timestamp is automatically set by JHE to record when the decision was made.

```{note}
The CommonHealth app currently supports consent decisions during initial enrollment. Modifying consent after enrollment (e.g., revoking previously granted consent) would use the same API endpoint with `PATCH` method and updated `consented` values.
```

### Practitioner-Initiated Changes

Practitioners with `member` or `manager` roles in the study's organization can update consent during enrollment:

The system checks whether the practitioner has a member or manager role in the study's organization before allowing consent modification.

This supports in-clinic enrollment workflows where research coordinators help patients through the consent process.

## Consent Revocation

Revoking consent has immediate effect:

1. **Setting `consented=False`**: Patient explicitly declines to share a data type
2. **Deleting consent records**: Using `DELETE /api/Patient/{id}/consents` removes consent entirely

After revocation:
- **No new observations** of that type will be accepted for that patient/study combination
- **Existing observations** remain in the database (data retention for research integrity)
- **Future queries** will exclude the revoked data type from results

```{note}
Consent revocation does not retroactively delete previously collected data. This aligns with research ethics guidelines that distinguish between "stop collecting new data" and "delete all historical data."
```

For full data deletion (GDPR "right to be forgotten"), the patient must be removed from the study entirely, which triggers cascading deletion of consent records and observations.

## Consent Audit Trail

Every consent change is timestamped, creating an immutable audit trail:

Each consent record includes the study enrollment, data type, consent status (granted/denied), and an immutable timestamp. This supports:
- **Compliance audits**: Proving data was collected with valid consent
- **Temporal queries**: "What was the patient's consent status on 2024-03-15?"
- **Research integrity**: Documenting when each data point was authorized for collection

## Multi-Study Consent

A patient can participate in multiple studies simultaneously, each with different consent preferences:

```
Patient: Alice
├── Study: Diabetes Management
│   ├── Blood glucose: ✓ Consented
│   ├── Physical activity: ✓ Consented
│   └── Sleep data: ✗ Declined
└── Study: Cardiac Monitoring
    ├── Heart rate: ✓ Consented
    ├── Blood pressure: ✓ Consented
    └── Sleep data: ✓ Consented  (different consent from first study)
```

Even though Alice declined to share sleep data with the Diabetes study, she can consent to share it with the Cardiac study. Each study maintains independent consent.

### Data Sharing Implications

When Alice uploads a blood glucose reading:
- **Diabetes Management Study**: ✓ Receives the observation (consented)
- **Cardiac Monitoring Study**: ✗ Does not receive it (no blood glucose consent)

When Alice uploads a heart rate measurement:
- **Diabetes Management Study**: ✗ Does not receive it (no heart rate consent)
- **Cardiac Monitoring Study**: ✓ Receives the observation (consented)

When Alice uploads sleep data:
- **Diabetes Management Study**: ✗ Does not receive it (declined)
- **Cardiac Monitoring Study**: ✓ Receives the observation (consented)

## Scope Actions: Read and Search Permissions

Each consent record includes a `scope_actions` field following [SMART on FHIR scope syntax](https://build.fhir.org/ig/HL7/smart-app-launch/scopes-and-launch-context.html#scopes-for-requesting-fhir-resources):

Each consent record can include scope actions (following SMART on FHIR syntax) specifying which operations are allowed. For example, "rs" means read and search operations are permitted.

**Scope actions**:
- `r`: **Read** - Retrieve individual observation by ID
- `s`: **Search** - Query multiple observations
- `rs`: **Read + Search** (default) - Both operations allowed

This aligns with FHIR access control patterns.

## Design Rationale

### Why CodeableConcept for Scopes?

Using standardized codes (Open mHealth, LOINC, SNOMED CT) instead of free text provides:

- **Interoperability**: Scopes match the data schemas they govern
- **Consistency**: "Blood glucose" always uses `omh:blood-glucose:3.0`
- **Machine readability**: Applications can programmatically understand scope meanings
- **Version control**: Schema versions (`:3.0`) track data format changes

### Why Study-Specific Consent?

Rather than global "Alice shares blood glucose with everyone," consent is scoped to studies because:

- **Purpose limitation**: GDPR requires data be used for specified, explicit purposes
- **Trust boundaries**: Patients trust different research teams differently
- **Regulatory compliance**: IRB protocols require study-specific consent
- **Flexibility**: Patients can participate in multiple studies with different sharing preferences

### Why Allow Consent=False?

Recording declined consent (rather than just omitting it) provides:

- **Audit evidence**: "Patient was asked and declined" vs. "we don't know"
- **UI clarity**: Show patients what they've declined, not just what they've approved
- **Compliance**: Document informed choice, not just passive non-consent

## Common Use Cases

### Use Case 1: In-Clinic Enrollment

A research coordinator enrolls a patient during an office visit:

1. Coordinator creates patient account in JHE
2. Coordinator enrolls patient in study
3. Coordinator reviews scope requests with patient
4. Coordinator updates consent on patient's behalf (requires `member` or `manager` role)
5. Patient receives invitation link for future self-service consent management

### Use Case 2: Remote Enrollment

A patient receives an email invitation:

1. Patient clicks invitation link
2. Link authenticates patient to CommonHealth app
3. App fetches pending consent requests from JHE
4. Patient reviews study details and scope requests
5. Patient grants/denies each scope individually
6. App uploads consent decisions to JHE
7. Patient connects data sources (Fitbit, Apple Health, etc.)

### Use Case 3: Consent Revocation

Patient initially consented but wants to stop sharing data:

1. Patient contacts research coordinator or uses JHE web interface (if available)
2. Consent is revoked by setting `consented=false` via `PATCH /api/Patient/{id}/consents`
3. Future data uploads for that scope are rejected for this study
4. Other studies' access to that data type (if any) remains unchanged
5. Previously collected data remains in the database for research integrity

### Use Case 4: Data Lifecycle Management

Managing patient data throughout a study lifecycle:

**What Works Today**:
1. **Consent always enforced**: Data uploads and queries check consent regardless of study age
2. **Patient can revoke consent**: Using `PATCH /api/Patient/{id}/consents` to set `consented=false` (existing data remains for research integrity)
3. **Patient removal from organization**: When a patient is removed from ALL organizations, observations are deleted (supports GDPR "right to be forgotten")

**What's Not Yet Implemented**:
- **Study status tracking**: No "active/completed/archived" states for studies
- **Automated retention policies**: No scheduled archival or deletion based on organization policies
- **Study completion workflows**: No formal process to mark a study as concluded

```{note}
Currently, studies remain "active" indefinitely. Data lifecycle management requires manual intervention by researchers with appropriate permissions to remove patients from studies or delete entire studies via the Django admin interface.
```

## Security Considerations

### Access Control Layers

Consent enforcement works alongside other security mechanisms:

1. **Authentication**: Prove identity (OAuth 2.0)
2. **Organization membership**: Prove affiliation (PatientOrganization)
3. **Study enrollment**: Prove participation (StudyPatient)
4. **Scope consent**: Prove data sharing permission (StudyPatientScopeConsent)
5. **Role-based access**: Prove practitioner authorization (PractitionerOrganization)

All layers must pass for data access.


## Related Documentation

- [Data Flow](./data-flow.md) - See how consent fits into the data pipeline
- [RBAC and Governance](./rbac.md) - Understand practitioner access control
- [Access Control Design](./access-control-design.md) - Deep dive into authorization architecture
- [FHIR Interoperability](./fhir-interoperability.md) - How consent integrates with FHIR APIs
