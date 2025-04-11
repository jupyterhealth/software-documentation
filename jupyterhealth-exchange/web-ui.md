---
title: Web User Interface
---

## Patients & Practitioners

- Any user accessing the Web UI is a data consumer and considered a [Practitioner](https://build.fhir.org/practitioner.html).
- Any user uploading data is considered a [Patient](https://build.fhir.org/patient.html).
- The same OAuth2.0 strategy is used for both Practitioners and Patients, the only difference being that the credentials are provided out-of-band for Patients.

## Organizations

- An [Organization](https://build.fhir.org/organization.html) is a group of Practitioners.
- An Organization is typically hierarchical with sub-Organizations, e.g., Institution, Department, Lab, etc.
- A Patient belongs to a single Organization (TBD: belong to multiple).
- A Practitioner belongs to at least one Organization.

## Studies

- A Study is a [Group](https://build.fhir.org/group.html) of Patients and belongs to a single Organization.
- A Study has one or more Data Sources and one or more Scope Requests.
- When a Patient is added to a Study, they must explicitly consent to sharing the requested Scopes before any data (Observations) can be uploaded or shared.

## Observations

- An [Observation](https://www.hl7.org/fhir/observation.html) is Patient data and belongs to a single Patient.
- An Observation must reference a Patient ID as the *subject* and a Data Source ID as the *device*.
- Personal device data is expected to be in the [Open mHealth](https://www.openmhealth.org/documentation/#/overview/get-started) (JSON) format. However, the system can be easily extended to support any binary data attachments or discrete Observation records.
- Observation data is stored as a *valueAttachment* in Base 64 encoded JSON binary.
- Authorization to view Observations depends on the relationship of Organization, Study, and Consents as described above.

## Data Sources

- A Data Source is anything that produces Observations (typically a device app, e.g., iHealth).
- A Data Source supports one or more Scopes (types) of Observations (e.g., Blood Glucose).
- An Observation references a Data Source ID in the *device* field.

## Use Case Example

1. Sign up as a new user from the web UI.
1. Create a new Organization.
1. Add yourself to the Organization (View Organization > Users+).
1. Create a new Study for the Organization (View Organization > Studies+).
1. Create a new Patient for the Organization using a different email than (1) (Patients > Add Patient).
1. Add Data Sources and Scopes to the Study (View Study > Data Sources+, Scope Requests+).
1. Add the Patient to the Study (Patients > check box > Add Patient(s) to Study).
1. Create an Invitation Link for the Patient (View Patient > Generate Invitation Link).
1. Use the code in the invitation link with the Auth API to swap it for tokens.
1. Upload Observations using the FHIR API.
