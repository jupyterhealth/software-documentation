# Connecting JupyterHealth to a New Wearable API

This guide shows you how to integrate a new wearable device or health data source into JupyterHealth Exchange.

### Getting Started
- [Overview](#overview)
- [Prerequisites](#prerequisites)

### Exchange Configuration
- [Add DataSource to Exchange](#add-datasource-to-exchange)
- [Map Device Data to OMH Schemas](#map-device-data-to-omh-schemas)

### Extended Setup
- [If Device Supports New Data Types](#if-device-supports-new-data-types)
- [Link DataSource to Studies](#link-datasource-to-studies)

### Mobile App Integration
- [Implement Mobile App Integration](#implement-mobile-app-integration)

### Testing and Troubleshooting
- [Test Integration](#test-integration)
- [Troubleshooting](#troubleshooting)

### Reference
- [Related Documentation](#related-documentation)

---

## Overview

JupyterHealth Exchange receives health data from mobile applications (like CommonHealth Android) that connect to wearable devices. Adding a new wearable involves:

1. Creating a DataSource entry in the Exchange
2. Mapping device data types to Open mHealth (OMH) schemas
3. Implementing device integration in the mobile app

## Prerequisites

- Access to JupyterHealth Exchange Console with a **super user** account, OR
- Django shell access, OR
- Super user API access
- Understanding of the wearable device's API
- Knowledge of Open mHealth data schemas
- Mobile app development access if implementing client-side integration

## Add DataSource to Exchange

### 1. Create DataSource

#### Using JupyterHealth Exchange Console (Recommended)

1. Login to the JupyterHealth Exchange Console at:
   ```
   https://your-jhe-instance.com/portal/
   ```

2. Login with a **super user** account (e.g., `sam@example.com`)

3. Navigate to the **Data Sources** section

4. Click the **"Add Data Source"** button

5. Fill in the form:
   - **Name**: Device manufacturer or app name (e.g., "Fitbit", "Withings", "Apple Health")
   - **Type**: Currently only `personal_device` (Personal Device) is supported

6. Click **"Create"**

7. Note the assigned ID (e.g., `70002`)

**Note**: Only super users can create data sources. Django admin does not have DataSource registered. Use the console or Django shell instead.

Reference: `jupyterhealth-exchange/core/models.py`

#### Via Django Shell (Alternative)

```bash
cd /path/to/jupyterhealth-exchange
python manage.py shell -c "from core.models import DataSource; ds = DataSource.objects.create(name='Fitbit', type='personal_device'); print(f'Created DataSource ID: {ds.id}')"
```

### 2. Via API (Alternative)

```bash
curl -X POST https://your-jhe-instance.com/api/v1/data_sources \
  -H "Authorization: Bearer $SUPER_USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Fitbit",
    "type": "personal_device"
  }'
```

**Note**: Only super users can create DataSources.

Reference: `jupyterhealth-exchange/core/permissions.py`

### 3. Link Supported Data Types

For each data type the device supports, create a `DataSourceSupportedScope` link.

#### Using JupyterHealth Exchange Console (Recommended)

1. In the **Data Sources** section, click the **View** (eye icon) button next to your data source

2. In the **Supported Scopes** section, click the **Add** button (plus icon)

3. Select the data type/scope from the dropdown (e.g., "Blood pressure", "Heart Rate")

4. Click **Add** to confirm

5. Repeat for each data type the device supports

#### Via Django Shell (Alternative)
```python
python manage.py shell

from core.models import DataSource, CodeableConcept, DataSourceSupportedScope

# Get your data source
fitbit = DataSource.objects.get(name="Fitbit")

# Get supported data types
blood_pressure = CodeableConcept.objects.get(coding_code="omh:blood-pressure:4.0")
heart_rate = CodeableConcept.objects.get(coding_code="omh:heart-rate:2.0")

# Create links
DataSourceSupportedScope.objects.create(data_source=fitbit, scope_code=blood_pressure)
DataSourceSupportedScope.objects.create(data_source=fitbit, scope_code=heart_rate)
```

## Map Device Data to OMH Schemas

### 1. Identify Available OMH Schemas

List existing schemas:

```bash
ls jupyterhealth-exchange/data/omh/json-schemas/data/
```

Common schemas:
- `schema-omh_blood-pressure_4-0.json`
- `schema-omh_blood-glucose_4-0.json`
- `schema-omh_heart-rate_2-0.json`
- `schema-omh_body-temperature_4-0.json`
- `schema-omh_oxygen-saturation_2-0.json`
- `schema-omh_respiratory-rate_2-0.json`
- `schema-omh_rr-interval_1-0.json`

### 2. Review Schema Requirements

Example: Blood pressure schema at `data/omh/json-schemas/data/schema-omh_blood-pressure_4-0.json`

```json
{
  "type": "object",
  "properties": {
    "effective_time_frame": { "$ref": "#/definitions/time_frame" },
    "systolic_blood_pressure": { "$ref": "#/definitions/systolic_blood_pressure" },
    "diastolic_blood_pressure": { "$ref": "#/definitions/diastolic_blood_pressure" },
    "position_during_measurement": { "$ref": "#/definitions/position_during_measurement" }
  },
  "required": ["effective_time_frame", "systolic_blood_pressure", "diastolic_blood_pressure"]
}
```

### 3. Create Data Transformation Logic

Document the mapping from device API format to OMH format:

**Device API Response (Fitbit example):**
```json
{
  "bp": [{
    "time": "2024-05-02T14:21:00Z",
    "systolic": 122,
    "diastolic": 77
  }]
}
```

**OMH Format (for JHE):**
```json
{
  "header": {
    "uuid": "550e8400-e29b-41d4-a716-446655440000",
    "schema_id": {
      "namespace": "omh",
      "name": "blood-pressure",
      "version": "4.0"
    },
    "source_creation_date_time": "2024-05-02T14:21:00Z",
    "modality": "sensed",
    "external_datasheets": [
      {
        "datasheet_type": "manufacturer",
        "datasheet_reference": "Fitbit"
      }
    ]
  },
  "body": {
    "effective_time_frame": {
      "date_time": "2024-05-02T14:21:00Z"
    },
    "systolic_blood_pressure": {
      "value": 122,
      "unit": "mmHg"
    },
    "diastolic_blood_pressure": {
      "value": 77,
      "unit": "mmHg"
    }
  }
}
```

Reference: Example data at `jupyterhealth-exchange/data/omh/examples/data-points/`

## If Device Supports New Data Types

If your wearable device supports data types that don't yet exist in JupyterHealth Exchange (e.g., `sleep-duration`, `body-weight`, `physical-activity`), you'll need to add those data types first before linking them to your DataSource.

**See**: [Add a Data Source, Data Type to the Exchange](add-data-source-type.md) for complete instructions on:
- Downloading OMH schemas
- Creating CodeableConcept entries
- Adding example data files
- Validating the new data type

Once the data type exists in the Exchange, return to this guide to link it to your DataSource (Step 3 in "Add DataSource to Exchange" section above).

## Link DataSource to Studies

After creating the DataSource, associate it with studies that should collect this data.

### Via API

```bash
# Add DataSource to Study
curl -X POST https://your-jhe-instance.com/api/v1/studies/10001/data_sources \
  -H "Authorization: Bearer $MANAGER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "data_source_id": 70002
  }'
```

### Using JupyterHealth Exchange Console

1. Navigate to the **Studies** section

2. Find your study and click the **View** (eye icon) button

3. In the **Data Sources** section, click the **Add** button (plus icon)

4. Select your newly created data source from the dropdown

5. Click **Add** to confirm

Reference: `jupyterhealth-exchange/core/views/study.py`

## Implement Mobile App Integration

### 1. Add Device OAuth Configuration

In your mobile app, implement OAuth flow for the new device.

Create configuration class:

```kotlin
object FitbitConfiguration : DataProviderConfiguration {
    override val authorizationEndpoint = "https://www.fitbit.com/oauth2/authorize"
    override val tokenEndpoint = "https://api.fitbit.com/oauth2/token"
    override val clientId = BuildConfig.FITBIT_CLIENT_ID
    override val clientSecret = BuildConfig.FITBIT_CLIENT_SECRET
    override val scopes = listOf("activity", "heartrate", "profile")
    override val redirectUri = "commonhealth://fitbit/callback"
}
```

### 2. Implement Data Fetching

Create repository to fetch data from device API:

```kotlin
class FitbitRepository(
    private val httpService: HTTPService,
    private val tokenManager: TokenManager
) {

    suspend fun fetchBloodPressure(since: Date?): List<BloodPressureReading> {
        val accessToken = tokenManager.getAccessToken("fitbit")

        val response = httpService.get<FitbitBPResponse>(
            url = "https://api.fitbit.com/1/user/-/bp/date/${since?.format()}/30d.json",
            headers = mapOf("Authorization" to "Bearer $accessToken")
        )

        return response.bp.map { it.toBloodPressureReading() }
    }
}
```

### 3. Transform to OMH Format

```kotlin
fun FitbitBPReading.toOpenMHealth(): OpenMHealthDataPoint {
    return OpenMHealthDataPoint(
        header = OMHHeader(
            uuid = UUID.randomUUID().toString(),
            schemaId = SchemaId(
                namespace = "omh",
                name = "blood-pressure",
                version = "4.0"
            ),
            creationDateTime = Instant.now(),
            sourceCreationDateTime = this.time
        ),
        body = mapOf(
            "effective_time_frame" to mapOf("date_time" to this.time.toString()),
            "systolic_blood_pressure" to mapOf("value" to this.systolic, "unit" to "mmHg"),
            "diastolic_blood_pressure" to mapOf("value" to this.diastolic, "unit" to "mmHg")
        )
    )
}
```

### 4. Convert to FHIR and Upload

```kotlin
fun OpenMHealthDataPoint.toFHIRObservation(
    patientId: String,
    dataSourceId: String
): Observation {
    val observation = Observation()

    // Set coding from OMH schema
    val coding = Coding()
    coding.system = "https://w3id.org/openmhealth"
    coding.code = "omh:${header.schemaId.name}:${header.schemaId.version}"
    observation.code = CodeableConcept().addCoding(coding)

    // Base64 encode OMH data
    val json = Json.encodeToString(this)
    val attachment = Attachment()
    attachment.contentType = "application/json"
    attachment.data = json.toByteArray()
    observation.value = attachment

    // Set patient and device references
    observation.subject = Reference("Patient/$patientId")
    observation.device = Reference("Device/$dataSourceId")

    // Set identifier
    observation.identifier = listOf(
        Identifier().apply {
            system = "https://commonhealth.org"
            value = UUID.randomUUID().toString()
        }
    )

    observation.status = Observation.ObservationStatus.FINAL

    return observation
}

// Upload to Exchange
suspend fun uploadObservations(observations: List<Observation>) {
    val bundle = Bundle().apply {
        type = Bundle.BundleType.BATCH
        entry = observations.map { obs ->
            Bundle.BundleEntryComponent().apply {
                resource = obs
                request = Bundle.BundleEntryRequestComponent().apply {
                    method = Bundle.HTTPVerb.POST
                    url = "Observation"
                }
            }
        }
    }

    httpClient.postJSON(
        url = "$baseUrl/fhir/r5/",
        json = fhirParser.encodeResourceToString(bundle),
        headers = mapOf("Authorization" to "Bearer $accessToken")
    )
}
```

## Test Integration

### 1. Verify DataSource is Available

```bash
# List all data sources
curl https://your-jhe-instance.com/api/v1/data_sources \
  -H "Authorization: Bearer $TOKEN"

# Check supported scopes
curl "https://your-jhe-instance.com/api/v1/data_sources?include_scopes=true" \
  -H "Authorization: Bearer $TOKEN"
```

### 2. Test Data Upload

Upload a test observation:

```bash
# Encode OMH data as base64
OMH_DATA=$(echo '{"header":{...},"body":{...}}' | base64)

# Upload observation
curl -X POST https://your-jhe-instance.com/fhir/r5/Observation \
  -H "Authorization: Bearer $PATIENT_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"resourceType\": \"Observation\",
    \"status\": \"final\",
    \"code\": {
      \"coding\": [{
        \"system\": \"https://w3id.org/openmhealth\",
        \"code\": \"omh:blood-pressure:4.0\"
      }]
    },
    \"subject\": {\"reference\": \"Patient/10001\"},
    \"device\": {\"reference\": \"Device/70002\"},
    \"identifier\": [{
      \"system\": \"https://test.example.com\",
      \"value\": \"test-bp-001\"
    }],
    \"valueAttachment\": {
      \"contentType\": \"application/json\",
      \"data\": \"$OMH_DATA\"
    }
  }"
```

Expected response: `201 Created` with observation resource

Reference: `jupyterhealth-exchange/core/views/observation.py`

### 3. Verify Data Retrieval

```bash
# Retrieve observations for patient
curl "https://your-jhe-instance.com/fhir/r5/Observation?patient=10001&code=https://w3id.org/openmhealth|omh:blood-pressure:4.0" \
  -H "Authorization: Bearer $PRACTITIONER_TOKEN"
```

Should return FHIR Bundle with your test observation.

## Troubleshooting

### DataSource Not Appearing in Study Configuration

**Cause**: DataSource may not have supported scopes defined.

**Solution**: Verify `DataSourceSupportedScope` links exist:

```python
python manage.py shell

from core.models import DataSource
ds = DataSource.objects.get(name="Your Device")
print(ds.supported_scopes.all())
```

### Upload Fails with "Device not found"

**Cause**: Device reference in FHIR Observation doesn't match a DataSource ID.

**Solution**: Ensure `device.reference` uses correct format: `Device/{datasource_id}`

Reference: `jupyterhealth-exchange/core/models.py`

### Schema Validation Error

**Cause**: OMH data doesn't match schema requirements.

**Solution**: Validate data against schema before uploading:

```bash
# Install jsonschema validator
pip install jsonschema

# Validate
python -c "
import json
from jsonschema import validate

schema = json.load(open('data/omh/json-schemas/data/schema-omh_blood-pressure_4-0.json'))
data = json.load(open('your-data.json'))

validate(instance=data['body'], schema=schema)
print('Valid!')
"
```

Reference: `jupyterhealth-exchange/core/models.py`

## Related Documentation

- [Add a Data Source, Data Type to the Exchange](add-data-source-type.md)
- [Troubleshooting Common Ingestion Errors](troubleshooting-ingestion-errors.md)
- [Open mHealth Standards](../../explanation/openmhealth-standards.md)
