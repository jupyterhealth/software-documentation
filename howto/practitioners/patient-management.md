# Patient Management

This guide shows you how to manage patients in JupyterHealth Exchange using either the web-based **Console** (Method 1) or the **REST API** (Method 2).

### Introduction

- [Prerequisites](#prerequisites)
- [Patient Attributes](#understanding-patient-attributes)

### Core Patient Operations

- [Add New Patient](#add-new-patient)
- [Add Existing Patient to Organization](#add-existing-patient-to-organization)
- [Enroll Patient in Study](#enroll-patient-in-study)
- [Update Patient Information](#update-patient-information)
- [Manage Patient Consent](#manage-patient-consent)
- [Remove Patient from Study](#remove-patient-from-study)
- [Remove Patient from Organization](#remove-patient-from-organization)
- [View Patient's Data](#view-patients-data)

### Advanced Topics

- [Bulk Patient Import](#bulk-patient-import)
- [Common Issues](#common-issues)
- [Best Practices](#best-practices)
- [Example: Complete Patient Onboarding via REST API](#example-complete-patient-onboarding-via-rest-api)

### Reference

- [Choosing Between Console and REST API](#choosing-between-jupyterhealth-exchange-console-and-rest-api)
- [Related Documentation](#related-documentation)

______________________________________________________________________

## Introduction

### Prerequisites

**Note**: The JupyterHealth Exchange Console is a web-based Single Page Application (SPA) that provides a user-friendly interface for managing patients, studies, and health data.

**For Console Access:**

- Practitioner account with login credentials
- Member or manager role in an organization
- Patient's demographic information

**For REST API Access:**

- OAuth2 access token
- Member or manager role in an organization
- HTTP client (curl, Python, JavaScript, etc.)

Choose the method that best fits your workflow.

### Understanding Patient Attributes

In JupyterHealth Exchange, patients:

- Have a user account for authentication
- Belong to one or more organizations
- Can be enrolled in multiple studies
- Must consent to data sharing for each study
- Own and control their health data

Reference: `jupyterhealth-exchange/core/models.py`

______________________________________________________________________

## Add New Patient

### Method 1: Using JupyterHealth Exchange Console

The JupyterHealth Exchange Console provides a web-based interface for practitioners to manage patients interactively.

#### Prerequisites

- Practitioner account in JupyterHealth Exchange
- Member or manager role in an organization
- Patient's demographic information
- Access to JupyterHealth Exchange web portal

#### Step 1: Access the Console

1. Navigate to the JupyterHealth Exchange web portal:

   ```
   https://your-jhe-instance.com/portal/
   ```

1. Log in with your practitioner credentials

   - Enter your email address
   - Enter your password
   - You'll be authenticated via OAuth2/OIDC
   - After successful login, you'll be redirected to the Organizations page

#### Step 2: Navigate to Patients

1. In the console navigation, click the **Patients** tab
1. At the top left, select your **Organization** from the dropdown
1. Optionally, filter by **Study** using the second dropdown (or select "All Studies")

You'll see a table of existing patients in the selected organization/study.

#### Step 3: Create New Patient

1. Click the **"Add Patient..."** button (green button, top left area)

1. A modal dialog opens titled "Create Patient"

1. **Option 1: Check if patient already exists**

   - Enter the patient's email in the "Patient E-mail" field
   - Click **"Lookup"** button
   - If patient exists in another organization:
     - Patient details will appear
     - You can add them to your organization (skip to Step 5)
   - If patient doesn't exist:
     - Continue with patient creation below

1. **Create new patient** by filling in the form:

   - **E-mail** (required): `john.smith@example.com`
     - This will be the patient's login email
     - Automatically filled if you used lookup
   - **External Identifier**: Medical record number or other ID (e.g., `MRN-12345`)
   - **Family Name** (required): `Smith`
   - **Given Name** (required): `John`
   - **DOB** (required): `1980-01-01` (YYYY-MM-DD format)
   - **Cell**: `555-1234` (optional phone number)

1. Click **"Create"** button

1. The patient is created and added to the selected organization

1. The modal closes and the patient appears in the patient table

#### Step 4: View Patient Details

After creating the patient, you can view their full details:

1. Find the patient in the table
1. Click the **"eye" icon** (View/Read button) in the leftmost column
1. A modal opens showing:
   - Patient ID and demographic information
   - **Organizations**: List of organizations this patient belongs to
   - **Studies Pending Response**: Studies awaiting patient consent
   - **Studies Responded To**: Studies with consent status (checkmarks for consented)

#### Step 5: Generate Invitation Link

**This is a key step** - the patient needs an invitation link to connect their mobile app and start uploading health data.

1. With the patient details modal open (from Step 4):

1. Scroll down to the **"Generate Invitation Link"** section

1. **Configure email option**:

   - Check **"Send link by email"** to automatically email the link to the patient
   - Or uncheck it to manually copy and send the link yourself

1. Click the **"Generate Invitation Link"** button (green)

1. The invitation link appears in the text area.

1. Send Link to Patient
   Option A: Email sent automatically

   - If "Send link by email" was checked, the patient receives an email with the invitation link
   - Email template includes instructions for installing the CommonHealth app

   Option B: Copy link manually

   - Click **"Copy to Clipboard"** button
   - Send the link to the patient via:
     - Email
     - SMS
     - Patient portal
     - Printed QR code
     - In-person instruction

**Link Components**:

- App package ID: Links to CommonHealth app in app store
- Exchange hostname: Your JupyterHealth Exchange instance
- Authorization code: One-time code for patient authentication
  Example:
  ```
  https://play.google.com/store/apps/details?id=org.thecommonsproject.android.phr.dev&referrer=cloud_sharing=jhe.yourdomain.com|LhS05iR1rOnpS4JWfP6GeVUIhaRcRh
  ```

#### Step 6: Enroll Patient in Study (Optional)

To enroll one or more patients in a study:

1. Navigate to the **Patients** section

1. Select the organization from the "Organization" dropdown at the top

1. Check the checkbox(es) next to the patient(s) you want to enroll

1. Click the **"Add Patient(s) to Study..."** button (blue button)

1. You'll be taken to the **Studies** page where you can select which study to add the patients to

1. Find the target study and click to add the selected patients

1. All selected patients are enrolled in the study

**Notes**:

- You can enroll a single patient or multiple patients at once using this method
- Patient and study must be in the same organization

#### Step 7: Verify Patient Setup

After completing the above steps, verify the patient is properly configured:

1. Click the **"eye" icon** to view patient details
1. Check that:
   - ✅ Patient belongs to correct organization(s)
   - ✅ Patient is enrolled in desired study (appears in "Studies Pending Response")
   - ✅ Invitation link has been generated and sent
1. Patient should receive the invitation email (if email option was selected)
1. Patient can now install the CommonHealth app and authenticate

#### Console Tips

- **Pagination**: Use the page selector and "per page" dropdown to navigate large patient lists
- **Filtering**: Use Organization and Study dropdowns to narrow down the patient list
- **Checkboxes**: Select multiple patients for bulk operations
- **Action Icons**:
  - Eye icon = View patient details
  - Pencil icon = Edit patient information
  - Trash icon = Delete patient from organization
- **Keyboard shortcuts**: Press Enter in page input field to jump to that page

### Method 2: Using REST API

The REST API enables programmatic patient management, ideal for bulk imports and automated workflows.

#### Prerequisites

- OAuth2 access token (practitioner authentication)
- Member or manager role in an organization
- HTTP client (curl, Postman, or programming language)

#### Step 1: Authenticate

Obtain an access token as a practitioner. See [Using the REST API](using-rest-api.md#authentication) for the full authentication flow.

```bash
ACCESS_TOKEN="your-access-token-here"
```

#### Step 2: Create Patient via API

```bash
curl -X POST https://your-jhe-instance.com/api/v1/patients \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "telecomEmail": "john.smith@example.com",
    "nameFamily": "Smith",
    "nameGiven": "John",
    "birthDate": "1980-01-01",
    "telecomPhone": "555-1234",
    "organizationId": 1
  }'
```

**Fields**:

- `telecomEmail` (required): Patient's email address (used for authentication)
- `nameFamily` (required): Last name
- `nameGiven` (required): First name
- `birthDate` (required): Date of birth in YYYY-MM-DD format
- `telecomPhone` (optional): Phone number
- `organizationId` (required): Your organization ID

Response:

```json
{
  "id": 10001,
  "jheUserId": 5,
  "identifier": null,
  "nameFamily": "Smith",
  "nameGiven": "John",
  "birthDate": "1980-01-01",
  "telecomPhone": "555-1234",
  "telecomEmail": "john.smith@example.com",
  "organizationId": 1
}
```

Note the patient ID (e.g., `10001`) for subsequent operations.

Reference: `jupyterhealth-exchange/core/views/patient.py`

#### Step 3: Generate Invitation Link

Create an invitation link for the patient to connect their mobile app:

```bash
curl https://your-jhe-instance.com/api/v1/patients/10001/invitation_link \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Response:

```json
{
  "invitationLink": "https://play.google.com/store/apps/details?id=org.thecommonsproject.android.phr.dev&referrer=cloud_sharing=jhe.yourdomain.com|LhS05iR1rOnpS4JWfP6GeVUIhaRcRh"
}
```

**Link Components**:

- App package ID: `org.thecommonsproject.android.phr.dev`
- Exchange hostname: `jhe.yourdomain.com`
- Authorization code: `LhS05iR1rOnpS4JWfP6GeVUIhaRcRh`

Reference: `jupyterhealth-exchange/core/models.py` and `jupyterhealth-exchange/core/views/patient.py`

#### Step 4: Send Invitation to Patient

##### Option 1: Automatic Email

```bash
curl "https://your-jhe-instance.com/api/v1/patients/10001/invitation_link?send_email=true" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

This sends an email to the patient's registered email address with the invitation link.

Reference: `jupyterhealth-exchange/core/views/patient.py`

##### Option 2: Manual Distribution

Send the invitation link via:

- Email
- SMS
- Patient portal
- Printed QR code
- In-person instruction

**Example email template**:

```
Subject: Join [Study Name] - Connect Your Health Data

Dear John,

You've been invited to participate in our health study. To get started:

1. Click this link on your mobile device: [INVITATION_LINK]
2. Install the CommonHealth app (if not already installed)
3. Follow the instructions to connect your health devices
4. Review and provide consent for data sharing

If you have questions, contact us at research@example.com.

Thank you,
[Your Organization Name]
```

______________________________________________________________________

## Add Existing Patient to Organization

If a patient already exists in the Exchange but not in your organization, you can add them using either method.

### Method 1: Using JupyterHealth Exchange Console

1. Navigate to **Patients** tab
1. Click **"Add Patient..."** button
1. In the modal, enter the patient's email in the "Patient E-mail" field
1. Click **"Lookup"** button
1. If the patient exists in another organization:
   - Patient details will appear
   - An alert shows: "This Patient already exists. Click the Update button below to add this Patient to the Organization"
1. Click **"Update"** button
1. Patient is added to your organization

### Method 2: Using REST API

#### Step 1: Search for Patient

```bash
curl "https://your-jhe-instance.com/api/v1/patients/global_lookup?email=john.smith@example.com" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Response:

```json
[
  {
    "id": 10001,
    "nameFamily": "Smith",
    "nameGiven": "John",
    "birthDate": "1980-01-01",
    "telecomEmail": "john.smith@example.com"
  }
]
```

**Note**: This searches across all organizations in the Exchange.

Reference: `jupyterhealth-exchange/core/views/patient.py`

#### Step 2: Add to Your Organization

```bash
curl -X PATCH "https://your-jhe-instance.com/api/v1/patients/10001/global_add_organization?organization_id=1" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Response:

```json
{
  "id": 10001,
  "nameFamily": "Smith",
  "nameGiven": "John",
  "organizationId": 1
}
```

Reference: `jupyterhealth-exchange/core/views/patient.py`

______________________________________________________________________

## Enroll Patient in Study

You can enroll patients in studies using either the Console or REST API.

### Method 1: Using JupyterHealth Exchange Console

1. Navigate to **Patients** tab
1. Select your **Organization** from the dropdown
1. Select the **Study** from the dropdown (not "All Studies")
1. Check the checkbox(es) next to the patient(s) you want to enroll
1. Click **"Add Patient(s) to Study..."** button (blue button)
1. Confirm the enrollment
1. Patient(s) are enrolled in the selected study

**Bulk enrollment**: You can select multiple patients and enroll them all at once.

**Note**: Patient and study must be in the same organization.

### Method 2: Using REST API

#### Step 1: List Available Studies

```bash
curl "https://your-jhe-instance.com/api/v1/studies?organization_id=1" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Response:

```json
{
  "results": [
    {
      "id": 10001,
      "name": "Diabetes Management Study",
      "description": "Monitoring blood glucose levels",
      "scopeRequests": [
        {
          "codingCode": "omh:blood-glucose:4.0",
          "text": "Blood Glucose"
        }
      ]
    }
  ]
}
```

#### Step 2: Enroll Patient

```bash
curl -X POST https://your-jhe-instance.com/api/v1/studies/10001/patients \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "patient_ids": [10001]
  }'
```

**Note**: Patient and study must be in the same organization.

Reference: `jupyterhealth-exchange/core/views/study.py`

#### Step 3: Verify Enrollment

```bash
curl "https://your-jhe-instance.com/api/v1/patients?organization_id=1&study_id=10001" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Response should include the newly enrolled patient.

______________________________________________________________________

## Manage Patient Consent

Patient consent can be viewed through the Console. To grant or revoke consent on behalf of a patient, use the REST API.

### Method 1: View Patient Consent Using JupyterHealth Exchange Console

1. Navigate to **Patients** tab
1. Find the patient and click the **"eye" icon** (View button)
1. In the patient details modal, scroll down to view:
   - **Studies Pending Response**: Studies awaiting patient's consent decision
     - Shows which data types (scopes) are pending
   - **Studies Responded To**: Studies with consent decisions
     - Checkmark icon (✓) = Patient consented to this data type
     - X icon = Patient declined this data type

**Note:** Consent status in the Console is **read-only**. Patients typically grant consent via their mobile app. To grant or revoke consent on behalf of a patient (with appropriate authorization), use the REST API.

### Method 2: View Consent Status via REST API

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
    "nameGiven": "John",
    "birthDate": "1980-01-01"
  },
  "consolidatedConsentedScopes": [
    {
      "id": 1,
      "codingCode": "omh:blood-glucose:4.0",
      "text": "Blood Glucose"
    }
  ],
  "studiesPendingConsent": [
    {
      "id": 10002,
      "name": "Cardiac Health Study",
      "pendingScopeConsents": [
        {
          "code": {
            "codingCode": "omh:blood-pressure:4.0",
            "text": "Blood Pressure"
          },
          "consented": null,
          "consentedTime": null
        }
      ]
    }
  ],
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
        }
      ]
    }
  ]
}
```

**Key Fields**:

- `consolidatedConsentedScopes`: All data types patient has consented to (across all studies)
- `studiesPendingConsent`: Studies awaiting patient's consent decision
- `studies`: Studies patient has consented to or declined

Reference: `jupyterhealth-exchange/core/views/patient.py`

### Grant Consent (Practitioner on Behalf of Patient)

Practitioners with **member** or **manager** roles can grant consent on behalf of patients using the REST API.

**Important Notes:**

- Patients typically grant consent themselves via their mobile app
- Practitioners should only grant consent with explicit patient authorization
- This functionality is **not yet available in the Console UI** - must use REST API

**Prerequisites:**

- OAuth2 access token with practitioner authentication
- Member or manager role in the patient's organization (viewer role cannot grant consent)
- Patient must be enrolled in the study

```bash
curl -X POST https://your-jhe-instance.com/api/v1/patients/10001/consents \
  -H "Authorization: Bearer $PRACTITIONER_TOKEN" \
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

Reference: `jupyterhealth-exchange/core/views/patient.py` and authorization check at lines 198-200

### Revoke Consent (Practitioner on Behalf of Patient)

Practitioners with **member** or **manager** roles can revoke consent on behalf of patients using the REST API.

**Important Notes:**

- Patients typically revoke consent themselves via their mobile app
- Practitioners should only revoke consent with explicit patient authorization
- This functionality is **not yet available in the Console UI** - must use REST API

To revoke consent, set `consented: false`:

```bash
curl -X POST https://your-jhe-instance.com/api/v1/patients/10001/consents \
  -H "Authorization: Bearer $PRACTITIONER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "studyScopeConsents": [{
      "studyId": 10001,
      "scopeConsents": [{
        "codingSystem": "https://w3id.org/openmhealth",
        "codingCode": "omh:blood-glucose:4.0",
        "consented": false
      }]
    }]
  }'
```

**Effect**: Patient's existing data remains in the Exchange, but no new data of this type will be collected from the patient's devices.

______________________________________________________________________

## Update Patient Information

You can update patient demographics using either the Console or REST API.

### Method 1: Using JupyterHealth Exchange Console

1. Navigate to **Patients** tab
1. Find the patient in the table
1. Click the **"pencil" icon** (Edit/Update button)
1. In the modal, update the demographic fields:
   - External Identifier
   - Family Name
   - Given Name
   - DOB
   - Cell phone
1. Click **"Update"** button
1. Patient information is updated

**Note**: Email address is read-only in the Console and cannot be changed through the UI. Changing email requires REST API access.

### Method 2: Using REST API

```bash
curl -X PATCH https://your-jhe-instance.com/api/v1/patients/10001 \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "telecomPhone": "555-5678",
    "nameGiven": "Jonathan"
  }'
```

**Updatable Fields**:

- `nameFamily`
- `nameGiven`
- `birthDate`
- `telecomPhone`
- `telecomEmail`
- `identifier`

**Note**: Email address is used for authentication. Changing it requires patient to re-authenticate.

______________________________________________________________________

## Remove Patient from Study

You can remove patients from studies using either the Console or REST API.

### Method 1: Using JupyterHealth Exchange Console

1. Navigate to **Patients** tab
1. Select the **Organization** from the dropdown
1. Select the **Study** from the dropdown (the study you want to remove patients from)
1. Check the checkbox(es) next to the patient(s) you want to remove
1. Click **"Remove Patient(s) from Study"** button (orange/warning button)
1. Confirm the removal
1. Patient(s) are removed from the study

**Bulk removal**: You can select multiple patients and remove them all at once.

### Method 2: Using REST API

```bash
curl -X DELETE https://your-jhe-instance.com/api/v1/studies/10001/patients \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "patient_ids": [10001]
  }'
```

**Effect**:

- Removes patient from study
- Deletes consent records for this study
- Existing observation data remains (not deleted)

Reference: `jupyterhealth-exchange/core/views/study.py`

______________________________________________________________________

## Remove Patient from Organization

You can remove patients from organizations using either the Console or REST API.

### Method 1: Using JupyterHealth Exchange Console

1. Navigate to **Patients** tab
1. Select the **Organization** from the dropdown
1. Ensure "All Studies" is selected (to see all patients in organization)
1. Find the patient in the table
1. Click the **"trash" icon** (Delete button)
1. A confirmation modal appears: "Are you sure you want to delete this entire record?"
1. Click **"Delete"** button to confirm
1. Patient is removed from the selected organization

**Warning**:

- This removes the patient from the selected organization
- Patient is also removed from all studies in that organization
- If the patient has no other organizations, this will delete all observations and the patient record entirely

### Method 2: Using REST API

```bash
curl -X DELETE "https://your-jhe-instance.com/api/v1/patients/10001?organization_id=1" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

**Effect**:

- Removes patient from organization
- Removes from all studies in that organization
- Deletes all consent records for organization's studies
- If patient has no other organizations: deletes all observations and patient record

Reference: `jupyterhealth-exchange/core/views/patient.py`

______________________________________________________________________

## View Patient's Data

You can view patient data using either the Console or REST API.

### Method 1: Using JupyterHealth Exchange Console

1. Navigate to **Observations** tab in the Console
1. Select the **Organization** from the dropdown
1. Select the **Study** from the dropdown (optional)
1. Use filters to narrow the data:
   - **Patient**: Select specific patient or view all
   - **Data Type**: Filter by codeable concept (e.g., blood-glucose, steps)
   - **Date Range**: Filter by time period
1. The observations table displays:
   - Observation ID
   - Patient name
   - Data type (codeable concept)
   - Data source (device/wearable)
   - Status
   - Created date/time
1. Click on an observation row to view full details including OMH data payload
1. Use pagination controls to navigate large datasets

**Tips**:

- Export data via REST API for analysis (see below)
- Use date filters to focus on specific time periods
- Filter by data source to see data from specific devices

### Method 2: Using REST API

#### List Observations

```bash
curl "https://your-jhe-instance.com/api/v1/observations?organization_id=1&study_id=10001&patient_id=10001" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Response:

```json
{
  "count": 250,
  "next": "...",
  "previous": null,
  "results": [
    {
      "id": 1,
      "subjectPatient": 10001,
      "patientNameDisplay": "Smith, John",
      "codeableConcept": 1,
      "codingCode": "omh:blood-glucose:4.0",
      "dataSource": 70001,
      "status": "final",
      "created": "2024-05-02T08:30:00Z"
    }
  ]
}
```

Reference: `jupyterhealth-exchange/core/models.py`

#### Export Patient Data (FHIR)

```bash
curl "https://your-jhe-instance.com/fhir/r5/Observation?patient=10001&patient._has:Group:member:_id=10001" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Returns FHIR Bundle with all authorized observations for the patient.

Reference: See "[Fetching and Exporting Data](fetching-exporting-data.md)" guide for details.

______________________________________________________________________

## Bulk Patient Import

For importing multiple patients at once, use the REST API with a script:

```python
import requests
import csv

BASE_URL = "https://your-jhe-instance.com"
ACCESS_TOKEN = "your-access-token"
ORGANIZATION_ID = 1


def create_patient(email, family_name, given_name, birth_date, phone=None):
    response = requests.post(
        f"{BASE_URL}/api/v1/patients",
        headers={
            "Authorization": f"Bearer {ACCESS_TOKEN}",
            "Content-Type": "application/json",
        },
        json={
            "telecomEmail": email,
            "nameFamily": family_name,
            "nameGiven": given_name,
            "birthDate": birth_date,
            "telecomPhone": phone,
            "organizationId": ORGANIZATION_ID,
        },
    )

    if response.status_code == 201:
        patient = response.json()
        print(f"Created patient {patient['id']}: {given_name} {family_name}")
        return patient
    else:
        print(f"Error creating {given_name} {family_name}: {response.text}")
        return None


def generate_invitation(patient_id):
    response = requests.get(
        f"{BASE_URL}/api/v1/patients/{patient_id}/invitation_link?send_email=true",
        headers={"Authorization": f"Bearer {ACCESS_TOKEN}"},
    )

    if response.status_code == 200:
        data = response.json()
        print(f"  Invitation sent: {data['invitationLink']}")
    else:
        print(f"  Error generating invitation: {response.text}")


# Read from CSV
with open("patients.csv", "r") as f:
    reader = csv.DictReader(f)
    for row in reader:
        patient = create_patient(
            email=row["email"],
            family_name=row["last_name"],
            given_name=row["first_name"],
            birth_date=row["dob"],
            phone=row.get("phone"),
        )

        if patient:
            generate_invitation(patient["id"])

print("Bulk import complete!")
```

**CSV Format** (`patients.csv`):

```csv
email,last_name,first_name,dob,phone
john.smith@example.com,Smith,John,1980-01-01,555-1234
jane.doe@example.com,Doe,Jane,1975-05-15,555-5678
```

______________________________________________________________________

## Common Issues

### Patient Already Exists

**Error**: "User with this email already exists"

**Cause**: A patient with this email already has an account.

**Solution**: Use global lookup to find the patient and add them to your organization:

```bash
# Find patient
curl "https://your-jhe-instance.com/api/v1/patients/global_lookup?email=john.smith@example.com" \
  -H "Authorization: Bearer $ACCESS_TOKEN"

# Add to organization
curl -X PATCH "https://your-jhe-instance.com/api/v1/patients/10001/global_add_organization?organization_id=1" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Reference: `jupyterhealth-exchange/core/views/patient.py`

### Cannot Enroll Patient in Study

**Error**: "Patient and study must share the same organization"

**Cause**: Patient is in organization A but study is in organization B.

**Solution**: Ensure both patient and study are in the same organization:

```bash
# Check patient's organization
curl https://your-jhe-instance.com/api/v1/patients/10001 \
  -H "Authorization: Bearer $ACCESS_TOKEN"

# Check study's organization
curl "https://your-jhe-instance.com/api/v1/studies?organization_id=1" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Reference: `jupyterhealth-exchange/core/views/study.py`

### Invitation Link Doesn't Work

**Issue**: Patient clicks link but app doesn't open or connection fails.

**Common Causes**:

1. **Link expired**: Authorization codes expire after 2 weeks

   - Solution: Generate new invitation link

1. **Wrong app package**: Link targets wrong environment (dev vs prod)

   - Solution: Verify `CH_INVITATION_LINK_PREFIX` in `.env`

1. **App not installed**: Patient doesn't have CommonHealth app

   - Solution: Link should redirect to app store first

1. **Network issue**: Can't reach Exchange server

   - Solution: Verify Exchange URL is accessible

Reference: `jupyterhealth-exchange/jhe/settings.py` and `jupyterhealth-exchange/core/models.py`

### Permission Denied

**Error**: 403 Forbidden when creating or viewing patient

**Cause**: User doesn't have `patient.manage_for_organization` permission.

**Solution**: Verify your role in the organization:

```bash
curl https://your-jhe-instance.com/api/v1/organizations \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Your role must be `manager` or `member`, not `viewer`.

Reference: `jupyterhealth-exchange/core/permissions.py` and view permission check at `jupyterhealth-exchange/core/views/patient.py`

______________________________________________________________________

## Best Practices

### Patient Onboarding

1. **Verify information**: Double-check email address and demographics before creating patient
1. **Immediate invitation**: Generate and send invitation link right after creating patient
1. **Follow-up**: Contact patients who don't connect within 48 hours
1. **Test first**: Create a test patient account to verify the onboarding flow

### Consent Management

1. **Clear communication**: Explain what data will be collected and why
1. **Study-specific consent**: Patients consent per study, not globally
1. **Allow withdrawal**: Make it easy for patients to revoke consent
1. **Document consent**: Keep records of when patients consented

### Data Privacy

1. **Minimum necessary**: Only collect data types required for your study
1. **Role-based access**: Ensure only authorized practitioners can view patient data
1. **Audit access**: Regularly review who has accessed patient records
1. **De-identify for analysis**: Use de-identified datasets when possible

### Communication

1. **Multi-channel**: Provide invitation links via multiple channels (email, SMS, portal)
1. **Technical support**: Have a help desk for patients with connection issues
1. **Status updates**: Regularly check which patients are actively uploading data
1. **Engagement**: Reach out to patients who stop uploading data

______________________________________________________________________

## Example: Complete Patient Onboarding via REST API

```bash
# Set variables
BASE_URL="https://your-jhe-instance.com"
ACCESS_TOKEN="your-access-token"
ORG_ID=1
STUDY_ID=10001

# 1. Create patient
PATIENT=$(curl -X POST "$BASE_URL/api/v1/patients" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "telecomEmail": "john.smith@example.com",
    "nameFamily": "Smith",
    "nameGiven": "John",
    "birthDate": "1980-01-01",
    "telecomPhone": "555-1234",
    "organizationId": '$ORG_ID'
  }')

PATIENT_ID=$(echo $PATIENT | jq -r '.id')
echo "Created patient ID: $PATIENT_ID"

# 2. Enroll in study
curl -X POST "$BASE_URL/api/v1/studies/$STUDY_ID/patients" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"patient_ids": ['$PATIENT_ID']}'

echo "Enrolled in study $STUDY_ID"

# 3. Generate and send invitation
INVITATION=$(curl "$BASE_URL/api/v1/patients/$PATIENT_ID/invitation_link?send_email=true" \
  -H "Authorization: Bearer $ACCESS_TOKEN")

echo "Invitation sent:"
echo $INVITATION | jq -r '.invitationLink'

# 4. Verify patient status
curl "$BASE_URL/api/v1/patients/$PATIENT_ID/consents" \
  -H "Authorization: Bearer $ACCESS_TOKEN" | jq

echo "Patient onboarding complete!"
```

______________________________________________________________________

## Choosing Between JupyterHealth Exchange Console and REST API

### When to Use JupyterHealth Exchange Console

✅ **Best for:**

- Adding individual patients interactively
- Practitioners who prefer web-based interfaces
- Viewing and managing patient data visually
- Generating invitation links with email integration
- Monitoring patient consent status
- Quick testing and prototyping
- Training and demonstrations
- Day-to-day patient management tasks

❌ **Not ideal for:**

- Bulk patient imports (>20 patients)
- Automated workflows
- Integration with external systems (EHR, etc.)
- Scheduled or triggered operations
- Building custom applications

**Key Features:**

- Full patient lifecycle management (create, view, update, delete)
- One-click invitation link generation with email
- Visual consent status tracking
- Study enrollment with multi-select
- Real-time observation viewing
- Organization and study filtering
- No programming required

### When to Use REST API

✅ **Best for:**

- Bulk patient imports from CSV or database
- Automated patient enrollment workflows
- Integration with EHR systems or other applications
- Building custom UIs or mobile apps
- Scheduled or triggered patient creation
- Data export and analysis (CSV, DataFrame)
- Programmatic consent management
- Continuous integration/deployment pipelines

❌ **Not ideal for:**

- One-off patient additions
- Users unfamiliar with APIs or command line
- Quick interactive patient management
- Exploratory data viewing

**Key Features:**

- Batch operations (create 100s of patients at once)
- Scriptable workflows (Python, JavaScript, shell)
- Webhook integration
- Custom business logic
- Advanced filtering and querying
- FHIR R5 API compliance

### Hybrid Approach (Recommended)

Most organizations use both methods for different purposes:

**Typical Workflow:**

1. **Initial Setup** (Console):

   - Create organizations via web portal
   - Create studies with scope requests
   - Configure data sources
   - Add initial test patients

1. **Bulk Import** (REST API):

   - Import existing patients from EHR system
   - Enroll patients in studies programmatically
   - Generate and send invitation links in batch

1. **Daily Operations** (Console):

   - Add new patients one-by-one as they enroll
   - View consent status and follow up with patients
   - Monitor data collection in real-time
   - Generate invitation links for individual patients
   - Troubleshoot patient issues

1. **Analysis and Reporting** (REST API):

   - Export observation data for analysis
   - Generate reports via scripts
   - Integrate with data warehouses
   - Automated quality checks

______________________________________________________________________

## Related Documentation

- [Study Management](study-management.md)
- [Fetching and Exporting Data](fetching-exporting-data.md)
- [Using the REST API](using-rest-api.md)
- [Consent Management](../../explanation/consent-management.md)
