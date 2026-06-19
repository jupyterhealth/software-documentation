---
title: Patient Identifiers
---

### Patient Identifiers

A patient in JupyterHealth Exchange (JHE) is referred to by **three different identifiers**. They are easy to confuse because all three are sometimes loosely called an "ID", but each means something different and is used in different places. Mixing them up is a common source of integration bugs, so always be explicit about which one you mean.

| Identifier                | What it is                                                                          | How many per patient | Where you see it                                                                      |
| ------------------------- | ----------------------------------------------------------------------------------- | -------------------- | ------------------------------------------------------------------------------------- |
| **Patient ID**            | The primary key of the `Patient` record in JHE                                      | Exactly one          | FHIR `Patient/{id}` references, REST `/api/v1/patients/{id}`, the admin Patients list |
| **User ID** (`jheUserId`) | The primary key of the `JheUser` login account linked to the patient                | Zero or one          | The Observations displays (jhe-admin and Django admin), the `users/profile` response  |
| **External ID**           | An identifier the patient is known by in an outside system (for example an EHR MRN) | Zero or more         | FHIR `Patient.identifier`, the admin Patients list, identifier-based search           |

#### Patient ID

The Patient ID is the database primary key of the `Patient` record. It is JHE's canonical handle for a patient and is what most of the system uses internally.

- It is the `{id}` in REST calls such as `GET /api/v1/patients/{id}`.
- In FHIR it is the logical id of the Patient resource, so an Observation points at its patient with `subject.reference = "Patient/{Patient ID}"`.
- Every patient has exactly one Patient ID.

#### User ID (`jheUserId`)

The User ID is the primary key of the `JheUser` account that the patient logs in with. It is **not** the same number as the Patient ID: the Patient record is the clinical/data record, while the JheUser is the login account behind it.

- A patient created without a login account (for example pre-registered before they are invited) has **no** User ID until an account is associated.
- It is exposed as `jheUserId` on the patient in the `users/profile` response and is shown alongside observations in both the jhe-admin UI and the Django admin so an operator can tell which account produced the data.
- Use the User ID when you need to talk about the *account*; use the Patient ID when you need to talk about the *patient data record*.

#### External ID

An External ID is an identifier the patient is already known by in another system, such as a medical record number (MRN) from an EHR. JHE stores these as `PatientIdentifier` rows, each a `system` (a URI naming the issuing system) plus a `value`.

- A patient can have **several** External IDs, one per external system.
- They are surfaced in FHIR as entries in `Patient.identifier` and can be used to look a patient up by their external value instead of their JHE Patient ID.
- The combination of `system` and `value` is unique, so the same external identifier cannot be attached to two patients.

#### Which one should I use?

- Linking JHE data together (observations, consents, references): **Patient ID**.
- Talking about the login account or matching a signed-in user to their data: **User ID**.
- Reconciling a patient with a record in an external/EHR system: **External ID**.
