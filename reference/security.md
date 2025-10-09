---
title: Security Overview
---

## Introduction

The application focuses on establishing a secure privacy-preserving transmission of patient data from:

```mermaid
flowchart LR
  A[Medical Device API] --> B
  B[CommonHealth Application] --> C
  C[JupyterHealth Exchange] --> D
  D[Partner Data Store]
```

## API Credentials

- CommonHealth Application will authenticate with the Medical Device API using OAuth:
  - Client ID
  - Client Secret
- Each Partner will authenticate with the JupyterHealth Exchange using OAuth:
  - Partner Client ID
  - Partner Client Secret

### API credentials in JupyterHub

JupyterHub is registered as an OAuth Client with the JupyterHealth Exchange.
When a user logs in to JupyterHub, they login via their account in the Exchange.
Upon completion of OAuth, an **access token** is issued to JupyterHub,
which is encrypted and stored in the `auth_state` for the user in the JupyterHub database.

- Access to JupyterHub is granted only to members of the "JupyterHub Users" organization.
- When a user starts their session in JupyterHub,
  their access token for JupyterHealth Exchange is available
  in the environment as `$JHE_TOKEN`.

### SMART-on-FHIR credentials in JupyterHub

Users authenticated with JupyterHub can also gain credentials via SMART App Launch with our demonstration Provider (Medplum).
SMART App Launch is handled as a Public Client, authenticated via Client ID, PKCE and Redirect URL, without a Client Secret.
When a user logs in via SMART App Launch, they have both a SMART token for talking to the healthcare provider and a JHE for talking to the Exchange.
In the future, both should be able to use the same token (or at least exchange the SMART token for an appropriately limited JHE token).

#### Pre-MVP considerations

In the demonstration, all JupyterHealth Exchange users are equivalent permissions-wise.
Users are authenticated as themselves and only have access to patient data associated with studies run by their Organization.
However, currently all JHE users have full permission to manage Organizations, so effectively **all authenticated users have full access to all data in Exchange**.

Following the pre-MVP, appropriate access controls will be [implemented](https://github.com/jupyterhealth/jupyterhealth-exchange/issues/7) in the Exchange.

## Data Flow Diagram

```{image} ../assets/images/CHCS-Architecture.png
---
alt: CHCS Architecture
---
```
