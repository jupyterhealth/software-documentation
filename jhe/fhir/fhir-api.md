---
title: FHIR API
---

The JHE FHIR server is served at `FHIR/R5/` (the lowercase alias `fhir/r5/` is also supported for backward compatibility).

## Working with OMH Observations

OMH Observations are the primary data type in JHE and are stored natively in the system. They are read and written as FHIR R5 resources.

### Querying Observations

At least one patient-scoping parameter is required. When filtering by study (`patient._has:Group:member:_id`), results are additionally restricted to the study's consented observation codes.

| Query Parameter | Example | Description |
| --- | --- | --- |
| `patient._has:Group:member:_id` | `30001` | Observations for patients enrolled in Study 30001 |
| `patient` | `40001` | Observations for Patient 40001 |
| `patient.organization` | `20001` | Observations for patients in Organization 20001 |
| `patient.identifier` | `http://ehr.example.com\|abc123` | Observations for the patient with that identifier |
| `code` | `https://w3id.org/openmhealth\|omh:blood-pressure:4.0` | Filter by observation type |

Notes:
- `subject.reference` references a Patient ID
- `device.reference` references a Data Source (Device) ID
- `valueAttachment.data` is Base64-encoded JSON

```json
// GET /FHIR/R5/Observation?patient._has:Group:member:_id=30001&code=https://w3id.org/openmhealth|omh:blood-pressure:4.0
{
  "resourceType": "Bundle",
  "type": "searchset",
  "entry": [
    {
      "resource": {
        "resourceType": "Observation",
        "id": "63416",
        "meta": {
          "lastUpdated": "2024-10-25T21:14:02.871132+00:00"
        },
        "identifier": [
          {
            "value": "6e3db887-4a20-3222-9998-2972af6fb091",
            "system": "https://ehr.example.com"
          }
        ],
        "status": "final",
        "subject": {
          "reference": "Patient/40001"
        },
        "device": {
          "reference": "Device/70001"
        },
        "code": {
          "coding": [
            {
              "code": "omh:blood-pressure:4.0",
              "system": "https://w3id.org/openmhealth"
            }
          ]
        },
        "valueAttachment": {
          "data": "eyJoZWFkZXIiOnsidXVpZCI6IjZlM2RiODg3LTRhMjAtMzIyMi05OTk4LTI5NzJhZjZmYjA5MSIsInNjaGVtYV9pZCI6eyJuYW1lc3BhY2UiOiJvbWgiLCJuYW1lIjoiYmxvb2QtcHJlc3N1cmUiLCJ2ZXJzaW9uIjoiNC4wIn0sInNvdXJjZV9jcmVhdGlvbl9kYXRlX3RpbWUiOiIyMDIxLTAzLTE0VDA5OjI1OjAwLTA3OjAwIn0sImJvZHkiOnsic3lzdG9saWNfYmxvb2RfcHJlc3N1cmUiOnsidmFsdWUiOjE0MiwidW5pdCI6Im1tSGcifSwiZGlhc3RvbGljX2Jsb29kX3ByZXNzdXJlIjp7InZhbHVlIjo4OSwidW5pdCI6Im1tSGcifSwiZWZmZWN0aXZlX3RpbWVfZnJhbWUiOnsiZGF0ZV90aW1lIjoiMjAyMS0wMy0xNFQwOToyNTowMC0wNzowMCJ9fX0=",
          "contentType": "application/json"
        }
      }
    },
    ...
```

### Uploading Observations

Observations are uploaded as FHIR Batch Bundles via `POST` to the base endpoint.

```json
// POST /FHIR/R5/
{
  "resourceType": "Bundle",
  "type": "batch",
  "entry": [
    {
      "resource": {
        "resourceType": "Observation",
        "status": "final",
        "code": {
          "coding": [
            {
              "system": "https://w3id.org/openmhealth",
              "code": "omh:blood-pressure:4.0"
            }
          ]
        },
        "subject": {
          "reference": "Patient/40001"
        },
        "device": {
          "reference": "Device/70001"
        },
        "identifier": [
          {
            "value": "6e3db887-4a20-3222-9998-2972af6fb091",
            "system": "https://ehr.example.com"
          }
        ],
        "valueAttachment": {
          "contentType": "application/json",
          "data": "eyJoZWFkZXIiOnsidXVpZCI6IjZlM2RiODg3LTRhMjAtMzIyMi05OTk4LTI5NzJhZjZmYjA5MSIsInNjaGVtYV9pZCI6eyJuYW1lc3BhY2UiOiJvbWgiLCJuYW1lIjoiYmxvb2QtcHJlc3N1cmUiLCJ2ZXJzaW9uIjoiNC4wIn0sInNvdXJjZV9jcmVhdGlvbl9kYXRlX3RpbWUiOiIyMDIxLTAzLTE0VDA5OjI1OjAwLTA3OjAwIn0sImJvZHkiOnsic3lzdG9saWNfYmxvb2RfcHJlc3N1cmUiOnsidmFsdWUiOjE0MiwidW5pdCI6Im1tSGcifSwiZGlhc3RvbGljX2Jsb29kX3ByZXNzdXJlIjp7InZhbHVlIjo4OSwidW5pdCI6Im1tSGcifSwiZWZmZWN0aXZlX3RpbWVfZnJhbWUiOnsiZGF0ZV90aW1lIjoiMjAyMS0wMy0xNFQwOToyNTowMC0wNzowMCJ9fX0="
        }
      },
      "request": {
        "method": "POST",
        "url": "Observation"
      }
    },
    ...
```

## Working with Patients, Practitioners, Organizations, Groups (Studies) and Devices (Data Sources)

These resources are **read-only** projections of JHE system entities — writes to them via FHIR are not supported (use the `/api/v1/` REST API to manage these). All support the same patient-scoping query parameters.

| Query Parameter | Example | Description |
| --- | --- | --- |
| `_has:Group:member:_id` | `30001` | Resources belonging to patients in Study 30001 |
| `patient` | `40001` | Resources belonging to Patient 40001 |
| `patient.organization` | `20001` | Resources belonging to patients in Organization 20001 |
| `patient._has:Group:member:_id` | `30001` | Resources belonging to patients enrolled in Study 30001 |
| `patient.identifier` | `http://ehr.example.com\|abc123` | Resources belonging to the patient with that identifier |
| `identifier` | `http://ehr.example.com\|abc123` | Match the resource itself by identifier (Patient only) |

```json
// GET /FHIR/R5/Patient?_has:Group:member:_id=30001
{
  "resourceType": "Bundle",
  "type": "searchset",
  "entry": [
    {
      "resource": {
        "resourceType": "Patient",
        "id": "40001",
        "meta": {
          "lastUpdated": "2024-10-23T12:35:25.142027+00:00"
        },
        "identifier": [
          {
            "value": "fhir-1234",
            "system": "http://ehr.example.com"
          }
        ],
        "name": [
          {
            "given": ["Peter"],
            "family": "ThePatient"
          }
        ],
        "birthDate": "1980-01-01",
        "telecom": [
          {
            "value": "peter@example.com",
            "system": "email"
          },
          {
            "value": "347-111-1111",
            "system": "phone"
          }
        ]
      }
    },
    ...
```

The same scoping parameters apply to `Group` (Study), `Device` (Data Source), `Organization`, and `Practitioner`:

| Resource | Query | Returns |
| --- | --- | --- |
| `Group` (Study) | `GET /FHIR/R5/Group?patient._has:Group:member:_id=30001` | The study with ID 30001 |
| `Group` (Study) | `GET /FHIR/R5/Group?patient=40001` | Studies Patient 40001 is enrolled in |
| `Group` (Study) | `GET /FHIR/R5/Group?patient.organization=20001` | Studies in Organization 20001 |
| `Device` (Data Source) | `GET /FHIR/R5/Device?patient._has:Group:member:_id=30001` | Data sources used in Study 30001 |
| `Device` (Data Source) | `GET /FHIR/R5/Device?patient=40001` | Data sources used in Patient 40001's studies |
| `Device` (Data Source) | `GET /FHIR/R5/Device?patient.organization=20001` | Data sources in studies under Organization 20001 |
| `Organization` | `GET /FHIR/R5/Organization?patient._has:Group:member:_id=30001` | The organization backing Study 30001 |
| `Organization` | `GET /FHIR/R5/Organization?patient=40001` | Organizations Patient 40001 belongs to |
| `Organization` | `GET /FHIR/R5/Organization?patient.organization=20001` | Organization 20001 |
| `Practitioner` | `GET /FHIR/R5/Practitioner?patient._has:Group:member:_id=30001` | Practitioners in Study 30001's organization |
| `Practitioner` | `GET /FHIR/R5/Practitioner?patient=40001` | Practitioners in Patient 40001's organizations |
| `Practitioner` | `GET /FHIR/R5/Practitioner?patient.organization=20001` | Practitioners in Organization 20001 |

## Working with other resources

Any FHIR resource type not natively modeled in JHE (such as `QuestionnaireResponse`, `Condition`, or `MedicationRequest`) can be stored and retrieved via the auxiliary resource store. Before uploading, a patient must register a **FHIR Source** that identifies the upstream system the data originates from.

### Step 1: Register a FHIR Source

```
POST /api/v1/fhir_sources
Authorization: Bearer WMSk9Te6QpENVxudxkRjeNBFlCf5pq
Content-Type: application/json
Accept: application/json
```

```json
{
  "data_source": 70005,
  "label": "Neptune MyChart",
  "fhir_base_url": "https://fhir.neptune.org/api/FHIR/R4"
}
```

Response:

```json
{
  "id": 90001,
  "patient": 40001,
  "dataSource": 70005,
  "label": "Neptune MyChart",
  "fhirBaseUrl": "https://fhir.neptune.org/api/FHIR/R4",
  "lastUpdated": "2026-06-07T04:11:52.577605Z"
}
```

The `id` returned (`90001`) is the FHIR Source ID used in subsequent requests.

### Step 2: Upload the resource

Include the `X-JHE-FHIR-Source-ID` header set to the FHIR Source ID from step 1. Writes require this header — omitting it returns a `400`.

```
POST /FHIR/R5/QuestionnaireResponse
Authorization: Bearer WMSk9Te6QpENVxudxkRjeNBFlCf5pq
Content-Type: application/json
Accept: application/json
X-JHE-FHIR-Source-ID: 90001
```

```json
{
  "resourceType": "QuestionnaireResponse",
  "status": "completed",
  "questionnaire": "Questionnaire/weekly-symptom-severity-vas",
  "subject": {
    "reference": "Patient/123"
  },
  "authored": "2026-05-28T14:30:00Z",
  "item": [
    {
      "linkId": "cough-severity",
      "answer": [{"valueInteger": 37}]
    },
    {
      "linkId": "dyspnea-severity",
      "answer": [{"valueInteger": 62}]
    },
    {
      "linkId": "fatigue-severity",
      "answer": [{"valueInteger": 48}]
    }
  ]
}
```

Response:

```json
{
  "resourceType": "QuestionnaireResponse",
  "id": "2d50b34d-b33c-4f47-a5e8-595764d51f53",
  "status": "completed",
  "questionnaire": "Questionnaire/weekly-symptom-severity-vas",
  "subject": {
    "reference": "Patient/123"
  },
  "authored": "2026-05-28T14:30:00Z",
  "item": [
    {
      "linkId": "cough-severity",
      "answer": [{"valueInteger": 37}]
    },
    {
      "linkId": "dyspnea-severity",
      "answer": [{"valueInteger": 62}]
    },
    {
      "linkId": "fatigue-severity",
      "answer": [{"valueInteger": 48}]
    }
  ]
}
```

The `id` returned is a UUID assigned by JHE.

### Step 3: Retrieve the resource

Resources can be retrieved by FHIR Source (scoped to that source's patient) or by any of the standard patient-scoping query parameters.

**By FHIR Source** — the `X-JHE-FHIR-Source-ID` header takes precedence over query parameters:

```
GET /FHIR/R5/QuestionnaireResponse
Authorization: Bearer WMSk9Te6QpENVxudxkRjeNBFlCf5pq
X-JHE-FHIR-Source-ID: 90001
```

**By patient or study** — standard query parameters without the header:

```
GET /FHIR/R5/QuestionnaireResponse?patient=40001
GET /FHIR/R5/QuestionnaireResponse?patient._has:Group:member:_id=30001
```

**By resource ID**:

```
GET /FHIR/R5/QuestionnaireResponse/2d50b34d-b33c-4f47-a5e8-595764d51f53
```

All reads return a FHIR `searchset` Bundle (or a single resource for an ID lookup).
