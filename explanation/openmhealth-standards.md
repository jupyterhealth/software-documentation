---
title: Open mHealth Data Standards
description: Learn how the Open mHealth data standards impact the JupyterHealth Exchange
---

While FHIR provides the structural foundation for JupyterHealth Exchange, Open mHealth schemas provide the standardized format for the actual wearable and device data. Together, these two standards create a powerful combination for health data interoperability.

## The Wearable Data Fragmentation Problem

Personal health devices and wearables collect valuable data, but each manufacturer uses different formats:

**Proprietary Formats**:

- Fitbit uses its own JSON structure
- Dexcom CGM has manufacturer-specific formats
- Oura Ring has custom data structures
- Most consumer wearables export in vendor-specific formats

**The Challenge**:

- Researchers must write separate parsing code for each device
- Data from different devices can't be easily compared
- Switching devices breaks existing analysis pipelines
- New device integration requires significant development effort
- Cross-study data sharing becomes nearly impossible

```{note}
Most consumer wearable manufacturers (Apple, Fitbit, Oura, etc.) do not natively export data in IEEE 1752 format. The [CommonHealth Android App](https://play.google.com/store/apps/details?id=org.thecommonsproject.android.phr) acts as the transformation layer, converting proprietary device formats into standardized IEEE 1752 schemas before uploading to JupyterHealth Exchange.
```

**The Cost**:
A researcher studying diabetes with glucose monitors might need to:

1. Write a parser for Dexcom data
1. Write a different parser for Abbott FreeStyle data
1. Write yet another parser for Medtronic data
1. Normalize timestamps, units, and field names across all three
1. Maintain all parsers as manufacturers update their formats

This data wrangling can consume more time than actual analysis.

## What is Open mHealth?

[Open mHealth](https://www.openmhealth.org/) is an initiative to create open data standards for mobile health data. It provides standardized JSON schemas for common health data types collected by wearables and personal devices.

### Mission and Goals

**Core Mission**: Make sense of patient-generated health data

**Key Goals**:

- Standardize health data formats across devices and platforms
- Enable easier integration with healthcare systems
- Support clinical trials and research
- Facilitate remote patient monitoring
- Enable machine learning on health data

### Adoption and Support

Open mHealth is used by:

- **6,000+ developers** and health organizations
- **Research institutions**: Cornell, Stanford School of Medicine, UCSF
- **Healthcare organizations**: Kaiser Permanente
- **Clinical trials** requiring device data
- **Remote monitoring programs**

## IEEE 1752 Data Schema Structure

JupyterHealth Exchange uses **IEEE 1752**, the standardized evolution of Open mHealth schemas. Every data point follows a consistent structure with two main components:

```{note}
While IEEE 1752 evolved from Open mHealth, JHE uses the IEEE standardized version with modifications to the header structure. The body schemas remain compatible with Open mHealth specifications.
```

### Header (Metadata)

The header contains metadata about the data point using IEEE 1752 standard:

```json
{
  "uuid": "aaaa1234-1a2b-3c4d-5e6f-000000000002",
  "schema_id": {
    "namespace": "omh",
    "name": "blood-pressure",
    "version": "4.0"
  },
  "source_creation_date_time": "2025-01-01T01:02:01-08:00",
  "modality": "sensed",
  "external_datasheets": [
    {
      "datasheet_type": "manufacturer",
      "datasheet_reference": "iHealth"
    }
  ]
}
```

**Key Header Fields**:

- `uuid`: Unique identifier using RFC 4122 UUID format (required)
- `schema_id`: Which schema this data follows - includes namespace, name, and version (required)
- `source_creation_date_time`: When the data was created at the original source device (required)
- `modality`: How data was collected - typically `"sensed"` for device measurements (optional)
- `external_datasheets`: References to device manufacturer or study documentation (optional)

### Body (Actual Health Data)

The body contains the actual measurement, structured according to the specific schema:

```json
{
  "effective_time_frame": {
    "date_time": "2024-10-25T14:30:00-07:00"
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
```

**Key Body Characteristics**:

- Schema-specific fields (blood pressure has systolic/diastolic)
- Consistent units (UCUM standard: mmHg, kg, etc.)
- Timestamps with timezone information
- Optional fields for additional context

## Supported Schemas in JupyterHealth Exchange

JupyterHealth Exchange includes validation for several Open mHealth schemas:

### Blood Pressure (`omh:blood-pressure:4.0`)

Measures arterial blood pressure.

**Fields**:

- `systolic_blood_pressure`: Upper pressure value (mmHg)
- `diastolic_blood_pressure`: Lower pressure value (mmHg)
- `effective_time_frame`: When measured
- `position_during_measurement`: Optional (sitting, standing, lying down)

**Example Use Cases**:

- Hypertension studies
- Medication efficacy trials
- Home blood pressure monitoring programs

### Blood Glucose (`omh:blood-glucose:4.0`)

Measures blood sugar levels.

**Fields**:

- `blood_glucose`: Glucose value (mg/dL or mmol/L)
- `effective_time_frame`: When measured
- `temporal_relationship_to_meal`: Before/after meal
- `temporal_relationship_to_sleep`: Before/after sleep

**Example Use Cases**:

- Diabetes management research
- CGM (Continuous Glucose Monitor) data collection
- Dietary intervention studies

### Heart Rate (`omh:heart-rate:2.0`)

Measures cardiac rhythm.

**Fields**:

- `heart_rate`: Beats per minute (bpm)
- `effective_time_frame`: When measured
- `temporal_relationship_to_physical_activity`: At rest, during activity, after activity

**Example Use Cases**:

- Cardiovascular health studies
- Exercise intervention research
- Sleep quality analysis

### Physical Activity (`omh:physical-activity:2.0`)

Records exercise and movement.

**Fields**:

- `activity_name`: Type of activity (walking, running, cycling, etc.)
- `distance`: Distance covered (with unit)
- `effective_time_frame`: Duration of activity
- `kcal_burned`: Energy expenditure

**Example Use Cases**:

- Activity intervention studies
- Weight management programs
- Rehabilitation tracking

### Step Count (`omh:step-count:3.0`)

Daily or periodic step counts.

**Fields**:

- `step_count`: Number of steps
- `effective_time_frame`: Time period covered

**Example Use Cases**:

- Physical activity tracking
- Sedentary behavior studies
- Post-surgical recovery monitoring

### Body Weight (`omh:body-weight:2.0`)

Body mass measurements.

**Fields**:

- `body_weight`: Weight value (kg, lb, etc.)
- `effective_time_frame`: When measured

**Example Use Cases**:

- Weight loss program tracking
- Bariatric surgery outcomes
- Nutrition intervention studies

### Sleep Duration (`omh:sleep-duration:3.0`)

Sleep period length.

**Fields**:

- `sleep_duration`: Duration (hours, minutes)
- `effective_time_frame`: Sleep period timespan

**Example Use Cases**:

- Sleep disorder research
- Circadian rhythm studies
- Mental health correlations

## How JupyterHealth Exchange Uses Open mHealth

JHE integrates Open mHealth within the FHIR framework through a clever hybrid approach:

### The Integration Pattern

**FHIR Observation Structure** (the envelope):

```json
{
  "resourceType": "Observation",
  "id": "63416",
  "status": "final",
  "code": {
    "coding": [{
      "system": "https://w3id.org/openmhealth",
      "code": "omh:blood-pressure:4.0"
    }]
  },
  "subject": {"reference": "Patient/40001"},
  "device": {"reference": "Device/70001"},
  "valueAttachment": {
    "contentType": "application/json",
    "data": "<base64_encoded_open_mhealth_json>"
  }
}
```

**Open mHealth JSON** (the content):
Stored in Base64-encoded format within the `valueAttachment.data` field.

### Why This Hybrid Approach?

**FHIR Provides**:

- Healthcare system compatibility
- Standardized resource structure (who, what, when)
- References to patients and devices
- Search and query capabilities
- Integration with EHR systems

**Open mHealth Provides**:

- Wearable-specific data structures
- Rich device data formats
- Field-level standardization (units, timestamps)
- Validation schemas
- Device community adoption

**Together They Enable**:

- Device data in healthcare systems
- Standardized wearable data queries
- Cross-device data normalization
- Research and clinical integration

## Schema Validation in JupyterHealth Exchange

JHE validates data at multiple levels to ensure quality and consistency:

### Three-Tier Validation

**Tier 1: FHIR Structure Validation**

- Validates the FHIR Observation resource structure
- Ensures required fields are present (status, code, subject)
- Checks resource references (Patient exists, Device exists)
- Performed by: `fhir.resources` Python library

**Tier 2: Open mHealth Outer Schema**

- Validates the header-body structure
- Ensures both header and body are present
- Checks for required metadata fields
- Performed by: JSON Schema validator

**Tier 3: Open mHealth Specific Schema**

- Validates against the specific schema (e.g., blood-pressure:4.0)
- Ensures correct fields for that data type
- Validates units and value ranges
- Performed by: JSON Schema validator with type-specific schema

### Validation Example Flow

When a blood pressure observation is uploaded:

1. **FHIR Validation**: Is it a valid Observation resource?
1. **Patient Check**: Does the referenced patient exist?
1. **Device Check**: Does the referenced device exist?
1. **Consent Check**: Has patient consented to share blood pressure data?
1. **Outer Schema**: Does it have a valid header and body?
1. **Blood Pressure Schema**: Does the body match blood-pressure:4.0 schema?
1. **Store**: Save to database if all checks pass

If any validation fails, the entire upload is rejected with a specific error message.

## Benefits for Different Users

### For Researchers

**Device Agnostic Analysis**:

- Write analysis code once, works with any Open mHealth-compatible device
- Compare data across different device types
- Share analysis code with other researchers

Researchers can write device-agnostic analysis code that extracts values from the standardized Open mHealth structure. Whether the blood pressure reading comes from an Omron device, a Withings cuff, or any other manufacturer, the analysis code accesses the same fields (systolic value, diastolic value, timestamp) and applies the same clinical logic, making the code portable and reusable across different data sources.

### For Device Manufacturers and App Developers

**Clear Integration Path**:

- Well-documented schemas
- JSON Schema validators available
- Example implementations
- Active developer community

**Broader Reach**:

- One integration works with many research platforms
- Reduced support burden (standardized questions)
- Easier to compete with established players

### For Patients

**Device Choice Freedom**:

- Switch devices without losing data compatibility
- Use multiple devices simultaneously
- Confidence that data will be usable in future studies

### For Healthcare Systems

**Research Integration**:

- Research data can flow into clinical systems (via FHIR)
- Clinical data can inform research studies
- Standardized formats enable automated processing

## Real-World Scenarios

### Scenario 1: Multi-Device Study

A researcher studying metabolic health wants participants to use:

- Dexcom G7 for glucose
- Oura Ring for sleep
- Apple Watch for activity

**Without Standards**: Write three separate parsers, normalize three different timestamp formats, convert three different unit systems.

**With IEEE 1752 in JHE**: The CommonHealth app (or similar data aggregator) transforms all three proprietary formats into standardized IEEE 1752 JSON before uploading to JHE. Single analysis pipeline processes all data types regardless of source device.

### Scenario 2: Device Failure Mid-Study

A participant's blood pressure monitor breaks mid-study. They switch from an iHealth BP5 to a Withings device.

**Without Standards**: Data discontinuity. May need to analyze pre-switch and post-switch data separately.

**With IEEE 1752 in JHE**: Seamless transition. Both devices' data is transformed to `omh:blood-pressure:4.0` schema by the data collection app. Analysis pipeline doesn't change.

### Scenario 3: Cross-Study Meta-Analysis

Three different institutions run diabetes studies, each using different CGM devices.

**Without Standards**: Months of data harmonization work to combine datasets.

**With IEEE 1752 in JHE**: Direct data combination. All glucose data is stored in `omh:blood-glucose:4.0` schema format regardless of original device.

## Extending Open mHealth in JupyterHealth Exchange

JHE is designed to be extensible for new data types:

### Adding a New Schema

To support a new Open mHealth schema:

1. **Add Schema File**: Place the JSON schema in `data/omh/json-schemas/`
1. **Register Schema**: Add schema reference to validation system
1. **Create CodeableConcept**: Add data type to database
1. **Test**: Upload sample data and verify validation

**Example**: Adding oxygen saturation support:

- Schema: `omh:oxygen-saturation:2.0`
- File: `schema-omh_oxygen_saturation-2-0.json`
- CodeableConcept: `{"system": "https://w3id.org/openmhealth", "code": "omh:oxygen-saturation:2.0"}`

### Custom Schemas

Organizations can also create custom schemas for proprietary devices:

```json
{
  "schema_id": {
    "namespace": "org.example",
    "name": "custom-biomarker",
    "version": "1.0"
  }
}
```

While custom schemas reduce interoperability, they enable bleeding-edge research with novel devices while maintaining the Open mHealth structure.

## Open mHealth Ecosystem Tools

### Available Resources

**Schema Repository**:

- All schemas: [github.com/openmhealth/schemas](https://github.com/openmhealth/schemas)
- Documentation: [openmhealth.org/documentation](https://www.openmhealth.org/documentation/)
- JSON Schema validators

**Data Visualization**:

- Open mHealth provides visualization libraries
- Pre-built charts for common data types
- Customizable for specific needs

**Sample Data**:

- Example data for each schema
- Testing and development support

### Integration Libraries

Libraries exist for multiple languages:

- **JavaScript**: Parse and validate Open mHealth data in web apps
- **Python**: Analysis and visualization (JupyterHealth Client uses this)
- **Java/Android**: Native mobile integration
- **iOS/Swift**: Apple ecosystem integration

## Comparing Open mHealth to Other Standards

### Open mHealth vs. Proprietary Formats

| Aspect           | Open mHealth                | Proprietary           |
| ---------------- | --------------------------- | --------------------- |
| Interoperability | High - works across devices | Low - vendor specific |
| Documentation    | Public, comprehensive       | Often limited         |
| Validation       | JSON Schema available       | Varies by vendor      |
| Community        | Open source, 6000+ devs     | Vendor controlled     |
| Cost             | Free, open standard         | May require licensing |

### Open mHealth vs. FHIR Observation

| Aspect      | Open mHealth                | FHIR Observation            |
| ----------- | --------------------------- | --------------------------- |
| Purpose     | Device data format          | Clinical observation record |
| Granularity | Very detailed for wearables | Flexible for many uses      |
| Adoption    | Wearable/device community   | Healthcare systems          |
| Validation  | JSON Schema                 | FHIR validators             |
| Best For    | Patient-generated data      | Clinical and device data    |

**JHE's Approach**: Use both together. FHIR for structure and healthcare compatibility, Open mHealth for device data richness.

## Open mHealth Standards Evolution

### Emerging Data Types

The Open mHealth community continues to develop schemas for new device types:

- Environmental sensors (air quality, noise)
- Mental health indicators (stress, mood)
- Advanced biometrics (hydration, blood oxygen variability)
- Medication adherence tracking

### FHIR and Open mHealth Convergence

The FHIR and Open mHealth communities are collaborating on:

- FHIR profiles referencing Open mHealth schemas
- Implementation guides for wearable data
- Standards convergence where appropriate

### Research Opportunities with Standardized Data

Consistent data formats enable advanced research:

- Training ML models across multiple studies
- Transfer learning between device types
- Federated learning while preserving privacy
- Algorithm validation across diverse datasets

## Learn More

**Open mHealth Resources**:

- [Open mHealth Website](https://www.openmhealth.org/)
- [Schema Documentation](https://www.openmhealth.org/documentation/)
- [GitHub Repository](https://github.com/openmhealth/schemas)

**Related JupyterHealth Exchange Documentation**:

- [Why FHIR? Interoperability Explained](./fhir-interoperability.md) - Understanding the FHIR side
- [API Reference](../reference/exchange-apis.md) - How to upload Open mHealth data
- [Architecture](../reference/exchange-architecture.md) - Technical implementation details

**Standards Organizations**:

- [HL7 FHIR](https://www.hl7.org/fhir/) - Healthcare data standards
- [IEEE 1752.1](https://standards.ieee.org/standard/1752_1-2023.html) - Mobile health data standard

## Summary

Open mHealth provides the crucial missing piece for wearable data interoperability: standardized schemas that normalize data across different device manufacturers. By combining Open mHealth's device-focused standards with FHIR's healthcare system compatibility, JupyterHealth Exchange creates a comprehensive solution for research data exchange.

**Key Takeaways**:

- Open mHealth standardizes device data formats across manufacturers
- JHE stores Open mHealth JSON within FHIR Observation resources
- Three-tier validation ensures data quality and schema compliance
- Researchers write device-agnostic analysis code
- Standards enable cross-study meta-analyses and long-term research
- The ecosystem continues growing with new schemas and tools

This hybrid approach—FHIR for structure, Open mHealth for content—represents the best of both worlds for health data interoperability.
