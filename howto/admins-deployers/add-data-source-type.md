# Add a Data Source, Data Type to the Exchange

This guide shows you how to add new data sources and data types to JupyterHealth Exchange.

### Adding a New Data Source

- [Prerequisites](#prerequisites-for-adding-data-source)
- [Step 1: Create DataSource Entry](#step-1-create-datasource-entry)
- [Step 2: Link Supported Data Types](#step-2-link-supported-data-types)
- [Step 3: Verify DataSource](#step-3-verify-datasource)

### Adding a New Data Type

- [Prerequisites](#prerequisites-for-adding-data-type)
- [Step 1: Obtain IEEE 1752 or Open mHealth Schema](#step-1-obtain-ieee-1752-or-open-mhealth-schema)
- [Step 2: Review Schema Structure](#step-2-review-schema-structure)
- [Step 3: Create Example Data Point](#step-3-create-example-data-point)
- [Step 4: Create CodeableConcept](#step-4-create-codeableconcept)
- [Step 5: Update Seed Command](#step-5-update-seed-command-optional)
- [Step 6: Verify Data Type](#step-6-verify-data-type)

### Testing New Data Source and Type

- [Step 1: Add Data Source to Study](#step-1-add-data-source-to-study)
- [Step 2: Add Scope Request to Study](#step-2-add-scope-request-to-study)
- [Step 3: Grant Patient Consent](#step-3-grant-patient-consent)
- [Step 4: Upload Test Observation](#step-4-upload-test-observation)
- [Step 5: Retrieve Test Observation](#step-5-retrieve-test-observation)

### Common Issues

- [Schema Validation Fails](#schema-validation-fails)
- [CodeableConcept Not Found](#codeableconcept-not-found)
- [DataSource Not Appearing in Study](#datasource-not-appearing-in-study)

### Advanced

<!-- - [Adding Custom (Non-OMH) Data Types](#adding-custom-non-omh-data-types) -->

### Reference

- [Related Documentation](#related-documentation)

______________________________________________________________________

## Adding a New Data Source

Data sources represent devices or applications that produce health observations (e.g., Fitbit, Apple Watch, Dexcom).

### Prerequisites for Adding Data Source

- Access to JupyterHealth Exchange Console with a **super user** account, OR
- Django shell access, OR
- Super user API access
- Understanding of the device's capabilities
- List of data types the device supports

### Step 1: Create DataSource Entry

#### Using JupyterHealth Exchange Console (Recommended)

1. Login to the JupyterHealth Exchange Console:

   ```
   https://your-jhe-instance.com/portal/
   ```

1. Login with a **super user** account (e.g., `sam@example.com`)

1. Navigate to the **Data Sources** section

1. Click the **"Add Data Source"** button

1. Fill in the modal form:

   - **Name**: Manufacturer or app name (e.g., "Apple Watch", "Omron", "Garmin")
   - **Type**: Currently only `personal_device` (Personal Device) is supported

1. Click **"Create"**

1. Note the generated ID (e.g., `70004`)

**Note**: Only super users can create data sources. The "Add Data Source" button is only visible to super users.

#### Via Django Shell

Django admin does not have DataSource registered. Use Django shell instead:

```bash
cd /path/to/jupyterhealth-exchange
python manage.py shell -c "from core.models import DataSource; ds = DataSource.objects.create(name='Apple Watch', type='personal_device'); print(f'Created DataSource ID: {ds.id}')"
```

#### Via API

```bash
curl -X POST https://your-jhe-instance.com/api/v1/data_sources \
  -H "Authorization: Bearer $SUPER_USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Apple Watch",
    "type": "personal_device"
  }'
```

Response:

```json
{
  "id": 70004,
  "name": "Apple Watch",
  "type": "personal_device"
}
```

**Note**: Only super users can create data sources.

Reference: `jupyterhealth-exchange/core/permissions.py` and `jupyterhealth-exchange/core/models.py`

### Step 2: Link Supported Data Types

For each OMH data type the device supports, create a relationship.

#### Using JupyterHealth Exchange Console (Recommended)

1. In the Data Sources list, click the **View** (eye icon) button next to your data source

1. In the modal that opens, find the **"Supported Scopes"** section

1. Click the **Add** button (plus icon)

1. Select the scope/data type from the dropdown (e.g., "Blood pressure", "Heart Rate")

1. Click **"Add"** to confirm

1. Repeat for each data type the device supports

**Note**: You can add supported scopes by clicking the "View" (eye icon) button to open the data source details modal. The "Add" button for scopes appears in the viewing modal, not in edit mode.

#### Via Django Shell

```python
python manage.py shell

from core.models import DataSource, CodeableConcept, DataSourceSupportedScope

# Get the data source
apple_watch = DataSource.objects.get(name="Apple Watch")

# Get data types
heart_rate = CodeableConcept.objects.get(coding_code="omh:heart-rate:2.0")
step_count = CodeableConcept.objects.get(coding_code="omh:step-count:2.0")
oxygen_saturation = CodeableConcept.objects.get(coding_code="omh:oxygen-saturation:2.0")

# Create relationships
DataSourceSupportedScope.objects.create(
    data_source=apple_watch,
    scope_code=heart_rate
)
DataSourceSupportedScope.objects.create(
    data_source=apple_watch,
    scope_code=step_count
)
DataSourceSupportedScope.objects.create(
    data_source=apple_watch,
    scope_code=oxygen_saturation
)

# Verify
print(f"Apple Watch supports {apple_watch.supported_scopes.count()} data types")
```

### Step 3: Verify DataSource

#### Using JupyterHealth Exchange Console

1. Navigate to the **Data Sources** section in the console

1. Find your newly created data source in the list

1. Click the **View** (eye icon) button to see details

1. Verify the data source name, type, and supported scopes are correct

#### Via API

```bash
# List all data sources with scopes
curl "https://your-jhe-instance.com/api/v1/data_sources?include_scopes=true" \
  -H "Authorization: Bearer $TOKEN"
```

Expected response includes:

```json
{
  "id": 70004,
  "name": "Apple Watch",
  "type": "personal_device",
  "supportedScopes": [
    {
      "id": 1,
      "codingSystem": "https://w3id.org/openmhealth",
      "codingCode": "omh:heart-rate:2.0",
      "text": "Heart Rate"
    }
  ]
}
```

Reference: `jupyterhealth-exchange/core/models.py`

## Adding a New Data Type

Data types define the structure and schema for health observations using Open mHealth standards.

### Prerequisites for Adding Data Type

- Django shell access (CodeableConcept not available in Django admin)
- IEEE 1752 schema for the data type (see [IEEE 1752 Schema Repository](https://opensource.ieee.org/omh/1752/-/tree/main/schemas) or [Open mHealth Schemas](https://github.com/openmhealth/schemas) for compatibility)
- Understanding of FHIR Observation structure
- Understanding of how data flows from source format to IEEE 1752 schema (see [Understanding Data Mapping](#understanding-data-mapping) below)

### Understanding Data Mapping

Before adding a new data type, it's important to understand how data flows from device-specific formats into the standardized IEEE 1752 format that JupyterHealth Exchange uses.

#### The Data Transformation Flow

```
┌─────────────────┐      ┌──────────────────┐      ┌─────────────────┐
│  Device Data    │ ───> │  IEEE 1752 JSON  │ ───> │ FHIR Observation│
│  (Proprietary)  │      │  (Standardized)  │      │   (Storage)     │
└─────────────────┘      └──────────────────┘      └─────────────────┘
   Fitbit, Apple,          header + body             valueAttachment
   Dexcom, etc.           validated schema           with base64 data
```

**Step-by-Step:**

1. **Source Data Collection**: Devices (Fitbit, Apple Watch, Dexcom, etc.) collect health data in their proprietary formats
1. **Transformation to IEEE 1752**: A data collection app (like CommonHealth Android App) transforms the proprietary format into standardized IEEE 1752 JSON format
1. **FHIR Observation Creation**: The IEEE 1752 JSON is base64-encoded and stored in a FHIR Observation resource's `valueAttachment` field
1. **Schema Validation**: JupyterHealth Exchange validates the data against the IEEE 1752 schema before accepting it

#### Where Schemas Are Stored

IEEE 1752 schema files are stored in the JupyterHealth Exchange codebase at:

```
jupyterhealth-exchange/data/omh/json-schemas/
├── data/          # Data type schemas (blood-pressure, heart-rate, etc.)
├── metadata/      # Header and data-point structure schemas
└── utility/       # Utility schemas (time-frame, unit-value, etc.)
```

**Naming Convention**: Schema files follow the pattern `schema-{coding_code}.json` where:

- `:` is replaced with `_`
- `.` is replaced with `-`

**Example**: The coding code `omh:blood-pressure:4.0` maps to the file `schema-omh_blood-pressure_4-0.json`

#### How Validation Works

When an Observation is created or updated, JupyterHealth Exchange performs validation in `core/models.py`:

1. **Header Validation**: Validates the IEEE 1752 header structure against `header-1.0.json`
1. **Body Validation**: Loads the specific schema file based on the `CodeableConcept.coding_code` and validates the body data
1. **Schema Registry**: Uses a preloaded schema registry (`core/utils.py`) to resolve schema references without network calls

**Code References**:

- Schema validation logic: `jupyterhealth-exchange/core/models.py:1484-1503` (Observation.clean() method)
- Schema registry builder: `jupyterhealth-exchange/core/utils.py:36-58` (build_schema_registry() function)

#### Where the Mapping Happens

The transformation from proprietary device format to IEEE 1752 format typically happens **outside** of JupyterHealth Exchange:

- **Mobile Apps**: The CommonHealth Android App reads device data via APIs (Google Fit, Apple HealthKit, etc.) and transforms it to IEEE 1752 format
- **Integration Services**: Custom integrations may transform API responses from devices (Dexcom, Fitbit, etc.) into IEEE 1752 format
- **Manual Tools**: Researchers can use scripts to convert exported device data to IEEE 1752 format

**JupyterHealth Exchange's Role**: JHE doesn't perform the transformation—it validates that incoming data conforms to the IEEE 1752 schema and stores it in FHIR format.

For more details on IEEE 1752 and Open mHealth standards, see [Open mHealth Data Standards](../../explanation/openmhealth-standards.md).

### Step 1: Obtain IEEE 1752 or Open mHealth Schema

#### Check if Schema Exists

JupyterHealth Exchange uses **IEEE 1752** schemas (the standardized evolution of Open mHealth). Check for available schemas:

**Primary Source - IEEE 1752 Repository**:

```
https://opensource.ieee.org/omh/1752/-/tree/main/schemas
```

**Compatible Source - Open mHealth Repository** (for reference and compatibility):

```
https://github.com/openmhealth/schemas/tree/master/schema
```

```{note}
IEEE 1752 is the official IEEE standard that evolved from Open mHealth. JupyterHealth Exchange uses a hybrid approach:
- **Header/metadata schemas**: IEEE 1752 standard (header-1.0, data-point-1.0, etc.)
- **Data type body schemas**: Open mHealth schemas with IEEE 1752 utility references
- **Utility schemas**: Mix of IEEE 1752 and Open mHealth (time-frame, unit-value, etc.)

The data type schemas (blood-pressure, heart-rate, etc.) retain their Open mHealth namespace but reference IEEE 1752 utility schemas for common components. This ensures compatibility while leveraging the IEEE standard's improvements.
```

Common schemas available:

- `heart-rate` (v2.0)
- `blood-pressure` (v4.0)
- `blood-glucose` (v4.0)
- `body-temperature` (v4.0)
- `oxygen-saturation` (v2.0)
- `respiratory-rate` (v2.0)
- `step-count` (v2.0)
- `physical-activity` (v2.0)
- `body-weight` (v2.0)
- `body-mass-index` (v2.0)

```{note}
**Schema Versions**: The versions listed above represent the schemas currently supported in JupyterHealth Exchange. Some Open mHealth schemas have newer versions (e.g., step-count v3.0) that reference IEEE 1752 schemas and mark older versions as deprecated. Before adding a new data type:

1. Check which version is currently used in the JHE codebase (`jupyterhealth-exchange/data/omh/json-schemas/data/`)
2. If you need a newer version, ensure the schema file and all its dependencies are added to the repository
3. Update the CodeableConcept with the correct version number

The examples in this documentation use the versions currently available in the JupyterHealth Exchange repository.
```

#### Download Schema

```bash
cd /opt/jupyterhealth-exchange/data/omh/json-schemas/data/

# Download schema (example: step count)
wget https://raw.githubusercontent.com/openmhealth/schemas/master/schema/omh/step-count-2.0.json

# Rename to JHE naming convention
mv step-count-2.0.json schema-omh_step-count_2-0.json
```

**Naming Convention**: `schema-omh_{type-name}_{version-with-dashes}.json`

Examples:

- `schema-omh_heart-rate_2-0.json`
- `schema-omh_body-weight_2-0.json`

### Step 2: Review Schema Structure

Understand the schema requirements:

```bash
cat schema-omh_step-count_2-0.json
```

Example schema structure:

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "effective_time_frame": {
      "$ref": "time-frame-1.x.json"
    },
    "step_count": {
      "type": "integer"
    }
  },
  "required": ["effective_time_frame", "step_count"]
}
```

Note the required fields - these will be validated on upload.

### Step 3: Create Example Data Point

Create example file for reference:

```bash
cd /opt/jupyterhealth-exchange/data/omh/examples/data-points/

nano omh_step-count_2-0.json
```

Content:

```json
{
  "header": {
    "uuid": "d9c46d90-4e36-4c3e-a331-28d7c2b84c4d",
    "schema_id": {
      "namespace": "omh",
      "name": "step-count",
      "version": "2.0"
    },
    "source_creation_date_time": "2024-05-02T00:00:00Z",
    "modality": "sensed",
    "external_datasheets": [
      {
        "datasheet_type": "manufacturer",
        "datasheet_reference": "Apple Watch"
      }
    ]
  },
  "body": {
    "effective_time_frame": {
      "time_interval": {
        "start_date_time": "2024-05-02T00:00:00Z",
        "end_date_time": "2024-05-02T23:59:59Z"
      }
    },
    "step_count": 8725
  }
}
```

**Header Fields**:

*Required:*

- `uuid`: Unique identifier for this data point (RFC 4122 UUID format)
- `schema_id`: Must match your schema with `namespace`, `name`, and `version`
- `source_creation_date_time`: When the measurement was taken by the device

*Optional:*

- `modality`: How the data was obtained (`"sensed"` or `"self-reported"`)
- `external_datasheets`: References to documentation about the device/software
  - `datasheet_type`: Type of component (e.g., `"manufacturer"`, `"software"`, `"study"`)
  - `datasheet_reference`: Name or IRI of the datasheet
- `acquisition_rate`: Rate at which measures are acquired (frequency_unit_value)

**Body Requirements**:

- `effective_time_frame`: Required for all OMH data (when measurement occurred)
- Other fields per schema specification

Reference: Example at `jupyterhealth-exchange/data/omh/examples/data-points/omh_blood-pressure_4-0.json`

### Step 4: Create CodeableConcept

#### Via Django Shell

Django admin does not have CodeableConcept registered. Use Django shell:

```bash
cd /path/to/jupyterhealth-exchange
python manage.py shell -c "from core.models import CodeableConcept; cc = CodeableConcept.objects.create(coding_system='https://w3id.org/openmhealth', coding_code='omh:step-count:2.0', text='Step Count'); print(f'Created CodeableConcept ID: {cc.id}')"
```

Or interactively:

```python
python manage.py shell

from core.models import CodeableConcept

# Create the data type
step_count = CodeableConcept.objects.create(
    coding_system="https://w3id.org/openmhealth",
    coding_code="omh:step-count:2.0",
    text="Step Count"
)

print(f"Created CodeableConcept ID: {step_count.id}")
```

**Coding Code Format**: `omh:{schema-name}:{version}`

Reference: `jupyterhealth-exchange/core/models.py`

### Step 5: Update Seed Command (Optional)

For commonly used data types, add to seed command:

```bash
nano /opt/jupyterhealth-exchange/core/management/commands/seed.py
```

Add to `seed_codeable_concept()` function (around line 78-110):

```python
def seed_codeable_concept(apps):
    CodeableConcept = apps.get_model("core", "CodeableConcept")

    # ... existing code ...

    CodeableConcept.objects.get_or_create(
        coding_system="https://w3id.org/openmhealth",
        coding_code="omh:step-count:2.0",
        defaults={"text": "Step Count"},
    )

    CodeableConcept.objects.get_or_create(
        coding_system="https://w3id.org/openmhealth",
        coding_code="omh:body-weight:2.0",
        defaults={"text": "Body Weight"},
    )
```

Reference: `jupyterhealth-exchange/core/management/commands/seed.py`

### Step 6: Verify Data Type

```python
python manage.py shell

from core.models import CodeableConcept

# List all OMH data types
omh_types = CodeableConcept.objects.filter(
    coding_system="https://w3id.org/openmhealth"
)

print("Available OMH Data Types:")
for cc in omh_types:
    print(f"  {cc.coding_code}: {cc.text} (ID: {cc.id})")
```

## Testing New Data Source and Type

### Step 1: Add Data Source to Study

#### Using JupyterHealth Exchange Console

1. Navigate to the **Studies** section

1. Find your study and click the **View** (eye icon) button

1. In the **Data Sources** section, click the **Add** button (plus icon)

1. Select your newly created data source from the dropdown

1. Click **Add** to confirm

#### Via API

```bash
# Associate data source with study
curl -X POST https://your-jhe-instance.com/api/v1/studies/10001/data_sources \
  -H "Authorization: Bearer $MANAGER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "data_source_id": 70004
  }'
```

Reference: `jupyterhealth-exchange/core/views/study.py`

### Step 2: Add Scope Request to Study

#### Using JupyterHealth Exchange Console

1. Navigate to the **Studies** section

1. Find your study and click the **View** (eye icon) button

1. In the **Scope Requests** section, click the **Add** button (plus icon)

1. Select your newly created data type/scope from the dropdown

1. Click **Add** to confirm

#### Via API

```bash
# Request step count data for study
curl -X POST https://your-jhe-instance.com/api/v1/studies/10001/scope_requests \
  -H "Authorization: Bearer $MANAGER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "scope_code_id": 8
  }'
```

Replace `8` with the ID of your CodeableConcept.

Reference: `jupyterhealth-exchange/core/views/study.py`

### Step 3: Grant Patient Consent

```{note}
**JHE Console Access**: The JupyterHealth Exchange Console is for **Practitioners and Admins only**. Patients cannot and should not log into the Console. Patient consent is managed either:
- By practitioners via the Console (on behalf of patients)
- By patients via API or mobile app (CommonHealth)
```

#### Using JupyterHealth Exchange Console (Practitioners Only)

Practitioners with **manager** or **member** roles can grant consent on behalf of patients:

1. Login as a practitioner with manager/member role (e.g., `mary@example.com`, `megan@example.com`, `mark@example.com`)

1. Navigate to the **Patients** section

1. Find and view the patient profile

1. In the **Studies Pending Consent** section, grant consent on behalf of the patient

**Note**: Practitioners with **viewer** role cannot grant consent.

#### Via API (Patients or Practitioners)

```bash
# Patient consents to share step count data
curl -X POST https://your-jhe-instance.com/api/v1/patients/10001/consents \
  -H "Authorization: Bearer $PATIENT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "studyScopeConsents": [{
      "studyId": 10001,
      "scopeConsents": [{
        "codingSystem": "https://w3id.org/openmhealth",
        "codingCode": "omh:step-count:2.0",
        "consented": true
      }]
    }]
  }'
```

Reference: `jupyterhealth-exchange/core/views/patient.py`

### Step 4: Upload Test Observation

**Note**: Uploading observations can only be done via the FHIR API. There is no console UI for uploading observations.

Prepare test data:

```json
{
  "resourceType": "Observation",
  "status": "final",
  "code": {
    "coding": [{
      "system": "https://w3id.org/openmhealth",
      "code": "omh:step-count:2.0"
    }]
  },
  "subject": {
    "reference": "Patient/10001"
  },
  "device": {
    "reference": "Device/70004"
  },
  "identifier": [{
    "system": "https://test.example.com",
    "value": "test-step-count-001"
  }],
  "valueAttachment": {
    "contentType": "application/json",
    "data": "BASE64_ENCODED_OMH_DATA_HERE"
  }
}
```

Generate base64 data:

```python
import json
import base64

omh_data = {
    "header": {
        "uuid": "d9c46d90-4e36-4c3e-a331-28d7c2b84c4d",
        "schema_id": {"namespace": "omh", "name": "step-count", "version": "2.0"},
        "creation_date_time": "2024-10-25T21:13:31.438Z",
        "source_creation_date_time": "2024-05-02T00:00:00Z",
    },
    "body": {
        "effective_time_frame": {
            "time_interval": {
                "start_date_time": "2024-05-02T00:00:00Z",
                "end_date_time": "2024-05-02T23:59:59Z",
            }
        },
        "step_count": 8725,
    },
}

json_string = json.dumps(omh_data)
encoded = base64.b64encode(json_string.encode("utf-8")).decode("utf-8")
print(encoded)
```

Upload:

```bash
curl -X POST https://your-jhe-instance.com/FHIR/R5/Observation \
  -H "Authorization: Bearer $PATIENT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "resourceType": "Observation",
    "status": "final",
    "code": {
      "coding": [{
        "system": "https://w3id.org/openmhealth",
        "code": "omh:step-count:2.0"
      }]
    },
    "subject": {"reference": "Patient/10001"},
    "device": {"reference": "Device/70004"},
    "identifier": [{
      "system": "https://test.example.com",
      "value": "test-step-count-001"
    }],
    "valueAttachment": {
      "contentType": "application/json",
      "data": "'$ENCODED_DATA'"
    }
  }'
```

Expected response: `201 Created`

Reference: `jupyterhealth-exchange/core/models.py`

### Step 5: Retrieve Test Observation

#### Using JupyterHealth Exchange Console

1. Navigate to the **Observations** section

1. Use the filters to select the organization and study

1. View the list of observations including your newly uploaded test data

1. Click the **View** (eye icon) button to see the full observation details

#### Via API

```bash
# Retrieve observations for patient
curl "https://your-jhe-instance.com/FHIR/R5/Observation?patient=10001&code=https://w3id.org/openmhealth|omh:step-count:2.0" \
  -H "Authorization: Bearer $PRACTITIONER_TOKEN"
```

Should return FHIR Bundle with your observation.

Reference: `jupyterhealth-exchange/core/views/observation.py`

## Common Issues

### Schema Validation Fails

**Error**: "Required property missing" or similar schema validation error.

**Solution**:

1. Validate your OMH data against the schema:

```python
import json
from jsonschema import validate, ValidationError

# Load schema
with open("data/omh/json-schemas/data/schema-omh_step-count_2-0.json") as f:
    schema = json.load(f)

# Load your data body
data = {
    "effective_time_frame": {
        "time_interval": {
            "start_date_time": "2024-05-02T00:00:00Z",
            "end_date_time": "2024-05-02T23:59:59Z",
        }
    },
    "step_count": 8725,
}

try:
    validate(instance=data, schema=schema)
    print("Valid!")
except ValidationError as e:
    print(f"Validation error: {e.message}")
    print(f"Path: {list(e.path)}")
```

2. Compare with example data:

```bash
cat data/omh/examples/data-points/omh_step-count_2-0.json
```

Reference: `jupyterhealth-exchange/core/models.py`

### CodeableConcept Not Found

**Error**: "Code not found: system=... code=..."

**Solution**:
Verify CodeableConcept exists:

```python
python manage.py shell

from core.models import CodeableConcept

try:
    cc = CodeableConcept.objects.get(
        coding_system="https://w3id.org/openmhealth",
        coding_code="omh:step-count:2.0"
    )
    print(f"Found: {cc.text} (ID: {cc.id})")
except CodeableConcept.DoesNotExist:
    print("CodeableConcept not found - create it first")
```

Reference: `jupyterhealth-exchange/core/models.py`

### DataSource Not Appearing in Study

**Solution**:
Verify DataSourceSupportedScope links exist:

```python
python manage.py shell

from core.models import DataSource

ds = DataSource.objects.get(name="Apple Watch")
scopes = ds.supported_scopes.all()

if scopes.count() == 0:
    print("No supported scopes defined!")
else:
    for scope in scopes:
        print(f"  {scope.coding_code}: {scope.text}")
```

<!--
## Adding Custom (Non-OMH) Data Types

If you need to add a proprietary data type not from Open mHealth:

### Step 1: Define Custom Schema

```bash
cd /opt/jupyterhealth-exchange/data/custom-schemas/

nano schema-custom_glucose-meter_1-0.json
```

Content:

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "timestamp": {
      "type": "string",
      "format": "date-time"
    },
    "glucose_mg_dl": {
      "type": "number"
    },
    "meal_context": {
      "type": "string",
      "enum": ["fasting", "before_meal", "after_meal"]
    }
  },
  "required": ["timestamp", "glucose_mg_dl"]
}
```

### Step 2: Create CodeableConcept with Custom System

```python
python manage.py shell

from core.models import CodeableConcept

custom_type = CodeableConcept.objects.create(
    coding_system="https://yourdomain.com/schemas",
    coding_code="custom:glucose-meter:1.0",
    text="Custom Glucose Meter Reading"
)
```

### Step 3: Update Validation Logic

Edit `core/models.py` to handle custom schemas:

```python
def clean(self):
    # Existing OMH validation...

    # Add custom schema validation
    if self.codeable_concept.coding_system == "https://yourdomain.com/schemas":
        schema_path = (
            settings.DATA_DIR_PATH.custom_schemas
            / f"schema-{self.codeable_concept.coding_code.replace(':', '_')}.json"
        )

        if schema_path.exists():
            schema = json.loads(schema_path.read_text())
            validate_with_registry(
                instance=value_attachment_data.get("body"), schema=schema
            )
```

**Note**: Custom schemas require code changes and should be avoided if possible. Prefer using or extending Open mHealth schemas.
-->

## Related Documentation

- [Connecting JupyterHealth to a New Wearable API](connecting-new-wearable-api.md)
- [Troubleshooting Common Ingestion Errors](troubleshooting-ingestion-errors.md)
- [Open mHealth Standards](../../explanation/openmhealth-standards.md)
