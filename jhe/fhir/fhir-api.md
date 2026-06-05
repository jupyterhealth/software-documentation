---
title: FHIR API
---

### Patients

The `FHIR Patient` endpoint returns a list of Patients as a FHIR Bundle for a given Study ID passed as query parameter`_has:Group:member:_id` or alternatively a single Patient matching the query parameter `identifier=<system>|<value>`

| Query Parameter         | Example                  | Description                                            |
| ----------------------- | ------------------------ | ------------------------------------------------------ |
| `_has:Group:member:_id` | `30001`                  | Filter by Patients that are in the Study with ID 30001 |
| `identifier`            | \`http://ehr.example.com | abc123\`                                               |

```json
// GET /fhir/r5/Patient?_has:Group:member:_id=30001
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
            "given": [
              "Peter"
            ],
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

### Observations

- The `FHIR Observation` endpoint returns a list of Observations as a FHIR Bundle
- At least one of Study ID, passed as `patient._has:Group:member:_id` or Patient ID, passed as `patient` or Patient Identifier passed as `patient.identifier=<system>|<value>` query parameters are required
- `subject.reference` references a Patient ID
- `device.reference` references a Data Source ID
- `valueAttachment` is Base 64 Encoded Binary JSON

| Query Parameter                 | Example                                                | Description                                                                                       |
| ------------------------------- | ------------------------------------------------------ | ------------------------------------------------------------------------------------------------- |
| `patient._has:Group:member:_id` | `30001`                                                | Filter by Patients that are in the Study with ID 30001                                            |
| `patient`                       | `40001`                                                | Filter by single Patient with ID 40001                                                            |
| `patient.identifier`            | `http://ehr.example.com\|abc123`                       | Filter by single Patient with Identifier System `http://ehr.example.com` and Value `abc123`       |
| `code`                          | `https://w3id.org/openmhealth\|omh:blood-pressure:4.0` | Filter by Type/Scope with System `https://w3id.org/openmhealth` and Code `omh:blood-pressure:4.0` |

```json
// GET /fhir/r5/Observation?patient._has:Group:member:_id=30001&patient=40001&code=https://w3id.org/openmhealth|omh:blood-pressure:4.0
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

Observations are uploaded as FHIR Batch bundles sent as a POST to the root endpoint

```json
// POST /fhir/r5/
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
