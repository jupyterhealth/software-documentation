# Troubleshooting Common Ingestion Errors

This guide shows you how to diagnose and fix common data ingestion errors in JupyterHealth Exchange.

### Getting Started
- [Prerequisites](#prerequisites)

### Common Errors
- [Missing Consent Errors](#missing-consent-errors)
- [Invalid Patient Reference](#invalid-patient-reference)
- [Missing Authorization](#missing-authorization)
- [Duplicate Identifier](#duplicate-identifier)
- [Invalid Device Reference](#invalid-device-reference)
- [Schema Validation Errors](#schema-validation-errors)
- [Base64 Encoding Issues](#base64-encoding-issues)
- [Missing Code/CodeableConcept](#missing-code-codeableconcept)

### Advanced Debugging
- [Debugging Techniques](#debugging-techniques)

### Reference
- [Related Documentation](#related-documentation)

---

## Prerequisites

- Access to Django admin or shell
- Access to application logs
- Understanding of FHIR Observation structure
- (Optional) Database query access for deep debugging

## Missing Consent Errors

### Error Message

```
HTTP 403 Forbidden
{
  "resourceType": "OperationOutcome",
  "issue": [{
    "severity": "error",
    "code": "forbidden",
    "diagnostics": "Observation data with coding_system=https://w3id.org/openmhealth coding_code=omh:blood-glucose:4.0 has not been consented for any studies by this Patient."
  }]
}
```

### Cause

Patient has not granted consent to share this data type for any active study.

Reference: `jupyterhealth-exchange/core/models.py`

### Solution

#### Step 1: Verify Patient Consent Status

```bash
# Get patient consents via API
curl https://your-jhe-instance.com/api/v1/patients/10001/consents \
  -H "Authorization: Bearer $PRACTITIONER_TOKEN"
```

Check the response:
- `consolidatedConsentedScopes`: Lists all data types patient has consented to
- `studies`: Shows consent status per study
- `studiesPendingConsent`: Studies awaiting patient response

#### Step 2: Check if Study Requests This Data Type

Verify the study has requested the scope:

```python
python manage.py shell

from core.models import Study, CodeableConcept

study = Study.objects.get(id=10001)
requested_scopes = study.scope_requests.all()

for scope_request in requested_scopes:
    print(f"{scope_request.scope_code.coding_code}: {scope_request.scope_code.text}")
```

#### Step 3: Grant Consent

If patient wants to share this data, submit consent:

```bash
curl -X POST https://your-jhe-instance.com/api/v1/patients/10001/consents \
  -H "Authorization: Bearer $PATIENT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "studyScopeConsents": [{
      "studyId": 10001,
      "scopeConsents": [{
        "codingSystem": "https://w3id.org/openmhealth",
        "codingCode": "omh:blood-glucose:4.0",
        "consented": true
      }]
    }]
  }'
```

Reference: `jupyterhealth-exchange/core/views/patient.py`

## Invalid Patient Reference

### Error Message

```
HTTP 400 Bad Request
{
  "resourceType": "OperationOutcome",
  "issue": [{
    "severity": "error",
    "code": "invalid",
    "diagnostics": "Patient id=99999 can not be found"
  }]
}
```

### Cause

The `subject.reference` in the FHIR Observation points to a non-existent Patient.

Reference: `jupyterhealth-exchange/core/models.py`

### Solution

#### Step 1: Verify Patient Exists

```bash
# List patients in organization
curl https://your-jhe-instance.com/api/v1/patients?organization_id=1 \
  -H "Authorization: Bearer $PRACTITIONER_TOKEN"
```

Or via Django shell:

```python
python manage.py shell

from core.models import Patient

# Search by email
patient = Patient.objects.filter(jhe_user__email="patient@example.com").first()
if patient:
    print(f"Patient ID: {patient.id}")
else:
    print("Patient not found")
```

#### Step 2: Create Patient if Missing

If the patient doesn't exist, create them:

```bash
curl -X POST https://your-jhe-instance.com/api/v1/patients \
  -H "Authorization: Bearer $PRACTITIONER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "telecomEmail": "patient@example.com",
    "nameFamily": "Smith",
    "nameGiven": "John",
    "birthDate": "1980-01-01",
    "organizationId": 1
  }'
```

Note the returned `id` field - use this in subsequent uploads.

Reference: `jupyterhealth-exchange/core/views/patient.py`

#### Step 3: Update Upload Code

Ensure your upload code uses the correct patient ID:

```json
{
  "subject": {
    "reference": "Patient/10001"
  }
}
```

## Missing Authorization

### Error Message

```
HTTP 403 Forbidden
{
  "resourceType": "OperationOutcome",
  "issue": [{
    "severity": "error",
    "code": "forbidden",
    "diagnostics": "Current user doesn't have access to the Patient"
  }]
}
```

### Cause

The authenticated user (practitioner or patient) is not authorized to access this patient's data.

Reference: `jupyterhealth-exchange/core/models.py`

### Solution

#### For Practitioner Uploads

Verify practitioner is authorized for the patient's study:

```python
python manage.py shell

from core.models import Patient, Study

patient = Patient.objects.get(id=10001)
practitioner_user_id = 1  # Your practitioner's user ID
study_id = 1

# Check authorization
authorized_patients = Patient.for_practitioner_organization_study(
    practitioner_user_id=practitioner_user_id,
    organization_id=patient.organizations.first().id,
    study_id=study_id
)

if authorized_patients.filter(id=patient.id).exists():
    print("Practitioner is authorized")
else:
    print("Practitioner is NOT authorized")
```

Reference: `jupyterhealth-exchange/core/models.py`

#### For Patient Uploads

Ensure the authenticated patient user matches the subject:

```python
# In your API client, verify token belongs to correct patient
decoded_token = jwt.decode(access_token, verify=False)
subject_id = decoded_token['sub']  # Should match patient's jhe_user_id

print(f"Token subject: {subject_id}")
print(f"Patient jhe_user_id: {patient.jhe_user_id}")
```

## Duplicate Identifier

### Error Message

```
HTTP 400 Bad Request
{
  "resourceType": "OperationOutcome",
  "issue": [{
    "severity": "error",
    "code": "duplicate",
    "diagnostics": "Identifier already exists: system=https://commonhealth.org value=abc-123"
  }]
}
```

### Cause

An observation with this identifier was already uploaded.

Reference: `jupyterhealth-exchange/core/models.py`

### Solution

#### Step 1: Check if Observation Exists

```python
python manage.py shell

from core.models import Observation, ObservationIdentifier

# Find observation by identifier
obs_identifier = ObservationIdentifier.objects.filter(
    system="https://commonhealth.org",
    value="abc-123"
).first()

if obs_identifier:
    observation = obs_identifier.observation
    print(f"Observation exists: ID={observation.id}, Created={observation.created}")
else:
    print("Identifier not found")
```

#### Step 2: Use Unique Identifiers

Ensure each observation has a unique identifier:

```python
import uuid

# Generate unique identifier per upload
unique_id = str(uuid.uuid4())

observation = {
    "identifier": [{
        "system": "https://commonhealth.org",
        "value": unique_id
    }]
}
```

#### Step 3: Check Before Upload (Idempotent)

Implement idempotent uploads by checking for existence:

```python
async def upload_observation_idempotent(obs_data):
    identifier_value = obs_data['identifier'][0]['value']

    # Check if already uploaded
    response = await check_observation_exists(identifier_value)

    if response.status_code == 200:
        print(f"Observation {identifier_value} already exists, skipping")
        return response

    # Upload new observation
    return await upload_observation(obs_data)
```

## Invalid Device Reference

### Error Message

```
HTTP 400 Bad Request
{
  "resourceType": "OperationOutcome",
  "issue": [{
    "severity": "error",
    "code": "invalid",
    "diagnostics": "Device Data Source id=99999 can not be found"
  }]
}
```

### Cause

The `device.reference` points to a non-existent DataSource.

Reference: `jupyterhealth-exchange/core/models.py`

### Solution

#### Step 1: List Available DataSources

```bash
curl https://your-jhe-instance.com/api/v1/data_sources \
  -H "Authorization: Bearer $TOKEN"
```

Or via Django shell:

```python
python manage.py shell

from core.models import DataSource

for ds in DataSource.objects.all():
    print(f"ID: {ds.id}, Name: {ds.name}, Type: {ds.type}")
```

#### Step 2: Use Correct DataSource ID

Update your observation to reference an existing DataSource:

```json
{
  "device": {
    "reference": "Device/70001"
  }
}
```

Common DataSource IDs (from seed data):
- `70001`: iHealth
- `70002`: Dexcom
- `70003`: CareX

Reference: `jupyterhealth-exchange/core/management/commands/seed.py`

## Schema Validation Errors

### Error Message

```
HTTP 400 Bad Request
{
  "resourceType": "OperationOutcome",
  "issue": [{
    "severity": "error",
    "code": "invalid",
    "diagnostics": "'systolic_blood_pressure' is a required property"
  }]
}
```

### Cause

The OMH data in `valueAttachment.data` doesn't conform to the schema.

Reference: `jupyterhealth-exchange/core/models.py`

### Solution

#### Step 1: Retrieve Schema

```bash
# Find schema for your data type
ls jupyterhealth-exchange/data/omh/json-schemas/data/

# View schema requirements
cat jupyterhealth-exchange/data/omh/json-schemas/data/schema-omh_blood-pressure_4-0.json
```

#### Step 2: Validate Data Against Schema

Before uploading, validate your OMH data:

```python
import json
from jsonschema import validate, ValidationError

# Load schema
schema_path = "data/omh/json-schemas/data/schema-omh_blood-pressure_4-0.json"
with open(schema_path) as f:
    schema = json.load(f)

# Your data
data = {
    "effective_time_frame": {"date_time": "2024-05-02T14:21:00Z"},
    "systolic_blood_pressure": {"value": 122, "unit": "mmHg"},
    "diastolic_blood_pressure": {"value": 77, "unit": "mmHg"}
}

try:
    validate(instance=data, schema=schema)
    print("Data is valid!")
except ValidationError as e:
    print(f"Validation error: {e.message}")
```

#### Step 3: Review Example Data

Check example data for reference:

```bash
cat jupyterhealth-exchange/data/omh/examples/data-points/omh_blood-pressure_4-0.json
```

#### Step 4: Fix Common Issues

**Missing required fields:**
```json
{
  "body": {
    "effective_time_frame": {"date_time": "2024-05-02T14:21:00Z"},  // REQUIRED
    "systolic_blood_pressure": {"value": 122, "unit": "mmHg"},      // REQUIRED
    "diastolic_blood_pressure": {"value": 77, "unit": "mmHg"}       // REQUIRED
  }
}
```

**Invalid units:**
```json
{
  "systolic_blood_pressure": {
    "value": 122,
    "unit": "mmHg"  // Must be exactly "mmHg", not "mm Hg" or "millimeters of mercury"
  }
}
```

**Invalid timestamp format:**
```json
{
  "effective_time_frame": {
    "date_time": "2024-05-02T14:21:00Z"  // ISO 8601 format required, not "05/02/2024"
  }
}
```

## Base64 Encoding Issues

### Error Message

```
HTTP 400 Bad Request
{
  "resourceType": "OperationOutcome",
  "issue": [{
    "severity": "error",
    "code": "invalid",
    "diagnostics": "valueAttachment.data must be Base 64 Encoded Binary JSON"
  }]
}
```

### Cause

The `valueAttachment.data` field is not properly base64 encoded.

Reference: `jupyterhealth-exchange/core/models.py`

### Solution

#### Step 1: Verify Encoding

```python
import json
import base64

# Create OMH data
omh_data = {
    "header": {
        "uuid": "550e8400-e29b-41d4-a716-446655440000",
        "schema_id": {
            "namespace": "omh",
            "name": "blood-pressure",
            "version": "4.0"
        },
        "creation_date_time": "2024-10-25T21:13:31.438Z"
    },
    "body": {
        "effective_time_frame": {"date_time": "2024-05-02T14:21:00Z"},
        "systolic_blood_pressure": {"value": 122, "unit": "mmHg"},
        "diastolic_blood_pressure": {"value": 77, "unit": "mmHg"}
    }
}

# Convert to JSON string
json_string = json.dumps(omh_data)

# Encode as base64
encoded = base64.b64encode(json_string.encode('utf-8')).decode('utf-8')

print(f"Encoded data: {encoded}")
```

#### Step 2: Verify Decoding

Test that your encoded data can be decoded:

```python
# Decode to verify
decoded = base64.b64decode(encoded).decode('utf-8')
decoded_data = json.loads(decoded)

print(f"Decoded successfully: {decoded_data['body']}")
```

#### Step 3: Check for Common Mistakes

**Missing encoding:**
```json
// WRONG - raw JSON
{
  "valueAttachment": {
    "data": "{\"header\":{...}}"
  }
}
```

```json
// CORRECT - base64 encoded
{
  "valueAttachment": {
    "data": "eyJoZWFkZXIiOnsidXVpZCI6..."
  }
}
```

**Double encoding:**
```python
# WRONG - encoding twice
encoded_once = base64.b64encode(json_string.encode('utf-8'))
encoded_twice = base64.b64encode(encoded_once)  # Don't do this!

# CORRECT - encode once
encoded = base64.b64encode(json_string.encode('utf-8')).decode('utf-8')
```

## Missing Code/CodeableConcept

### Error Message

```
HTTP 400 Bad Request
{
  "resourceType": "OperationOutcome",
  "issue": [{
    "severity": "error",
    "code": "invalid",
    "diagnostics": "Code not found: system=https://w3id.org/openmhealth code=omh:step-count:2.0"
  }]
}
```

### Cause

The data type (CodeableConcept) doesn't exist in the database.

Reference: `jupyterhealth-exchange/core/models.py`

### Solution

#### Step 1: List Available Codes

```python
python manage.py shell

from core.models import CodeableConcept

print("Available data types:")
for cc in CodeableConcept.objects.filter(coding_system="https://w3id.org/openmhealth"):
    print(f"  {cc.coding_code}: {cc.text}")
```

Expected output:
```
Available data types:
  omh:blood-glucose:4.0: Blood Glucose
  omh:blood-pressure:4.0: Blood Pressure
  omh:heart-rate:2.0: Heart Rate
  omh:body-temperature:4.0: Body Temperature
  omh:oxygen-saturation:2.0: Oxygen Saturation
  omh:respiratory-rate:2.0: Respiratory Rate
  omh:rr-interval:1.0: RR Interval
```

#### Step 2: Create Missing CodeableConcept

If your data type is missing, create it:

```python
python manage.py shell

from core.models import CodeableConcept

CodeableConcept.objects.create(
    coding_system="https://w3id.org/openmhealth",
    coding_code="omh:step-count:2.0",
    text="Step Count"
)
```

Reference: `jupyterhealth-exchange/core/models.py`

#### Step 3: Verify Schema Exists

Ensure the corresponding OMH schema file exists:

```bash
ls -la jupyterhealth-exchange/data/omh/json-schemas/data/schema-omh_step-count_2-0.json
```

If missing, obtain from [Open mHealth schemas repository](https://github.com/openmhealth/schemas).

## Debugging Techniques

### Enable Debug Logging

Temporarily increase log verbosity:

```bash
# In .env
DJANGO_LOG_LEVEL="DEBUG"

# Restart application
sudo systemctl restart jhe
```

Watch logs in real-time:

```bash
# For systemd services
sudo journalctl -u jhe -f

# For Docker
docker logs -f jhe-container

# For development server
python manage.py runserver
```

Reference: `jupyterhealth-exchange/jhe/settings.py`

### Inspect Raw SQL Queries

For complex authorization issues, enable SQL query logging:

```python
# In Django shell
import logging

# Enable query logging
logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger('django.db.backends')
logger.setLevel(logging.DEBUG)
logger.addHandler(logging.StreamHandler())

# Run query
from core.models import Patient
patients = Patient.for_practitioner_organization_study(
    practitioner_user_id=1,
    organization_id=1,
    study_id=1
)

# SQL will be printed to console
```

Reference: `jupyterhealth-exchange/core/models.py`

### Use Django Shell for Interactive Debugging

```python
python manage.py shell

from core.models import Observation
from django.contrib.auth import get_user_model

# Test observation creation
user = get_user_model().objects.get(email="patient@example.com")

# Simulate FHIR create
data = {
    "resourceType": "Observation",
    "subject": {"reference": "Patient/10001"},
    # ... rest of observation
}

try:
    obs = Observation.fhir_create(data, user)
    print(f"Success! Observation ID: {obs.id}")
except Exception as e:
    print(f"Error: {e}")
    import traceback
    traceback.print_exc()
```

Reference: `jupyterhealth-exchange/core/models.py`

### Check Database Constraints

Verify foreign key relationships:

```sql
-- Connect to database
psql -h localhost -U jheuser -d jhe_production

-- Check patient exists
SELECT id, jhe_user_id, identifier FROM core_patient WHERE id = 10001;

-- Check data source exists
SELECT id, name, type FROM core_datasource WHERE id = 70001;

-- Check consents for patient
SELECT
    sp.patient_id,
    s.name as study_name,
    cc.coding_code,
    spsc.consented
FROM core_studypatientscopeconsent spsc
JOIN core_studypatient sp ON spsc.study_patient_id = sp.id
JOIN core_study s ON sp.study_id = s.id
JOIN core_codeableconcept cc ON spsc.scope_code_id = cc.id
WHERE sp.patient_id = 10001;
```

## Related Documentation

- [Connecting JupyterHealth to a New Wearable API](connecting-new-wearable-api.md)
- [Add a Data Source, Data Type to the Exchange](add-data-source-type.md)
- [Data Flow](../../explanation/data-flow.md)
