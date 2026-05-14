---
title: Client Flow
---

## Overview

The practitioner uses JHE to create the organization and study, register the patient, and send the patient a secure invitation link.

When the patient opens the link on their device, it launches the JHE Client. The client extracts the JHE host and invitation token from the link, then sends the invitation token to JHE. JHE creates an OAuth 2 grant and returns the grant details along with an authorization code. The client then exchanges the grant and authorization code for an access token.

Using the access token, the JHE Client retrieves the required consent scopes and presents them to the patient. The patient selects their consent preferences, which are then recorded in JHE.

After consent is captured, the JHE Client uploads the patient’s health data as FHIR Observations. JHE validates, associates, and stores the data for the study.

```mermaid
sequenceDiagram
    autonumber

    actor Prac as Practitioner
    participant JHE as JHE
    actor Pat as Patient
    participant Device
    participant Client as JHE Client

    rect rgb(245, 245, 245)
        note over Prac,JHE: Study and patient setup
        Prac->>JHE: Create Organization
        Prac->>JHE: Create Study
        Prac->>JHE: Register Patient<br/>(name, DOB, email/SMS)
        Prac->>JHE: Add Patient to Study
        JHE-->>Pat: Send secure invitation link<br/>(email or SMS)
    end

    rect rgb(235, 245, 255)
        note over Pat,Client: App launch and authentication
        Pat->>Device: Open invitation link
        Device->>Client: Launch JHE Client via configured link
        Client->>Client: Parse hostname and invitation token
        Client->>JHE: Send invitation token
        JHE-->>Client: Return grant with code
        Client->>JHE: Exchange grant with code for access token
        JHE-->>Client: Return access token
    end

    rect rgb(240, 255, 240)
        note over Client,JHE: Consent flow
        Client->>JHE: Request consent requirements<br/>using access token
        JHE-->>Client: Return requested scopes<br/>(BP, heart rate, etc.)
        Client-->>Pat: Display consent options
        Pat->>Client: Select consent choices
        Client->>JHE: Submit consent decisions
        JHE-->>Client: Confirm consent recorded
    end

    rect rgb(255, 250, 235)
        note over Client,JHE: Data upload
        Client->>JHE: Upload data as FHIR Observations
        JHE->>JHE: Validate and store patient data
        JHE-->>Client: Confirm upload accepted
    end
```

## Implementing a Client Step by Step

1. The first thing you need to do is determine the invitation URL for your client and communicate this to the JHE admin to register and configure your client on the target JHE instance(s). The invitation URL is clicked/tapped by patients who receive an E-mail or SMS to join the study. For web based clients, the URL will typically point to a launch page, for app based clients, the URL will follow a domain or pattern that triggers the OS to launch the app. The position of the invitation code (`jhe.tcp.org_0wYuXvhoyRfko9yFYl9inpBiNkHLVBMy` in the example below) in the URL is configured when the client is registered, a couple of different examples:
   - `https://webapp.tcp.org/jhe/launch?invitation=jhe.tcp.org_0wYuXvhoyRfko9yFYl9inpBiNkHLVBMy`
   - `https://app.tcp.org/jhe/invitation/jhe.tcp.org_0wYuXvhoyRfko9yFYl9inpBiNkHLVBMy`
1. Your client then needs to consume the the invitation code (`jhe.tcp.org_0wYuXvhoyRfko9yFYl9inpBiNkHLVBMy` in the example above) and exchange it for an access token that is specific to the patient. See [Auth](auth) for instructions on how to do this.
1. Once you have the access token, it is added to the `Authorization` header with a `Bearer ` prefix to authorize all subsequent requests, eg `'Authorization: Bearer x3VdsqpjayuOQ08G9EnWyAf7LDUor6'`
1. Next, you need to present the Patient with the list of studies they have been invited to and the data scopes requested by the study. This is done is two steps below (see [Admin API](admin-api) for examples):
   1. Get the Patient ID by requesting `GET /api/v1/users/profile`
   1. Use that ID to list consents `GET /api/v1/patients/40001/consents`
1. After the Patient has agreed/consented to the studies and scopes presented in your UI, you need to record this by sending a `POST /api/v1/patients/40001/consents` (see [Admin API](admin-api) for examples). Equivalent `PATCH` and `DELETE` requests can be used to update or revoke consents from your UI.
1. Finally, now that JHE knows the Patient has consented to sharing their data, the Observations can be uploaded over FHIR.
   - Data is expected to be in [Open mHealth](https://www.openmhealth.org/documentation/#/schema-docs/schema-library) JSON format
   - Because the FHIR spec does not include JSON as an Observation value, the closest thing we have is to convert it to binary and then Base 64 encode it and send it in the `valueAttachment` JSON property. See [FHIR API](fhir-api) for an example.
   - Multiple Observations are uploaded at once by posting to the `/fhir/r5/` endpoint. See [FHIR API](fhir-api) for an example.
   - When the Observations hit JHE, the Open mHealth data is validated against the corresponding schema before it is accepted.
