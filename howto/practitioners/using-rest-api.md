# Using the JupyterHealth REST API

This guide shows you how to authenticate and interact with the JupyterHealth Exchange REST API. This is foundational reference documentation used by other practitioner guides.

### Introduction

- [Prerequisites](#prerequisites)
- [API Overview](#api-overview)

### Authentication

- [Discover OAuth2 Endpoints](#discover-oauth2-endpoints)
- [Practitioner Authentication (Authorization Code Flow)](#practitioner-authentication-authorization-code-flow)
- [Patient Authentication (Deep Link Flow)](#patient-authentication-deep-link-flow)
- [Making Authenticated Requests](#making-authenticated-requests)

### API Fundamentals

- [Pagination](#pagination)
- [Filtering](#filtering)
- [Error Handling](#error-handling)
- [Data Formats](#data-formats)

### Common Operations

- [Organizations and Studies](#organizations-and-studies)
- [Patients and Consent](#patients-and-consent)
- [Observations](#observations)
- [Data Sources](#data-sources)

### Advanced Topics

- [Rate Limiting and Performance](#rate-limiting-and-performance)
- [Client Libraries](#client-libraries)
- [API Documentation](#api-documentation)

### Reference

- [Related Documentation](#related-documentation)

______________________________________________________________________

## Introduction

### Prerequisites

- JupyterHealth Exchange instance URL
- OAuth2 client credentials (if practitioner)
- Patient authorization code (if patient)
- HTTP client (curl, Postman, or programming language HTTP library)

### API Overview

JupyterHealth Exchange provides two API styles. Other practitioner guides reference this document for authentication and API usage details.

#### Admin REST API (`/api/v1/`)

Traditional REST API for administrative operations:

- User management
- Organization management
- Patient management
- Study management
- Data source management

**When to use**: Task-oriented operations (create, update, delete resources), decoded OMH data

Base URL: `https://your-jhe-instance.com/api/v1/`

#### FHIR API (`/fhir/r5/`)

FHIR-compliant API for health data operations:

- Observation search and retrieval
- Patient search
- Batch observation uploads

**When to use**: Standards-based interoperability, FHIR Bundle operations, base64-encoded OMH data

Base URL: `https://your-jhe-instance.com/fhir/r5/`

Reference: `jupyterhealth-exchange/jhe/urls.py` and `jupyterhealth-exchange/core/urls.py`

______________________________________________________________________

## Authentication

All API endpoints require OAuth2 Bearer token authentication.

Reference: `jupyterhealth-exchange/jhe/settings.py,117`

### Discover OAuth2 Endpoints

```bash
# Get OpenID Connect configuration
curl https://your-jhe-instance.com/o/.well-known/openid-configuration
```

Response includes:

```json
{
  "authorization_endpoint": "https://your-jhe-instance.com/o/authorize/",
  "token_endpoint": "https://your-jhe-instance.com/o/token/",
  "userinfo_endpoint": "https://your-jhe-instance.com/o/userinfo/",
  "grant_types_supported": ["authorization_code", "refresh_token"]
}
```

### Practitioner Authentication (Authorization Code Flow)

#### Step 1: Generate PKCE Parameters

```python
import hashlib
import base64
import secrets

# Generate code verifier
code_verifier = (
    base64.urlsafe_b64encode(secrets.token_bytes(32)).decode("utf-8").rstrip("=")
)

# Generate code challenge
code_challenge = (
    base64.urlsafe_b64encode(hashlib.sha256(code_verifier.encode("utf-8")).digest())
    .decode("utf-8")
    .rstrip("=")
)

print(f"Code Verifier: {code_verifier}")
print(f"Code Challenge: {code_challenge}")
```

#### Step 2: Get Authorization Code

Redirect user to authorization URL:

```
https://your-jhe-instance.com/o/authorize/?response_type=code&client_id=YOUR_CLIENT_ID&redirect_uri=YOUR_REDIRECT_URI&code_challenge=CODE_CHALLENGE&code_challenge_method=S256&scope=read write
```

User logs in and authorizes. They're redirected to:

```
YOUR_REDIRECT_URI?code=AUTHORIZATION_CODE
```

#### Step 3: Exchange Code for Access Token

```bash
curl -X POST https://your-jhe-instance.com/o/token/ \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=authorization_code" \
  -d "code=AUTHORIZATION_CODE" \
  -d "redirect_uri=YOUR_REDIRECT_URI" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "code_verifier=CODE_VERIFIER"
```

Response:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expires_in": 36000,
  "token_type": "Bearer",
  "scope": "read write",
  "refresh_token": "xyzRefreshToken123..."
}
```

Reference: `jupyterhealth-exchange/README.md`

#### Step 4: Refresh Access Token

When access token expires:

```bash
curl -X POST https://your-jhe-instance.com/o/token/ \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=refresh_token" \
  -d "refresh_token=xyzRefreshToken123..." \
  -d "client_id=YOUR_CLIENT_ID"
```

Response:

```json
{
  "access_token": "newAccessToken...",
  "expires_in": 36000,
  "token_type": "Bearer",
  "scope": "read write",
  "refresh_token": "newRefreshToken..."
}
```

### Patient Authentication (Deep Link Flow)

Patients authenticate via invitation links generated by practitioners.

#### Step 1: Generate Invitation Link

Practitioner generates link via API:

```bash
curl https://your-jhe-instance.com/api/v1/patients/10001/invitation_link \
  -H "Authorization: Bearer $PRACTITIONER_TOKEN"
```

Response:

```json
{
  "invitationLink": "https://play.google.com/store/apps/details?id=org.thecommonsproject.android.phr.dev&referrer=cloud_sharing=jhe.yourdomain.com|LhS05iR1rOnpS4JWfP6GeVUIhaRcRh"
}
```

The authorization code is: `LhS05iR1rOnpS4JWfP6GeVUIhaRcRh`

Reference: `jupyterhealth-exchange/core/models.py` and `jupyterhealth-exchange/core/views/patient.py`

#### Step 2: Exchange Code for Access Token

Patient's mobile app exchanges the authorization code:

```bash
curl -X POST https://your-jhe-instance.com/o/token/ \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=authorization_code" \
  -d "code=LhS05iR1rOnpS4JWfP6GeVUIhaRcRh" \
  -d "redirect_uri=https://jhe.yourdomain.com/auth/callback" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "code_verifier=STATIC_VERIFIER_FROM_ENV"
```

**Note**: Patient authentication uses static PKCE verifier configured in `.env`:

```bash
PATIENT_AUTHORIZATION_CODE_VERIFIER="your-static-verifier"
PATIENT_AUTHORIZATION_CODE_CHALLENGE="your-static-challenge"
```

Reference: `jupyterhealth-exchange/jhe/settings.py`

### Making Authenticated Requests

Once you have an access token, include it in the `Authorization` header for all API requests.

#### Example: Get User Profile

```bash
curl https://your-jhe-instance.com/api/v1/users/profile \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Response:

```json
{
  "id": 1,
  "email": "practitioner@example.com",
  "firstName": "Jane",
  "lastName": "Doe",
  "practitioner": {
    "id": 1,
    "jheUserId": 1,
    "organizations": [
      {
        "id": 1,
        "name": "Example Clinic",
        "type": "prov"
      }
    ]
  }
}
```

______________________________________________________________________

## API Fundamentals

### Pagination

Admin API endpoints support pagination:

```bash
# Default: 20 items per page
curl https://your-jhe-instance.com/api/v1/patients?organization_id=1 \
  -H "Authorization: Bearer $TOKEN"
```

Response:

```json
{
  "count": 150,
  "next": "https://your-jhe-instance.com/api/v1/patients?organization_id=1&offset=20",
  "previous": null,
  "results": [...]
}
```

Specify pagination:

```bash
# Get 50 items, starting from offset 100
curl "https://your-jhe-instance.com/api/v1/patients?organization_id=1&limit=50&offset=100" \
  -H "Authorization: Bearer $TOKEN"
```

Reference: `jupyterhealth-exchange/jhe/settings.py`

### Filtering

Filter results by query parameters:

```bash
# Filter patients by organization and study
curl "https://your-jhe-instance.com/api/v1/patients?organization_id=1&study_id=10001" \
  -H "Authorization: Bearer $TOKEN"

# Filter observations by patient and data type
curl "https://your-jhe-instance.com/api/v1/observations?organization_id=1&study_id=10001&patient_id=10001" \
  -H "Authorization: Bearer $TOKEN"
```

### Error Handling

#### Authentication Errors

**401 Unauthorized** - Token expired or invalid

```json
{"detail": "Authentication credentials were not provided."}
```

**Solution**: Refresh access token or re-authenticate.

**403 Forbidden** - Insufficient permissions

```json
{"detail": "You do not have permission to perform this action."}
```

**Solution**: Verify user role and permissions for the resource.

#### Common HTTP Status Codes

- **400 Bad Request**: Invalid request syntax or missing required parameters
- **404 Not Found**: Requested resource does not exist
- **500 Internal Server Error**: Server error - retry with exponential backoff

#### Admin API Error Format

```json
{"detail": "Error message here"}
```

#### FHIR API Error Format (OperationOutcome)

```json
{
  "resourceType": "OperationOutcome",
  "issue": [{
    "severity": "error",
    "code": "invalid",
    "diagnostics": "Detailed error message"
  }]
}
```

#### Retry Strategy

See [Rate Limiting and Performance](#rate-limiting-and-performance) for retry code examples.

### Data Formats

#### CamelCase JSON

The Admin API uses camelCase for JSON keys (not snake_case):

```json
{
  "patientId": 10001,
  "nameFamily": "Smith",
  "nameGiven": "John",
  "birthDate": "1980-01-01",
  "organizationId": 1
}
```

Reference: `jupyterhealth-exchange/jhe/settings.py`

#### Response Formats

**Admin API responses** (simple JSON):

```json
{
  "count": 150,
  "next": "https://...",
  "previous": null,
  "results": [...]
}
```

**FHIR API responses** (FHIR Bundles):

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 150,
  "entry": [
    {"resource": {...}}
  ]
}
```

______________________________________________________________________

## Common Operations

### Organizations and Studies

#### Get Organizations for Practitioner

```bash
curl https://your-jhe-instance.com/api/v1/organizations \
  -H "Authorization: Bearer $PRACTITIONER_TOKEN"
```

Response:

```json
[
  {
    "id": 1,
    "name": "Example Clinic",
    "type": "prov",
    "partOf": null,
    "patients": [],
    "practitioners": [
      {
        "id": 1,
        "email": "practitioner@example.com",
        "role": "manager"
      }
    ]
  }
]
```

#### Get Studies for Organization

```bash
curl "https://your-jhe-instance.com/api/v1/studies?organization_id=1" \
  -H "Authorization: Bearer $PRACTITIONER_TOKEN"
```

Response:

```json
{
  "count": 5,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": 10001,
      "name": "Diabetes Management Study",
      "description": "Monitoring blood glucose levels",
      "organization": 1,
      "iconUrl": "https://example.com/icon.png",
      "patients": [],
      "scopeRequests": [
        {
          "id": 1,
          "codingSystem": "https://w3id.org/openmhealth",
          "codingCode": "omh:blood-glucose:4.0",
          "text": "Blood Glucose"
        }
      ],
      "dataSources": [
        {
          "id": 70001,
          "name": "iHealth",
          "type": "personal_device"
        }
      ]
    }
  ]
}
```

Reference: `jupyterhealth-exchange/core/views/study.py`

### Patients and Consent

#### Get Patients for Study

```bash
curl "https://your-jhe-instance.com/api/v1/patients?organization_id=1&study_id=10001" \
  -H "Authorization: Bearer $PRACTITIONER_TOKEN"
```

Response:

```json
{
  "count": 25,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": 10001,
      "jheUserId": 5,
      "identifier": "MRN-12345",
      "nameFamily": "Smith",
      "nameGiven": "John",
      "birthDate": "1980-01-01",
      "telecomPhone": "555-1234",
      "telecomEmail": "john.smith@example.com",
      "organizationId": 1
    }
  ]
}
```

Reference: `jupyterhealth-exchange/core/models.py`

#### Get Patient Consent Status

```bash
curl https://your-jhe-instance.com/api/v1/patients/10001/consents \
  -H "Authorization: Bearer $PRACTITIONER_TOKEN"
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
      "codingSystem": "https://w3id.org/openmhealth",
      "codingCode": "omh:blood-glucose:4.0",
      "text": "Blood Glucose"
    }
  ],
  "studiesPendingConsent": [],
  "studies": [
    {
      "id": 10001,
      "name": "Diabetes Management Study",
      "organization": {
        "id": 1,
        "name": "Example Clinic"
      },
      "scopeConsents": [
        {
          "code": {
            "id": 1,
            "codingCode": "omh:blood-glucose:4.0",
            "text": "Blood Glucose"
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

### Observations

#### Get Observations for Patient

Admin API:

```bash
curl "https://your-jhe-instance.com/api/v1/observations?organization_id=1&study_id=10001&patient_id=10001" \
  -H "Authorization: Bearer $PRACTITIONER_TOKEN"
```

Response:

```json
{
  "count": 350,
  "next": "...",
  "previous": null,
  "results": [
    {
      "id": 1,
      "subjectPatient": 10001,
      "patientNameDisplay": "Smith, John",
      "codeableConcept": 1,
      "codingSystem": "https://w3id.org/openmhealth",
      "codingCode": "omh:blood-glucose:4.0",
      "dataSource": 70001,
      "status": "final",
      "valueAttachmentData": {...},
      "created": "2024-05-02T14:30:00Z"
    }
  ]
}
```

Reference: `jupyterhealth-exchange/core/views/observation.py`

FHIR API (recommended for interoperability):

```bash
curl "https://your-jhe-instance.com/fhir/r5/Observation?patient._has:Group:member:_id=10001&patient=10001&code=https://w3id.org/openmhealth|omh:blood-glucose:4.0" \
  -H "Authorization: Bearer $PRACTITIONER_TOKEN"
```

Response:

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 350,
  "entry": [
    {
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
        "subject": {"reference": "Patient/10001"},
        "device": {"reference": "Device/70001"},
        "valueAttachment": {
          "contentType": "application/json",
          "data": "eyJoZWFkZXIiOnsidXVpZCI6..."
        }
      }
    }
  ]
}
```

Reference: `jupyterhealth-exchange/core/views/observation.py`

#### Search Patients (FHIR API)

```bash
curl "https://your-jhe-instance.com/fhir/r5/Patient?_has:Group:member:_id=10001" \
  -H "Authorization: Bearer $PRACTITIONER_TOKEN"
```

Response:

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 25,
  "entry": [
    {
      "resource": {
        "resourceType": "Patient",
        "id": "10001",
        "identifier": [{
          "system": "https://your-jhe-instance.com",
          "value": "MRN-12345"
        }],
        "name": [{
          "family": "Smith",
          "given": ["John"]
        }],
        "birthDate": "1980-01-01",
        "telecom": [
          {
            "system": "email",
            "value": "john.smith@example.com"
          },
          {
            "system": "phone",
            "value": "555-1234"
          }
        ]
      }
    }
  ]
}
```

Reference: `jupyterhealth-exchange/core/models.py`

### Data Sources

#### Get Available Data Sources

```bash
curl https://your-jhe-instance.com/api/v1/data_sources \
  -H "Authorization: Bearer $TOKEN"
```

Response:

```json
[
  {
    "id": 70001,
    "name": "iHealth",
    "type": "personal_device",
    "supportedScopes": [
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
]
```

______________________________________________________________________

## Rate Limiting and Performance

### Best Practices

1. **Use pagination** for large datasets:

   ```bash
   # Efficient: paginated requests
   curl "https://your-jhe-instance.com/api/v1/observations?limit=100&offset=0"

   # Inefficient: requesting all records
   curl "https://your-jhe-instance.com/api/v1/observations?limit=10000"
   ```

1. **Cache access tokens** until expiry (10 hours default):

   ```python
   import time


   class TokenManager:
       def __init__(self):
           self.token = None
           self.expires_at = 0

       def get_token(self):
           if time.time() >= self.expires_at:
               self.refresh()
           return self.token

       def refresh(self):
           response = oauth_token_request()
           self.token = response["access_token"]
           self.expires_at = time.time() + response["expires_in"] - 60  # 60s buffer
   ```

1. **Batch observation uploads** using FHIR Bundles:

   ```python
   # Efficient: one request with 100 observations
   bundle = create_fhir_bundle(observations)
   post("/fhir/r5/", bundle)

   # Inefficient: 100 individual requests
   for obs in observations:
       post("/fhir/r5/Observation", obs)
   ```

1. **Filter at the API level**, not in client code:

   ```bash
   # Efficient: server-side filtering
   curl "https://your-jhe-instance.com/api/v1/patients?organization_id=1&study_id=10001"

   # Inefficient: client-side filtering
   curl "https://your-jhe-instance.com/api/v1/patients" | jq '.results[] | select(.organizationId==1)'
   ```

### Connection Pooling

For high-volume applications, use connection pooling:

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

session = requests.Session()

# Retry configuration
retry = Retry(total=3, backoff_factor=0.3, status_forcelist=[500, 502, 503, 504])

# Connection pooling
adapter = HTTPAdapter(pool_connections=10, pool_maxsize=20, max_retries=retry)

session.mount("https://", adapter)
session.mount("http://", adapter)

# Use session for all requests
response = session.get(
    "https://your-jhe-instance.com/api/v1/patients",
    headers={"Authorization": f"Bearer {token}"},
)
```

______________________________________________________________________

## API Documentation

### OpenAPI Schema

Download machine-readable API schema:

```bash
curl https://your-jhe-instance.com/api/v1/schema/ > jhe-api-schema.yaml
```

### Interactive API Explorer

Browse API in Swagger UI:

```
https://your-jhe-instance.com/api/v1/schema/swagger-ui/
```

Or ReDoc:

```
https://your-jhe-instance.com/api/v1/schema/redoc/
```

Reference: `jupyterhealth-exchange/jhe/settings.py`

______________________________________________________________________

## Client Libraries

### Python Example

```python
import requests
import json
from datetime import datetime, timedelta


class JHEClient:
    def __init__(self, base_url, client_id, client_secret=None):
        self.base_url = base_url.rstrip("/")
        self.client_id = client_id
        self.client_secret = client_secret
        self.access_token = None
        self.token_expires = None

    def authenticate(self, username, password):
        """Practitioner authentication"""
        # Step 1: Get authorization code (simplified - normally via browser)
        # Step 2: Exchange for token
        response = requests.post(
            f"{self.base_url}/o/token/",
            data={
                "grant_type": "password",
                "username": username,
                "password": password,
                "client_id": self.client_id,
                "scope": "read write",
            },
        )
        response.raise_for_status()

        data = response.json()
        self.access_token = data["access_token"]
        self.token_expires = datetime.now() + timedelta(seconds=data["expires_in"])

        return self.access_token

    def _get_headers(self):
        if not self.access_token or datetime.now() >= self.token_expires:
            raise Exception("Not authenticated or token expired")

        return {
            "Authorization": f"Bearer {self.access_token}",
            "Content-Type": "application/json",
        }

    def get_patients(self, organization_id, study_id=None):
        """Get patients for organization/study"""
        params = {"organization_id": organization_id}
        if study_id:
            params["study_id"] = study_id

        response = requests.get(
            f"{self.base_url}/api/v1/patients",
            headers=self._get_headers(),
            params=params,
        )
        response.raise_for_status()
        return response.json()

    def get_observations(self, organization_id, study_id, patient_id):
        """Get observations for patient"""
        response = requests.get(
            f"{self.base_url}/api/v1/observations",
            headers=self._get_headers(),
            params={
                "organization_id": organization_id,
                "study_id": study_id,
                "patient_id": patient_id,
            },
        )
        response.raise_for_status()
        return response.json()


# Usage
client = JHEClient("https://jhe.yourdomain.com", "your_client_id")
client.authenticate("practitioner@example.com", "password")

patients = client.get_patients(organization_id=1, study_id=10001)
for patient in patients["results"]:
    print(f"{patient['nameGiven']} {patient['nameFamily']}")
```

### JavaScript/TypeScript Example

```typescript
class JHEClient {
  private baseUrl: string;
  private accessToken: string | null = null;

  constructor(baseUrl: string) {
    this.baseUrl = baseUrl.replace(/\/$/, '');
  }

  async authenticate(authCode: string, codeVerifier: string, clientId: string): Promise<void> {
    const response = await fetch(`${this.baseUrl}/o/token/`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
      },
      body: new URLSearchParams({
        grant_type: 'authorization_code',
        code: authCode,
        redirect_uri: window.location.origin + '/callback',
        client_id: clientId,
        code_verifier: codeVerifier,
      }),
    });

    if (!response.ok) throw new Error('Authentication failed');

    const data = await response.json();
    this.accessToken = data.access_token;
  }

  private getHeaders(): HeadersInit {
    if (!this.accessToken) throw new Error('Not authenticated');

    return {
      'Authorization': `Bearer ${this.accessToken}`,
      'Content-Type': 'application/json',
    };
  }

  async getPatients(organizationId: number, studyId?: number): Promise<any> {
    const params = new URLSearchParams({ organization_id: organizationId.toString() });
    if (studyId) params.append('study_id', studyId.toString());

    const response = await fetch(
      `${this.baseUrl}/api/v1/patients?${params}`,
      { headers: this.getHeaders() }
    );

    if (!response.ok) throw new Error('Failed to fetch patients');
    return response.json();
  }
}

// Usage
const client = new JHEClient('https://jhe.yourdomain.com');
await client.authenticate(authCode, codeVerifier, clientId);

const patients = await client.getPatients(1, 10001);
console.log(`Found ${patients.count} patients`);
```

______________________________________________________________________

## Related Documentation

- [Study Management](study-management.md)
- [Patient Management](patient-management.md)
- [Fetching and Exporting Data](fetching-exporting-data.md)
- [FHIR Interoperability](../../explanation/fhir-interoperability.md)
