# Fetching and Exporting Data

This guide shows you how to programmatically fetch, export, and analyze patient health data from JupyterHealth Exchange.

### Introduction

- [Prerequisites](#prerequisites)
- [Understanding Data Access](#understanding-data-access)

### Viewing Data (Quick Reference)

- [Viewing Data (Quick Reference)](#viewing-data-quick-reference)

### Fetching Data Programmatically

- [Fetch via Admin REST API](#fetch-via-admin-rest-api)
- [Fetch via FHIR API](#fetch-via-fhir-api)
- [Pagination Strategies](#pagination-strategies)

### Exporting Data for Analysis

- [Export to CSV](#export-to-csv)
- [Export to Pandas DataFrame](#export-to-pandas-dataframe)
- [Export All Patients in Study](#export-all-patients-in-study)

### Advanced Topics

- [Real-Time Data Monitoring](#real-time-data-monitoring)
- [Data Quality Checks](#data-quality-checks)
- [Common Issues](#common-issues)
- [Best Practices](#best-practices)

### Reference

- [Related Documentation](#related-documentation)

______________________________________________________________________

## Introduction

### Prerequisites

- Authenticated access to JupyterHealth Exchange
- Authorization for patient's study and organization
- Patient consent for data types you want to retrieve
- HTTP client (curl, Postman, or programming language library)

### Understanding Data Access

#### Authorization Model

You can only access patient data if:

1. You're authorized for the patient's organization
1. Patient is enrolled in a study you manage
1. Patient has consented to share the requested data types

Reference: `jupyterhealth-exchange/core/models.py`

#### Data Formats

JupyterHealth Exchange stores observations in:

- **Open mHealth (OMH)** format: JSON data conforming to OMH schemas
- **FHIR** wrapper: Observations wrapped in FHIR resources for interoperability

The OMH data is base64-encoded inside the FHIR `valueAttachment.data` field.

Reference: `jupyterhealth-exchange/core/models.py`

______________________________________________________________________

## Viewing Data (Quick Reference)

For basic viewing of patient observations, see:

- **[View Patient's Data](patient-management.md#view-patients-data)**: View observations in Console or via REST API
- **[Monitor Study Progress](study-management.md#monitor-study-progress)**: View study-wide data collection

This guide focuses on programmatic data fetching, export, and analysis workflows.

______________________________________________________________________

## Fetch via Admin REST API

The Admin REST API provides direct access to observation data in JSON format with decoded OMH payloads.

### List Observations for a Patient

```bash
curl "https://your-jhe-instance.com/api/v1/observations?organization_id=1&study_id=10001&patient_id=10001" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

**Query Parameters**:

- `organization_id` (required): Your organization ID
- `study_id` (required): Study ID
- `patient_id` (required): Patient ID
- `limit` (optional): Number of results per page (default: 20)
- `offset` (optional): Pagination offset

Response:

```json
{
  "count": 250,
  "next": "https://your-jhe-instance.com/api/v1/observations?limit=20&offset=20&...",
  "previous": null,
  "results": [
    {
      "id": 1,
      "subjectPatient": 10001,
      "patientNameDisplay": "Smith, John",
      "codeableConcept": 1,
      "codingSystem": "https://w3id.org/openmhealth",
      "codingCode": "omh:blood-glucose:4.0",
      "codingText": "Blood Glucose",
      "dataSource": 70001,
      "dataSourceName": "iHealth",
      "status": "final",
      "valueAttachmentData": {
        "header": {
          "uuid": "550e8400-e29b-41d4-a716-446655440000",
          "schema_id": {
            "namespace": "omh",
            "name": "blood-glucose",
            "version": "4.0"
          },
          "creation_date_time": "2024-10-25T21:13:31.438Z",
          "source_creation_date_time": "2024-05-02T08:30:00-07:00"
        },
        "body": {
          "effective_time_frame": {
            "date_time": "2024-05-02T08:30:00-07:00"
          },
          "blood_glucose": {
            "value": 105,
            "unit": "mg/dL"
          },
          "temporal_relationship_to_meal": "fasting"
        }
      },
      "created": "2024-05-02T15:45:00Z",
      "lastUpdated": "2024-05-02T15:45:00Z"
    }
  ]
}
```

Reference: `jupyterhealth-exchange/core/views/observation.py`

______________________________________________________________________

## Fetch via FHIR API

The FHIR API provides standardized interoperability for health data exchange.

### Search Observations

```bash
curl "https://your-jhe-instance.com/fhir/Observation?patient._has:Group:member:_id=10001&patient=10001&code=https://w3id.org/openmhealth|omh:blood-glucose:4.0" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

**Query Parameters**:

- `patient._has:Group:member:_id`: Study ID (required)
- `patient`: Patient ID (required if study has multiple patients)
- `patient.identifier`: Alternative to patient ID (format: `system|value`)
- `code`: Filter by data type (format: `system|code`)

**At least one patient identifier** (patient ID or patient.identifier) is required.

Reference: `jupyterhealth-exchange/core/views/observation.py`

Response (FHIR Bundle):

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 250,
  "link": [{
    "relation": "next",
    "url": "https://your-jhe-instance.com/fhir/Observation?..."
  }],
  "entry": [
    {
      "fullUrl": "https://your-jhe-instance.com/fhir/Observation/1",
      "resource": {
        "resourceType": "Observation",
        "id": "1",
        "status": "final",
        "code": {
          "coding": [{
            "system": "https://w3id.org/openmhealth",
            "code": "omh:blood-glucose:4.0"
          }]
        },
        "subject": {
          "reference": "Patient/10001"
        },
        "device": {
          "reference": "Device/70001"
        },
        "identifier": [{
          "system": "https://commonhealth.org",
          "value": "obs-12345"
        }],
        "valueAttachment": {
          "contentType": "application/json",
          "data": "eyJoZWFkZXIiOnsiIHV1aWQiOiI1NTBlODQwMC1lMjliLTQxZDQtYTcxNi00NDY2NTU0NDAwMDAiLCJzY2hlbWFfaWQiOnsibmFtZXNwYWNlIjoib21oIiwibmFtZSI6ImJsb29kLWdsdWNvc2UiLCJ2ZXJzaW9uIjoiNC4wIn0sImNyZWF0aW9uX2RhdGVfdGltZSI6IjIwMjQtMTAtMjVUMjE6MTM6MzEuNDM4WiIsInNvdXJjZV9jcmVhdGlvbl9kYXRlX3RpbWUiOiIyMDI0LTA1LTAyVDA4OjMwOjAwLTA3OjAwIn0sImJvZHkiOnsiZWZmZWN0aXZlX3RpbWVfZnJhbWUiOnsiZGF0ZV90aW1lIjoiMjAyNC0wNS0wMlQwODozMDowMC0wNzowMCJ9LCJibG9vZF9nbHVjb3NlIjp7InZhbHVlIjoxMDUsInVuaXQiOiJtZy9kTCJ9LCJ0ZW1wb3JhbF9yZWxhdGlvbnNoaXBfdG9fbWVhbCI6ImZhc3RpbmcifX0="
        }
      }
    }
  ]
}
```

### Decode Base64 OMH Data

The `valueAttachment.data` field contains base64-encoded OMH JSON:

```python
import base64
import json


def decode_omh_data(fhir_observation):
    """Extract and decode OMH data from FHIR observation"""
    encoded_data = fhir_observation["valueAttachment"]["data"]

    # Decode base64
    decoded_bytes = base64.b64decode(encoded_data)

    # Parse JSON
    omh_data = json.loads(decoded_bytes.decode("utf-8"))

    return omh_data


# Example
omh_data = decode_omh_data(observation)
print(f"UUID: {omh_data['header']['uuid']}")
print(f"Data type: {omh_data['header']['schema_id']['name']}")
print(f"Value: {omh_data['body']}")
```

Output:

```python
{
    "header": {
        "uuid": "550e8400-e29b-41d4-a716-446655440000",
        "schema_id": {"namespace": "omh", "name": "blood-glucose", "version": "4.0"},
        "creation_date_time": "2024-10-25T21:13:31.438Z",
        "source_creation_date_time": "2024-05-02T08:30:00-07:00",
    },
    "body": {
        "effective_time_frame": {"date_time": "2024-05-02T08:30:00-07:00"},
        "blood_glucose": {"value": 105, "unit": "mg/dL"},
        "temporal_relationship_to_meal": "fasting",
    },
}
```

Reference: `jupyterhealth-exchange/core/models.py`

### Filter by Data Type

```bash
# Blood glucose only
curl "https://your-jhe-instance.com/fhir/Observation?patient._has:Group:member:_id=10001&patient=10001&code=https://w3id.org/openmhealth|omh:blood-glucose:4.0" \
  -H "Authorization: Bearer $ACCESS_TOKEN"

# Blood pressure only
curl "https://your-jhe-instance.com/fhir/Observation?patient._has:Group:member:_id=10001&patient=10001&code=https://w3id.org/openmhealth|omh:blood-pressure:4.0" \
  -H "Authorization: Bearer $ACCESS_TOKEN"

# Heart rate only
curl "https://your-jhe-instance.com/fhir/Observation?patient._has:Group:member:_id=10001&patient=10001&code=https://w3id.org/openmhealth|omh:heart-rate:2.0" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

### Search by Patient Identifier

If you don't know the patient's ID but have another identifier:

```bash
curl "https://your-jhe-instance.com/fhir/Observation?patient._has:Group:member:_id=10001&patient.identifier=https://hospital.example.com|MRN-12345" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Reference: `jupyterhealth-exchange/core/views/observation.py`

______________________________________________________________________

## Pagination Strategies

Both Admin REST API and FHIR API support pagination for large result sets.

### Admin REST API Pagination

The Admin REST API uses `next` and `previous` links for pagination:

```python
import requests

BASE_URL = "https://your-jhe-instance.com"
ACCESS_TOKEN = "your-access-token"


def fetch_all_observations(org_id, study_id, patient_id):
    """Fetch all observations using pagination"""
    all_observations = []
    url = f"{BASE_URL}/api/v1/observations"
    params = {
        "organization_id": org_id,
        "study_id": study_id,
        "patient_id": patient_id,
        "limit": 100,
    }

    while url:
        response = requests.get(
            url,
            headers={"Authorization": f"Bearer {ACCESS_TOKEN}"},
            params=params if params else None,
        )
        response.raise_for_status()

        data = response.json()
        all_observations.extend(data["results"])

        # Get next page
        url = data.get("next")
        params = None  # Next URL has params embedded

        print(f"Fetched {len(all_observations)} / {data['count']} observations")

    return all_observations


# Usage
observations = fetch_all_observations(org_id=1, study_id=10001, patient_id=10001)

print(f"Total observations: {len(observations)}")
```

### FHIR API Pagination

The FHIR API uses `link` elements with `relation="next"` for pagination:

```python
def fetch_all_fhir_observations(study_id, patient_id):
    """Fetch all FHIR observations using pagination"""
    all_entries = []
    url = f"{BASE_URL}/fhir/Observation"
    params = {"patient._has:Group:member:_id": study_id, "patient": patient_id}

    while url:
        response = requests.get(
            url,
            headers={"Authorization": f"Bearer {ACCESS_TOKEN}"},
            params=params if params else None,
        )
        response.raise_for_status()

        bundle = response.json()
        all_entries.extend(bundle.get("entry", []))

        # Find next link
        next_link = next(
            (link for link in bundle.get("link", []) if link.get("relation") == "next"),
            None,
        )
        url = next_link["url"] if next_link else None
        params = None  # Next URL has params embedded

        print(f"Fetched {len(all_entries)} / {bundle.get('total', 0)} entries")

    return all_entries
```

**Note**: To fetch the list of patients in a study, see [Patient Management](patient-management.md#enroll-patient-in-study).

______________________________________________________________________

## Export Data for Analysis

### Export to CSV

```python
import requests
import csv
import base64
import json
from datetime import datetime

BASE_URL = "https://your-jhe-instance.com"
ACCESS_TOKEN = "your-access-token"


def fetch_observations(org_id, study_id, patient_id):
    """Fetch all observations for a patient"""
    all_obs = []
    url = f"{BASE_URL}/api/v1/observations"
    params = {
        "organization_id": org_id,
        "study_id": study_id,
        "patient_id": patient_id,
        "limit": 100,
    }

    while url:
        response = requests.get(
            url,
            headers={"Authorization": f"Bearer {ACCESS_TOKEN}"},
            params=params if params else None,
        )
        response.raise_for_status()

        data = response.json()
        all_obs.extend(data["results"])
        url = data.get("next")
        params = None

    return all_obs


def export_blood_glucose_to_csv(observations, output_file):
    """Export blood glucose observations to CSV"""
    with open(output_file, "w", newline="") as f:
        writer = csv.writer(f)

        # Header
        writer.writerow(
            [
                "Observation ID",
                "Patient Name",
                "Date/Time",
                "Glucose (mg/dL)",
                "Meal Context",
                "Data Source",
                "Created",
            ]
        )

        # Data
        for obs in observations:
            if obs["codingCode"] != "omh:blood-glucose:4.0":
                continue

            omh_data = obs["valueAttachmentData"]
            body = omh_data["body"]

            writer.writerow(
                [
                    obs["id"],
                    obs["patientNameDisplay"],
                    body["effective_time_frame"]["date_time"],
                    body["blood_glucose"]["value"],
                    body.get("temporal_relationship_to_meal", "N/A"),
                    obs["dataSourceName"],
                    obs["created"],
                ]
            )

    print(f"Exported to {output_file}")


# Usage
observations = fetch_observations(org_id=1, study_id=10001, patient_id=10001)

export_blood_glucose_to_csv(observations, "blood_glucose.csv")
```

### Export to Pandas DataFrame

```python
import pandas as pd
import requests


def fetch_to_dataframe(org_id, study_id, patient_id, data_type):
    """Fetch observations and convert to pandas DataFrame"""

    # Fetch data
    observations = fetch_observations(org_id, study_id, patient_id)

    # Filter by data type
    filtered = [obs for obs in observations if obs["codingCode"] == data_type]

    # Convert to DataFrame
    rows = []
    for obs in filtered:
        omh_data = obs["valueAttachmentData"]
        body = omh_data["body"]

        row = {
            "observation_id": obs["id"],
            "patient_id": obs["subjectPatient"],
            "patient_name": obs["patientNameDisplay"],
            "timestamp": body["effective_time_frame"]["date_time"],
            "data_source": obs["dataSourceName"],
            "created": obs["created"],
        }

        # Add data-type-specific fields
        if data_type == "omh:blood-glucose:4.0":
            row["glucose_mg_dl"] = body["blood_glucose"]["value"]
            row["meal_context"] = body.get("temporal_relationship_to_meal")

        elif data_type == "omh:blood-pressure:4.0":
            row["systolic_mmhg"] = body["systolic_blood_pressure"]["value"]
            row["diastolic_mmhg"] = body["diastolic_blood_pressure"]["value"]

        elif data_type == "omh:heart-rate:2.0":
            row["heart_rate_bpm"] = body["heart_rate"]["value"]

        rows.append(row)

    df = pd.DataFrame(rows)

    # Convert timestamps
    df["timestamp"] = pd.to_datetime(df["timestamp"])
    df["created"] = pd.to_datetime(df["created"])

    return df


# Usage
df_glucose = fetch_to_dataframe(1, 10001, 10001, "omh:blood-glucose:4.0")
df_bp = fetch_to_dataframe(1, 10001, 10001, "omh:blood-pressure:4.0")

# Analyze
print(f"Mean glucose: {df_glucose['glucose_mg_dl'].mean():.1f} mg/dL")
print(f"Mean systolic BP: {df_bp['systolic_mmhg'].mean():.1f} mmHg")

# Save
df_glucose.to_csv("glucose_data.csv", index=False)
df_bp.to_csv("bp_data.csv", index=False)
```

### Export All Patients in Study

```python
import requests
import pandas as pd


def export_study_data(org_id, study_id, data_type):
    """Export data for all patients in a study"""

    # Get all patients
    response = requests.get(
        f"{BASE_URL}/api/v1/patients",
        headers={"Authorization": f"Bearer {ACCESS_TOKEN}"},
        params={"organization_id": org_id, "study_id": study_id},
    )
    patients = response.json()["results"]

    # Fetch data for each patient
    all_data = []
    for patient in patients:
        print(f"Fetching data for {patient['nameGiven']} {patient['nameFamily']}...")

        df = fetch_to_dataframe(org_id, study_id, patient["id"], data_type)
        all_data.append(df)

    # Combine
    combined_df = pd.concat(all_data, ignore_index=True)

    return combined_df


# Usage
study_df = export_study_data(1, 10001, "omh:blood-glucose:4.0")
study_df.to_csv("study_blood_glucose.csv", index=False)

print(
    f"Exported {len(study_df)} observations from {study_df['patient_id'].nunique()} patients"
)
```

______________________________________________________________________

## Real-Time Data Monitoring

### Poll for New Data

```python
import time
from datetime import datetime, timedelta


def monitor_new_observations(org_id, study_id, patient_id, poll_interval=60):
    """Poll for new observations every poll_interval seconds"""

    last_check = datetime.now()

    print(f"Monitoring started at {last_check}")

    while True:
        # Fetch observations created since last check
        observations = fetch_observations(org_id, study_id, patient_id)

        new_obs = [
            obs
            for obs in observations
            if datetime.fromisoformat(obs["created"].replace("Z", "+00:00"))
            > last_check
        ]

        if new_obs:
            print(f"\n{len(new_obs)} new observation(s) at {datetime.now()}:")
            for obs in new_obs:
                print(f"  - {obs['codingText']}: {obs['id']}")

        last_check = datetime.now()

        # Wait
        time.sleep(poll_interval)


# Usage
monitor_new_observations(
    org_id=1,
    study_id=10001,
    patient_id=10001,
    poll_interval=60,  # Check every 60 seconds
)
```

### Webhook Alternative

For production systems, implement webhooks instead of polling:

1. Configure webhook endpoint on your server
1. Exchange POSTs to your endpoint when new data arrives
1. Process data immediately without polling overhead

*Note: Webhook support depends on Exchange configuration.*

______________________________________________________________________

## Data Quality Checks

### Identify Missing Data

```python
def check_data_completeness(org_id, study_id, expected_data_types):
    """Check which patients have uploaded each data type"""

    # Get all patients
    response = requests.get(
        f"{BASE_URL}/api/v1/patients",
        headers={"Authorization": f"Bearer {ACCESS_TOKEN}"},
        params={"organization_id": org_id, "study_id": study_id},
    )
    patients = response.json()["results"]

    results = []

    for patient in patients:
        row = {
            "patient_id": patient["id"],
            "patient_name": f"{patient['nameGiven']} {patient['nameFamily']}",
        }

        # Check each data type
        for data_type in expected_data_types:
            observations = fetch_observations(org_id, study_id, patient["id"])
            count = sum(1 for obs in observations if obs["codingCode"] == data_type)

            row[data_type] = count

        results.append(row)

    df = pd.DataFrame(results)
    return df


# Usage
completeness = check_data_completeness(
    org_id=1,
    study_id=10001,
    expected_data_types=[
        "omh:blood-glucose:4.0",
        "omh:blood-pressure:4.0",
        "omh:heart-rate:2.0",
    ],
)

print(completeness)

# Identify patients with missing data
missing = completeness[completeness["omh:blood-glucose:4.0"] == 0]
print(f"\n{len(missing)} patients have not uploaded blood glucose data:")
print(missing["patient_name"].tolist())
```

### Detect Outliers

```python
def detect_outliers(df, column, threshold_std=3):
    """Detect statistical outliers using standard deviation"""

    mean = df[column].mean()
    std = df[column].std()

    outliers = df[
        (df[column] < mean - threshold_std * std)
        | (df[column] > mean + threshold_std * std)
    ]

    return outliers


# Usage
df_glucose = fetch_to_dataframe(1, 10001, 10001, "omh:blood-glucose:4.0")

outliers = detect_outliers(df_glucose, "glucose_mg_dl", threshold_std=3)

if len(outliers) > 0:
    print(f"Found {len(outliers)} outlier readings:")
    print(outliers[["timestamp", "glucose_mg_dl"]])
```

______________________________________________________________________

## Common Issues

### Empty Results

**Issue**: Query returns 0 observations even though data exists.

**Common Causes**:

1. **Authorization**: You're not authorized for the patient's study
   - Solution: Verify you're a member/manager of the organization
1. **Consent**: Patient hasn't consented to share data
   - Solution: Check consent status via `/api/v1/patients/{id}/consents`
1. **Wrong parameters**: Incorrect organization_id or study_id
   - Solution: Verify IDs match the patient's enrollment

Reference: Authorization checks at `jupyterhealth-exchange/core/views/observation.py`

### Base64 Decoding Error

**Error**: "Invalid base64-encoded string"

**Cause**: The `valueAttachment.data` field is already decoded (Admin API) or incorrectly formatted.

**Solution**: Admin REST API returns decoded OMH data in `valueAttachmentData`. FHIR API returns base64-encoded data in `valueAttachment.data`.

```python
# Admin API - already decoded
omh_data = observation["valueAttachmentData"]  # Already a dict

# FHIR API - needs decoding
encoded = observation["valueAttachment"]["data"]
decoded = base64.b64decode(encoded)
omh_data = json.loads(decoded.decode("utf-8"))
```

### Permission Denied

**Error**: 403 Forbidden

**Cause**: User not authorized to view this patient's data.

**Solution**: Verify authorization:

```python
# Check if practitioner is authorized for patient
python manage.py shell

from core.models import Patient, Study

patient_id = 10001
practitioner_user_id = 1
study_id = 10001

# Get authorized patients
authorized = Patient.for_practitioner_organization_study(
    practitioner_user_id=practitioner_user_id,
    organization_id=1,
    study_id=study_id
)

if authorized.filter(id=patient_id).exists():
    print("Authorized")
else:
    print("Not authorized")
```

Reference: `jupyterhealth-exchange/core/models.py`

______________________________________________________________________

## Best Practices

### Performance

1. **Use pagination**: Fetch data in chunks (100-1000 records)
1. **Cache tokens**: Reuse access tokens until expiry (10 hours default)
1. **Filter server-side**: Use query parameters instead of filtering in client
1. **Batch requests**: Fetch data for multiple patients in parallel

### Data Privacy

1. **Minimum necessary**: Only fetch data types needed for analysis
1. **De-identify**: Remove PII before storing locally
1. **Secure storage**: Encrypt exported data at rest
1. **Audit access**: Log all data retrieval operations

### Data Quality

1. **Validate timestamps**: Check for future dates or invalid formats
1. **Check units**: Verify units match expected values (mmHg, mg/dL, etc.)
1. **Detect duplicates**: Use observation IDs to prevent duplicate processing
1. **Handle missing data**: Implement strategies for incomplete datasets

### Error Handling

```python
import requests
from requests.exceptions import HTTPError, ConnectionError, Timeout
import time


def fetch_with_retry(url, headers, params, max_retries=3):
    """Fetch data with exponential backoff retry"""

    for attempt in range(max_retries):
        try:
            response = requests.get(url, headers=headers, params=params, timeout=30)
            response.raise_for_status()
            return response.json()

        except HTTPError as e:
            if e.response.status_code == 401:
                # Token expired - refresh and retry
                refresh_token()
                continue

            elif e.response.status_code == 403:
                # Permission denied - don't retry
                raise

            elif e.response.status_code >= 500:
                # Server error - retry with backoff
                wait = 2**attempt
                print(f"Server error, retrying in {wait}s...")
                time.sleep(wait)
                continue

            else:
                raise

        except (ConnectionError, Timeout) as e:
            # Network error - retry with backoff
            wait = 2**attempt
            print(f"Network error, retrying in {wait}s...")
            time.sleep(wait)
            continue

    raise Exception(f"Failed after {max_retries} attempts")
```

______________________________________________________________________

## Related Documentation

- [Using the REST API](using-rest-api.md)
- [Study Management](study-management.md)
- [Patient Management](patient-management.md)
- [FHIR Interoperability](../../explanation/fhir-interoperability.md)
- [Data Flow](../../explanation/data-flow.md)
