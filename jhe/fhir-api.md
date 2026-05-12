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
          "data": "eyJib2R5IjogeyJlZmZlY3RpdmVfdGltZV9mcmFtZSI6IHsiZGF0ZV90aW1lIjogIjIwMjQtMDUt\nMDJUMDc6MjE6MDAtMDc6MDAifSwgInN5c3RvbGljX2Jsb29kX3ByZXNzdXJlIjogeyJ1bml0Ijog\nIm1tSGciLCAidmFsdWUiOiAxMjJ9LCAiZGlhc3RvbGljX2Jsb29kX3ByZXNzdXJlIjogeyJ1bml0\nIjogIm1tSGciLCAidmFsdWUiOiA3N319LCAiaGVhZGVyIjogeyJ1dWlkIjogIjZlM2RiODg3LTRh\nMjAtMzIyMi05OTk4LTI5NzJhZjZmYjA5MSIsICJtb2RhbGl0eSI6ICJzZW5zZWQiLCAic2NoZW1h\nX2lkIjogeyJuYW1lIjogImJsb29kLXByZXNzdXJlIiwgInZlcnNpb24iOiAiMy4xIiwgIm5hbWVz\ncGFjZSI6ICJvbWgifSwgImNyZWF0aW9uX2RhdGVfdGltZSI6ICIyMDI0LTEwLTI1VDIxOjEzOjMx\nLjQzOFoiLCAiZXh0ZXJuYWxfZGF0YXNoZWV0cyI6IFt7ImRhdGFzaGVldF90eXBlIjogIm1hbnVm\nYWN0dXJlciIsICJkYXRhc2hlZXRfcmVmZXJlbmNlIjogImh0dHBzOi8vaWhlYWx0aGxhYnMuY29t\nL3Byb2R1Y3RzIn1dLCAic291cmNlX2RhdGFfcG9pbnRfaWQiOiAiZTZjMTliMDQyOGM4NWJiYjdj\nMTk4MGNiOTRkZDE3N2YiLCAic291cmNlX2NyZWF0aW9uX2RhdGVfdGltZSI6ICIyMDI0LTA1LTAy\nVDA3OjIxOjAwLTA3OjAwIn19",
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
          "data": "eyJzeXN0b2xpY19ibG9vZF9wcmVzc3VyZSI6eyJ2YWx1ZSI6MTQyLCJ1bml0IjoibW1IZyJ9LCJkaWFzdG9saWNfYmxvb2RfcHJlc3N1cmUiOnsidmFsdWUiOjg5LCJ1bml0IjoibW1IZyJ9LCJlZmZlY3RpdmVfdGltZV9mcmFtZSI6eyJkYXRlX3RpbWUiOiIyMDIxLTAzLTE0VDA5OjI1OjAwLTA3OjAwIn19"
        }
      },
      "request": {
        "method": "POST",
        "url": "Observation"
      }
    },
    ...
```
