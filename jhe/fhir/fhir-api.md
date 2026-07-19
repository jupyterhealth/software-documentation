---
title: FHIR API
---

The JHE FHIR server is served at `FHIR/R5/` (the lowercase alias `fhir/r5/` is also supported for backward compatibility).

## Querying: JHE-native vs imported

Every search hits **one** store, selected by `_source`:

- no `_source` → **JHE-native** records (the default)
- `_source=https://jupyterhealth.org/fhir/fhir-source/90001` → one imported **FHIR Source**
- `_source:below=https://jupyterhealth.org/fhir/fhir-source/` → **all** imported resources

```
GET /FHIR/R5/Observation?patient=40001                                            # JHE-native (default)
GET /FHIR/R5/Condition?_source=https://jupyterhealth.org/fhir/fhir-source/90001    # one source
GET /FHIR/R5/Condition?_source:below=https://jupyterhealth.org/fhir/fhir-source/   # all imported
```

- JHE-native records carry `meta.source: "https://jupyterhealth.org/jhe"`.
- Imported records carry `meta.source: "https://jupyterhealth.org/fhir/fhir-source/<id>"`.
- A single query never spans both stores.

(filtering-sorting-counting)=

## Filtering, sorting & counting

Search parameters (US Core) — the set available depends on the resource type:

| Type                 | Examples                                                                                     |
| -------------------- | -------------------------------------------------------------------------------------------- |
| token                | `?clinical-status=active` · `?code=http://loinc.org\|1234-5` · `?category=problem-list-item` |
| date                 | `?recorded-date=ge2024-01-01` · `?date=le2024-12-31` (`ge`/`le`/`gt`/`lt` or a bare date)    |
| string (starts-with) | `?name=nor` matches "North Clinic" (case-insensitive)                                        |
| reference            | `?encounter=Encounter/abc` · `?patient=40001`                                                |

- **comma = OR:** `?category=problem-list-item,encounter-diagnosis`
- **repeat = AND / range:** `?recorded-date=ge2024-01-01&recorded-date=le2024-12-31`
- always available on any resource: `?_id=…` · `?_lastUpdated=ge2024-01-01`

Sort with `_sort` (prefix `-` for descending):

```
GET /FHIR/R5/Observation?patient=40001&_sort=date          # by effective[x], oldest first
GET /FHIR/R5/Condition?patient=40001&_sort=-_lastUpdated    # most recently updated first
```

- `_sort=date` → the resource's date (Observation `effective[x]`, Condition `recordedDate`, MedicationRequest `authoredOn`, …)
- `_sort=_lastUpdated` → record update time

Count only with `_summary=count`:

```json
// GET /FHIR/R5/Condition?patient=40001&_summary=count
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 12,
  "entry": []
}
```

## Working with OMH Observations

OMH Observations are the primary data type in JHE and are stored natively in the system. They are read and written as FHIR R5 resources.

### Querying Observations

Patient-scoping parameters are optional — a paramless search returns everything you can access (a patient user's own observations; a practitioner's observations across all their organizations). When filtering by study (`patient._has:Group:member:_id`), results are additionally restricted to the study's consented observation codes.

| Query Parameter                 | Example                                                | Description                                         |
| ------------------------------- | ------------------------------------------------------ | --------------------------------------------------- |
| `patient._has:Group:member:_id` | `30001`                                                | Observations for patients enrolled in Study 30001   |
| `patient`                       | `40001`                                                | Observations for Patient 40001                      |
| `patient.organization`          | `20001`                                                | Observations for patients in Organization 20001     |
| `patient.identifier`            | `http://ehr.example.com\|abc123`                       | Observations for the patient with that identifier   |
| `code`                          | `https://w3id.org/openmhealth\|omh:blood-pressure:4.0` | Filter by observation type                          |
| `date`                          | `ge2024-01-01`                                         | Filter by `effective[x]` time (`ge`/`le`/`gt`/`lt`) |

Notes:

- `subject.reference` references a Patient ID
- `device.reference` references a Data Source (Device) ID
- `valueAttachment.data` is Base64-encoded JSON

Examples:

```
# all blood-pressure observations for a patient
GET /FHIR/R5/Observation?patient=40001&code=https://w3id.org/openmhealth|omh:blood-pressure:4.0

# a study's observations, most recent effective time first
GET /FHIR/R5/Observation?patient._has:Group:member:_id=30001&_sort=-date

# newer than a given effective time (filters on effective[x] — see Effective Time below)
GET /FHIR/R5/Observation?patient=40001&date=gt2026-01-01

# an effective-time window (repeat = AND)
GET /FHIR/R5/Observation?patient=40001&date=ge2026-01-01&date=le2026-06-30

# two observation types at once (comma = OR)
GET /FHIR/R5/Observation?patient=40001&code=omh:blood-pressure:4.0,omh:heart-rate:2.0

# just the count
GET /FHIR/R5/Observation?patient=40001&_summary=count
```

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

### Effective Time

On upload, JHE lifts the OMH `body.effective_time_frame` out of the Base64 payload and onto indexed columns so observations can be queried by time. These columns are surfaced on read as FHIR R5 `Observation.effective[x]`. The `valueAttachment.data` blob remains the source of truth; the elevated values are a derived projection, so a missing or unrecognised time frame simply leaves the row without an `effective[x]` value.

A single point in time becomes `effectiveDateTime`; every OMH `time_interval` form becomes an `effectivePeriod` (start + end):

| OMH `effective_time_frame` shape         | Example                                                                         | FHIR R5 output                                             |
| ---------------------------------------- | ------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| `date_time`                              | `{ "date_time": "2026-07-18T09:30:00+09:00" }`                                  | `effectiveDateTime: "2026-07-18T09:30:00+09:00"`           |
| `time_interval` — `start` + `end`        | `{ "start_date_time": "…T09:00…", "end_date_time": "…T10:00…" }`                | `effectivePeriod: { start, end }`                          |
| `time_interval` — `start` + `duration`   | `{ "start_date_time": "…T09:00…", "duration": { "value": 60, "unit": "min" } }` | `effectivePeriod: { start, end }` (end = start + duration) |
| `time_interval` — `end` + `duration`     | `{ "end_date_time": "…T10:00…", "duration": { "value": 60, "unit": "min" } }`   | `effectivePeriod: { start, end }` (start = end − duration) |
| `time_interval` — `date` + `part_of_day` | `{ "date": "2026-07-18", "part_of_day": "morning" }`                            | `effectivePeriod: { start, end }` (the part-of-day window) |

Notes:

- **Timezone:** a datetime with no offset is interpreted as UTC, matching the legacy OMH convention.
- **Duration units** are drawn from the OMH duration value set (`ps, ns, us, ms, sec, min, h, d, wk, Mo, yr`); calendar units (`Mo`, `yr`) are added relative to the anchor date (e.g. Jan 31 + 1 month → Feb 28).
- **`part_of_day`** maps to a fixed window on the given day: `morning` 06:00–12:00, `afternoon` 12:00–18:00, `evening` 18:00–24:00, `night` 00:00–06:00.
- After a round-trip through storage, datetimes are returned in UTC (e.g. `+00:00`); this is the same instant, and the original offset is retained in `valueAttachment.data`.

## Working with Patients, Practitioners, Organizations, Groups (Studies) and Devices (Data Sources)

These resources are **read-only** projections of JHE system entities — writes to them via FHIR are not supported (use the `/api/v1/` REST API to manage these). All support the same patient-scoping query parameters.

| Query Parameter                 | Example                          | Description                                               |
| ------------------------------- | -------------------------------- | --------------------------------------------------------- |
| `_has:Group:member:_id`         | `30001`                          | Resources belonging to patients in Study 30001            |
| `patient`                       | `40001`                          | Resources belonging to Patient 40001                      |
| `patient.organization`          | `20001`                          | Resources belonging to patients in Organization 20001     |
| `patient._has:Group:member:_id` | `30001`                          | Resources belonging to patients enrolled in Study 30001   |
| `patient.identifier`            | `http://ehr.example.com\|abc123` | Resources belonging to the patient with that identifier   |
| `identifier`                    | `http://ehr.example.com\|abc123` | Match the resource itself by identifier (Patient only)    |
| `family` / `given` / `name`     | `Smith`                          | Patient name starts-with, case-insensitive (Patient only) |
| `birthdate`                     | `ge1980-01-01`                   | Patient date of birth (Patient only)                      |

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

| Resource               | Query                                                           | Returns                                          |
| ---------------------- | --------------------------------------------------------------- | ------------------------------------------------ |
| `Group` (Study)        | `GET /FHIR/R5/Group?patient._has:Group:member:_id=30001`        | The study with ID 30001                          |
| `Group` (Study)        | `GET /FHIR/R5/Group?patient=40001`                              | Studies Patient 40001 is enrolled in             |
| `Group` (Study)        | `GET /FHIR/R5/Group?patient.organization=20001`                 | Studies in Organization 20001                    |
| `Device` (Data Source) | `GET /FHIR/R5/Device?patient._has:Group:member:_id=30001`       | Data sources used in Study 30001                 |
| `Device` (Data Source) | `GET /FHIR/R5/Device?patient=40001`                             | Data sources used in Patient 40001's studies     |
| `Device` (Data Source) | `GET /FHIR/R5/Device?patient.organization=20001`                | Data sources in studies under Organization 20001 |
| `Organization`         | `GET /FHIR/R5/Organization?patient._has:Group:member:_id=30001` | The organization backing Study 30001             |
| `Organization`         | `GET /FHIR/R5/Organization?patient=40001`                       | Organizations Patient 40001 belongs to           |
| `Organization`         | `GET /FHIR/R5/Organization?patient.organization=20001`          | Organization 20001                               |
| `Practitioner`         | `GET /FHIR/R5/Practitioner?patient._has:Group:member:_id=30001` | Practitioners in Study 30001's organization      |
| `Practitioner`         | `GET /FHIR/R5/Practitioner?patient=40001`                       | Practitioners in Patient 40001's organizations   |
| `Practitioner`         | `GET /FHIR/R5/Practitioner?patient.organization=20001`          | Practitioners in Organization 20001              |

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

Name the FHIR Source one of two ways (omit both → `400`):

- **Preferred** — set `meta.source` on the resource body:
  ```json
  "meta": { "source": "https://jupyterhealth.org/fhir/fhir-source/90001" }
  ```
- **Or** — send the `X-JHE-FHIR-Source-ID` header (a convenience fallback; wins if both are present):
  ```
  X-JHE-FHIR-Source-ID: 90001
  ```

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

Retrieve by FHIR Source, by patient/study, or by resource ID. (The `X-JHE-FHIR-Source-ID` header is write-only — it is ignored on reads; select a source with `_source` instead.)

**From one FHIR Source**:

```
GET /FHIR/R5/QuestionnaireResponse?_source=https://jupyterhealth.org/fhir/fhir-source/90001
```

**From all imported sources**:

```
GET /FHIR/R5/QuestionnaireResponse?_source:below=https://jupyterhealth.org/fhir/fhir-source/
```

**By patient or study** (add any filter/sort from [Filtering, sorting & counting](#filtering-sorting-counting)):

```
GET /FHIR/R5/QuestionnaireResponse?patient=40001
GET /FHIR/R5/QuestionnaireResponse?patient._has:Group:member:_id=30001&status=completed&_sort=-date
```

**By resource ID**:

```
GET /FHIR/R5/QuestionnaireResponse/2d50b34d-b33c-4f47-a5e8-595764d51f53
```

All reads return a FHIR `searchset` Bundle (or a single resource for an ID lookup).
