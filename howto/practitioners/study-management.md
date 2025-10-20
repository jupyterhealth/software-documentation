# Study Management

This guide shows you how to manage research studies in JupyterHealth Exchange using either the web-based **Console** (Method 1) or the **REST API** (Method 2).

### Introduction

- [Prerequisites](#prerequisites)
- [Study Attributes](#study-attributes)

### Core Study Operations

- [Create Study](#create-study)
- [View Study](#view-study)
- [Update Study Details](#update-study-details)
- [Configure Data Collection](#configure-data-collection)
- [Delete Study](#delete-study)

### Advanced Topics

- [Enroll Patients in Study](#enroll-patients-in-study)
- [Monitor Study Progress](#monitor-study-progress)
- [Common Issues](#common-issues)
- [Best Practices](#best-practices)
- [Complete Study Setup Example](#complete-study-setup-example)

### Reference

- [Choosing Between Console and REST API](#choosing-between-console-and-rest-api)
- [Related Documentation](#related-documentation)

______________________________________________________________________

## Introduction

### Prerequisites

**For Console Access:**

- Practitioner account with login credentials
- Manager or member role in an organization
- List of data types you want to collect
- List of data sources (devices) participants will use

**For REST API Access:**

- OAuth2 access token
- Manager or member role in an organization
- HTTP client (curl, Python, JavaScript, etc.)

Choose the method that best fits your workflow.

### Study Attributes

A study in JupyterHealth Exchange represents a research project or clinical program that:

- Belongs to a single organization
- Requests specific types of health data (scopes)
- Supports specific data sources (devices/apps)
- Enrolls patients who consent to share their data

Reference: `jupyterhealth-exchange/core/models.py`

______________________________________________________________________

## Create Study

### Method 1: Using JupyterHealth Exchange Console

The Console allows you to create studies through the Organizations page.

#### Prerequisites

- Practitioner account in JupyterHealth Exchange
- Manager or member role in an organization
- Study name and description prepared
- Access to JupyterHealth Exchange web portal

#### Step 1: Navigate to Organizations Page

1. Log in to the Console:

   ```
   https://your-jhe-instance.com/portal/
   ```

1. Click the **Organizations** tab in the navigation

#### Step 2: Open Your Organization

1. Find your organization in the table
1. Click the **"eye" icon** (View button) to view organization details
1. A modal opens showing organization information

#### Step 3: Create New Study

1. In the organization details modal, scroll down to the **"Studies"** section

1. Click the **"+" icon** (Add Study button) next to "Studies"

1. The Studies modal closes and a "Create Study" modal opens

1. Fill in the study form:

   - **Name** (required): e.g., "Diabetes Management Study"
   - **Description** (required): Detailed study description
   - **Icon URL** (optional): URL to study icon/logo
     - As you type, the icon preview updates in real-time
   - **Organization Name**: Auto-filled (read-only)

1. Click **"Create"** button

1. The study is created and the modal closes

#### Step 4: Configure Study (Optional)

After creating the study, you can immediately configure it:

1. Navigate to the **Studies** tab
1. Select your organization from the dropdown
1. Find your new study in the table
1. Click the **"eye" icon** to view study details
1. Add data sources and scope requests (see [Configure Data Collection](#configure-data-collection))

**Note**: New studies are always created from the Organizations page. The Studies tab is for viewing, editing, and deleting existing studies.

### Method 2: Using REST API

The REST API enables programmatic study creation, ideal for bulk imports and automated workflows.

#### Prerequisites

- OAuth2 access token (practitioner authentication)
- Manager or member role in an organization
- HTTP client (curl, Postman, or programming language)

#### Step 1: Authenticate

Obtain an access token as a practitioner. See [Using the REST API](using-rest-api.md#authentication) for the full authentication flow.

```bash
ACCESS_TOKEN="your-access-token-here"
```

#### Step 2: Identify Your Organization

List organizations you belong to:

```bash
curl https://your-jhe-instance.com/api/v1/organizations \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Response:

```json
[
  {
    "id": 1,
    "name": "Example Clinic",
    "type": "prov",
    "practitioners": [
      {
        "id": 1,
        "email": "you@example.com",
        "role": "manager"
      }
    ]
  }
]
```

Note your organization ID (e.g., `1`) and verify your role is `manager` or `member`.

Reference: `jupyterhealth-exchange/core/permissions.py`

#### Step 3: Create the Study

```bash
curl -X POST https://your-jhe-instance.com/api/v1/studies \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Diabetes Management Study",
    "description": "Monitoring blood glucose levels in Type 2 diabetes patients to optimize treatment plans.",
    "organization": 1,
    "iconUrl": "https://example.com/study-icon.png"
  }'
```

**Fields**:

- `name` (required): Study name (e.g., "Diabetes Management Study")
- `description` (required): Detailed study description
- `organization` (required): Organization ID
- `iconUrl` (optional): URL to study icon/logo

Response:

```json
{
  "id": 10001,
  "name": "Diabetes Management Study",
  "description": "Monitoring blood glucose levels...",
  "organization": 1,
  "iconUrl": "https://example.com/study-icon.png",
  "patients": [],
  "scopeRequests": [],
  "dataSources": []
}
```

Note the study ID (e.g., `10001`) for subsequent steps.

Reference: `jupyterhealth-exchange/core/views/study.py`

______________________________________________________________________

## Configure Data Collection

After creating a study, you need to configure what types of data to collect and which devices patients can use.

### Method 1: Using JupyterHealth Exchange Console

The Console provides an interactive interface for managing study data sources and scope requests.

#### Prerequisites

- Study must already be created
- Manager or member role in the study's organization
- Know which data types and devices you want to support

#### Step 1: Navigate to Study Details

1. Log in to the Console and click the **Studies** tab
1. Select your **Organization** from the dropdown
1. Find your study in the table
1. Click the **"eye" icon** (View button) to open study details modal

#### Step 2: Add Data Sources (Devices)

In the study details modal:

1. Scroll down to the **"Data Sources"** section

1. Click the **"+" icon** next to "Data Sources"

1. A dropdown appears with available data sources:

   - Select a data source (e.g., "iHealth", "Dexcom", "CareX")

1. Click the **"Add"** button (green)

1. The data source appears in the list

1. Repeat for additional data sources

1. To remove a data source:

   - Click the **trash icon** next to the data source name
   - Confirmation is instant

#### Step 3: Add Scope Requests (Data Types)

In the same study details modal:

1. Scroll down to the **"Scope Requests"** section

1. Click the **"+" icon** next to "Scope Requests"

1. A dropdown appears with available data types:

   - Select a scope (e.g., "Blood Glucose", "Blood Pressure", "Heart Rate")

1. Click the **"Add"** button (green)

1. The scope request appears in the list

1. Repeat for additional data types

1. To remove a scope request:

   - Click the **trash icon** next to the scope name
   - **Warning**: Removing scope requests prevents collection of that data type going forward

**Console Tips:**

- You can add/remove data sources and scopes at any time
- Changes take effect immediately
- Adding new scope requests requires patients to provide additional consent
- The modal updates in real-time as you make changes

### Method 2: Using REST API

The REST API enables programmatic configuration of study data collection.

#### Prerequisites

- OAuth2 access token
- Study ID from study creation
- Manager or member role in the organization

#### Step 1: Add Data Types (Scope Requests)

Specify which types of health data the study will collect.

**List Available Data Types:**

```bash
curl https://your-jhe-instance.com/api/v1/data_sources \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Look for `supportedScopes` in the response to see available data types:

```json
{
  "supportedScopes": [
    {
      "id": 1,
      "codingSystem": "https://w3id.org/openmhealth",
      "codingCode": "omh:blood-glucose:4.0",
      "text": "Blood Glucose"
    },
    {
      "id": 2,
      "codingCode": "omh:blood-pressure:4.0",
      "text": "Blood Pressure"
    }
  ]
}
```

**Add Scope Requests to Study:**

```bash
# Add blood glucose
curl -X POST https://your-jhe-instance.com/api/v1/studies/10001/scope_requests \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "scope_code_id": 1
  }'

# Add blood pressure
curl -X POST https://your-jhe-instance.com/api/v1/studies/10001/scope_requests \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "scope_code_id": 2
  }'

# Add heart rate
curl -X POST https://your-jhe-instance.com/api/v1/studies/10001/scope_requests \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "scope_code_id": 4
  }'
```

**Note**: Each scope request represents one data type. Add as many as needed for your study.

Reference: `jupyterhealth-exchange/core/views/study.py`

**Verify Scope Requests:**

```bash
curl "https://your-jhe-instance.com/api/v1/studies?organization_id=1" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Response includes `scopeRequests`:

```json
{
  "id": 10001,
  "name": "Diabetes Management Study",
  "scopeRequests": [
    {
      "id": 1,
      "codingCode": "omh:blood-glucose:4.0",
      "text": "Blood Glucose"
    },
    {
      "id": 2,
      "codingCode": "omh:blood-pressure:4.0",
      "text": "Blood Pressure"
    }
  ]
}
```

#### Step 2: Add Data Sources (Devices)

Specify which devices/apps participants can use.

**List Available Data Sources:**

```bash
curl https://your-jhe-instance.com/api/v1/data_sources \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Response:

```json
[
  {
    "id": 70001,
    "name": "iHealth",
    "type": "personal_device"
  },
  {
    "id": 70002,
    "name": "Dexcom",
    "type": "personal_device"
  },
  {
    "id": 70003,
    "name": "CareX",
    "type": "personal_device"
  }
]
```

**Add Data Sources to Study:**

```bash
# Add iHealth
curl -X POST https://your-jhe-instance.com/api/v1/studies/10001/data_sources \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "data_source_id": 70001
  }'

# Add Dexcom
curl -X POST https://your-jhe-instance.com/api/v1/studies/10001/data_sources \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "data_source_id": 70002
  }'
```

Reference: `jupyterhealth-exchange/core/views/study.py`

**Verify Data Sources:**

```bash
curl "https://your-jhe-instance.com/api/v1/studies?organization_id=1" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Response includes `dataSources`:

```json
{
  "id": 10001,
  "name": "Diabetes Management Study",
  "dataSources": [
    {
      "id": 70001,
      "name": "iHealth",
      "type": "personal_device"
    },
    {
      "id": 70002,
      "name": "Dexcom",
      "type": "personal_device"
    }
  ]
}
```

______________________________________________________________________

## View Study

You can view study details using either the Console or REST API.

### Method 1: Using JupyterHealth Exchange Console

1. Navigate to the **Studies** tab in the Console
1. Select your **Organization** from the dropdown
1. Find the study in the table
1. Click the **"eye" icon** (View button)
1. A modal opens showing:
   - Study ID
   - Name and description
   - Icon URL (with live preview)
   - Organization name
   - **Data Sources**: List of supported devices
   - **Scope Requests**: List of data types being collected

### Method 2: Using REST API

```bash
curl "https://your-jhe-instance.com/api/v1/studies?organization_id=1" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Response includes all studies for the organization with full details.

Reference: `jupyterhealth-exchange/core/views/study.py`

______________________________________________________________________

## Update Study Details

You can update study name, description, and icon URL using either the Console or REST API.

### Method 1: Using JupyterHealth Exchange Console

1. Navigate to the **Studies** tab
1. Select your **Organization** from the dropdown
1. Find the study in the table
1. Click the **"pencil" icon** (Edit/Update button)
1. In the modal, update the fields:
   - **Name**: Study name
   - **Description**: Study description
   - **Icon URL**: URL to study icon
     - The icon preview updates in real-time as you type
1. Click **"Update"** button
1. Study information is updated

**Note**: You cannot change the organization of an existing study. To manage data sources and scope requests, use the View mode (eye icon) instead of Update mode.

### Method 2: Using REST API

```bash
curl -X PATCH https://your-jhe-instance.com/api/v1/studies/10001 \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "description": "Updated study description with additional details."
  }'
```

**Updatable Fields**:

- `name`
- `description`
- `iconUrl`

Reference: `jupyterhealth-exchange/core/views/study.py`

______________________________________________________________________

## Enroll Patients in Study

Patient enrollment in studies is covered in detail in the [Patient Management](patient-management.md#enroll-patient-in-study) guide, which includes step-by-step instructions for both Console and REST API methods.

______________________________________________________________________

## Monitor Study Progress

After enrolling patients in a study, you can monitor their consent status and data collection progress.

### Method 1: Using JupyterHealth Exchange Console

#### View Patient Consent Status

1. Navigate to the **Patients** tab in the Console
1. Select your **Organization** from the dropdown
1. Select the **Study** from the dropdown
1. Find a patient in the table
1. Click the **"eye" icon** (View button) to open patient details
1. Scroll down to view:
   - **Studies Pending Response**: Studies awaiting patient's consent decision
   - **Studies Responded To**: Studies with consent status
     - Checkmark icon (✓) = Patient consented to this data type
     - X icon = Patient declined this data type

#### View Study Data Collection

1. Navigate to the **Observations** tab in the Console

1. Select your **Organization** from the dropdown at the top

1. Select your **Study** from the dropdown

1. The observations table displays all data collected for the study with these columns:

   - **ID**: Observation ID number
   - **Scope**: Data type (e.g., "Blood pressure", "Heart Rate")
   - **Patient**: Patient name (Last, First format)
   - **Transaction Time**: When the data was uploaded (timestamp)
   - **Data**: Preview of the JSON data (Open mHealth format)

1. Use the pagination controls:

   - Navigate between pages using the page number input
   - Adjust items per page (20, 50, 100, etc.) from the dropdown

1. The **Data** column displays the full OMH (Open mHealth) JSON structure for each observation including:

   - `body`: The actual health measurements (e.g., systolic/diastolic blood pressure values, heart rate)
   - `header`: Metadata including UUID, schema ID, modality, source creation time, and data source reference

**Console Tips:**

- The JSON data is displayed directly in the table for easy viewing
- Each observation shows both the header (metadata) and body (actual measurements)
- Use the Organization and Study dropdowns to switch between different studies
- For programmatic access, filtering, or exporting data, use the REST API (see below)

### Method 2: Using REST API

#### Check Consent Status

```bash
curl https://your-jhe-instance.com/api/v1/patients/10001/consents \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Response:

```json
{
  "patient": {
    "id": 10001,
    "nameFamily": "Smith",
    "nameGiven": "John"
  },
  "consolidatedConsentedScopes": [
    {
      "codingCode": "omh:blood-glucose:4.0",
      "text": "Blood Glucose"
    }
  ],
  "studiesPendingConsent": [],
  "studies": [
    {
      "id": 10001,
      "name": "Diabetes Management Study",
      "scopeConsents": [
        {
          "code": {
            "codingCode": "omh:blood-glucose:4.0"
          },
          "consented": true,
          "consentedTime": "2024-05-01T10:30:00Z"
        },
        {
          "code": {
            "codingCode": "omh:blood-pressure:4.0"
          },
          "consented": true,
          "consentedTime": "2024-05-01T10:30:00Z"
        }
      ]
    }
  ]
}
```

Reference: `jupyterhealth-exchange/core/views/patient.py`

#### Check Data Collection

```bash
curl "https://your-jhe-instance.com/api/v1/observations?organization_id=1&study_id=10001&patient_id=10001" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Response:

```json
{
  "count": 150,
  "results": [
    {
      "id": 1,
      "patientNameDisplay": "Smith, John",
      "codingCode": "omh:blood-glucose:4.0",
      "created": "2024-05-02T08:30:00Z"
    }
  ]
}
```

______________________________________________________________________

## Delete Study

You can delete studies using either the Console or REST API.

### Method 1: Using JupyterHealth Exchange Console

1. Navigate to the **Studies** tab
1. Select your **Organization** from the dropdown
1. Find the study in the table
1. Click the **"trash" icon** (Delete button)
1. A confirmation modal appears: "Are you sure you want to delete this entire record?"
1. Click **"Delete"** button to confirm
1. The study is permanently deleted

**Warning**: This permanently deletes:

- The study
- All patient enrollments
- All consent records
- (Optional) Associated observation data

Consider archiving inactive studies instead of deleting them.

### Method 2: Using REST API

```bash
curl -X DELETE https://your-jhe-instance.com/api/v1/studies/10001 \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

**Warning**: This permanently deletes:

- The study
- All patient enrollments
- All consent records
- (Optional) Associated observation data

Consider archiving inactive studies instead of deleting them.

Reference: `jupyterhealth-exchange/core/views/study.py`

______________________________________________________________________

## Common Issues

### Permission Denied Creating Study

**Error**: 403 Forbidden

**Cause**: User doesn't have `study.manage_for_organization` permission.

**Solution**: Verify your role in the organization:

```bash
curl https://your-jhe-instance.com/api/v1/organizations \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Your role must be `manager` or `member`, not `viewer`.

Reference: `jupyterhealth-exchange/core/permissions.py`

### Cannot Add Patient to Study

**Error**: "Patient and study must share the same organization"

**Cause**: Patient belongs to different organization than study.

**Solution**: Patients can only be added to studies in their organization. Either:

1. Add patient to study's organization first
1. Create study in patient's organization

Reference: `jupyterhealth-exchange/core/views/study.py`

### Scope Request Not Appearing in Consent

**Cause**: Data source doesn't support the requested scope.

**Solution**: Verify data source supports the scope:

```python
python manage.py shell

from core.models import DataSource, CodeableConcept

ds = DataSource.objects.get(id=70001)
scope = CodeableConcept.objects.get(id=1)

# Check if supported
if ds.supported_scopes.filter(id=scope.id).exists():
    print("Supported")
else:
    print("Not supported - create DataSourceSupportedScope link")
```

______________________________________________________________________

## Best Practices

### Study Design

1. **Clear naming**: Use descriptive study names (e.g., "Type 2 Diabetes Glucose Monitoring 2024")
1. **Detailed descriptions**: Include objectives, procedures, expected duration
1. **Minimal data collection**: Only request data types you actually need
1. **Device flexibility**: Support multiple data sources when possible

### Data Collection Planning

1. **Start small**: Begin with one or two data types, expand if needed
1. **Test with pilot**: Enroll a few patients first to validate workflow
1. **Monitor compliance**: Regularly check that patients are uploading data
1. **Communicate clearly**: Ensure patients understand what data is collected and why

### Consent Management

1. **Informed consent**: Provide detailed study information in invitation
1. **Re-consent for changes**: When adding new data types, notify patients
1. **Respect withdrawal**: Make it easy for patients to revoke consent
1. **Document process**: Keep records of when patients consented

### Security

1. **Role-based access**: Only grant manager/member roles to authorized researchers
1. **Audit access**: Regularly review who has access to study data
1. **Data retention**: Define and enforce data retention policies
1. **De-identification**: Consider de-identifying data for analysis

______________________________________________________________________

## Complete Study Setup Example

```bash
# 1. Authenticate
ACCESS_TOKEN="your-access-token"
BASE_URL="https://your-jhe-instance.com"

# 2. Create study
STUDY=$(curl -X POST "$BASE_URL/api/v1/studies" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Hypertension Management Study",
    "description": "Remote blood pressure monitoring for hypertensive patients.",
    "organization": 1
  }')

STUDY_ID=$(echo $STUDY | jq -r '.id')
echo "Created study ID: $STUDY_ID"

# 3. Add scope requests (blood pressure, heart rate)
curl -X POST "$BASE_URL/api/v1/studies/$STUDY_ID/scope_requests" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"scope_code_id": 2}'

curl -X POST "$BASE_URL/api/v1/studies/$STUDY_ID/scope_requests" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"scope_code_id": 4}'

# 4. Add data sources (iHealth, Omron)
curl -X POST "$BASE_URL/api/v1/studies/$STUDY_ID/data_sources" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data_source_id": 70001}'

curl -X POST "$BASE_URL/api/v1/studies/$STUDY_ID/data_sources" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data_source_id": 70005}'

# 5. Enroll patients
curl -X POST "$BASE_URL/api/v1/studies/$STUDY_ID/patients" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"patient_ids": [10001, 10002, 10003]}'

# 6. Generate invitation links
for PATIENT_ID in 10001 10002 10003; do
  curl "$BASE_URL/api/v1/patients/$PATIENT_ID/invitation_link?send_email=true" \
    -H "Authorization: Bearer $ACCESS_TOKEN"
done

echo "Study setup complete!"
```

______________________________________________________________________

## Choosing Between Console and REST API

### When to Use JupyterHealth Exchange Console

✅ **Best for:**

- Creating individual studies interactively
- Configuring data sources and scope requests visually
- Viewing study details and monitoring progress
- Updating study information (name, description, icon)
- Practitioners who prefer web-based interfaces
- Day-to-day study management tasks
- Quick testing and prototyping
- Training and demonstrations

❌ **Not ideal for:**

- Bulk study creation
- Automated study setup workflows
- Integration with external systems
- Scheduled or triggered operations

**Key Features:**

- Create studies from Organizations page
- Visual icon preview with real-time updates
- Add/remove data sources and scopes with dropdowns
- View patient enrollment and consent status
- Delete studies with confirmation
- No programming required

### When to Use REST API

✅ **Best for:**

- Bulk study creation from templates
- Automated study configuration workflows
- Integration with research management systems
- Building custom UIs or applications
- Scheduled or triggered study setup
- Programmatic data collection monitoring
- Continuous integration/deployment pipelines
- Complex multi-step study configurations

❌ **Not ideal for:**

- One-off study creation
- Users unfamiliar with APIs or command line
- Quick interactive study management
- Exploratory study viewing

**Key Features:**

- Batch operations (create multiple studies at once)
- Scriptable workflows (Python, JavaScript, shell)
- Advanced filtering and querying
- Custom business logic
- FHIR R5 API compliance

### Hybrid Approach (Recommended)

Most organizations use both methods for different purposes:

**Typical Workflow:**

1. **Initial Setup** (Console):

   - Create organization via web portal
   - Create first study interactively
   - Configure data sources and scopes
   - Test with a few patients

1. **Bulk Configuration** (REST API):

   - Create multiple studies from templates
   - Configure data collection programmatically
   - Enroll patients in bulk

1. **Daily Operations** (Console):

   - Monitor patient consent status
   - View study progress and data collection
   - Add/remove data sources as needed
   - Generate reports and insights

1. **Analysis and Reporting** (REST API):

   - Export observation data for analysis
   - Generate automated reports
   - Integrate with data warehouses

______________________________________________________________________

## Related Documentation

- [Patient Management](patient-management.md)
- [Fetching and Exporting Data](fetching-exporting-data.md)
- [Using the REST API](using-rest-api.md)
- [RBAC and Governance](../../explanation/rbac-governance.md)
