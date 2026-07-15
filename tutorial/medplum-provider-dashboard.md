---
title: Launch a JupyterHealth dashboard via SMART on FHIR (MedPlum example)
---

In this tutorial you stand up a **provider-facing dashboard app** that a clinician
launches from an EHR. The EHR launch is the only login: the app exchanges the EHR's
`id_token` for a JupyterHealth Exchange token behind the scenes (see
[Provider EHR Launch](../jhe/provider-ehr-launch.md)), joins the launched patient to
their JHE record by MRN, and renders that patient's device data (CGM, wearables) in a
Voilà notebook.

[MedPlum](https://www.medplum.com) plays the EHR here because its hosted sandbox is
free and takes minutes to configure — but nothing below is MedPlum-specific: the same
app launches from any SMART-enabled EHR by changing only registration values (see the
[template's EHR registration guide](https://github.com/jupyterhealth/jupyterhealth-sof-provider-template/blob/main/docs/ehr-registration.md)
for Epic notes, including its stricter scope grammar).

**What you need:**

- A [MedPlum account](https://app.medplum.com) with a project you administer.
- A **JupyterHealth Exchange** instance with the token exchange configured — a local
  dev instance works. You'll need its `SITE_URL`, the "SoF EHR Launch" client
  credentials, and admin access to its settings
  ([how that works](../jhe/provider-ehr-launch.md)).
- A JHE **patient with device data** (the seed data works) and a JHE **practitioner**
  account for yourself.
- Docker.

## 1. Create the app from the template

Generate your app from
[jupyterhealth-sof-provider-template](https://github.com/jupyterhealth/jupyterhealth-sof-provider-template)
("Use this template", or clone it) and follow its
[QUICKSTART](https://github.com/jupyterhealth/jupyterhealth-sof-provider-template/blob/main/docs/QUICKSTART.md)
through the `.env` step. The values that matter for this tutorial:

```
JHE_URL=<your JHE base URL — must equal the JHE instance's SITE_URL exactly>
JHE_CLIENT_ID=<the JHE "SoF EHR Launch" client id>
JHE_CLIENT_SECRET=<its secret>
SMART_CLIENT_ID=<from step 2 below>
SMART_SCOPES=openid fhirUser launch patient/*.read
MRN_IDENTIFIER_SYSTEM=<any URI you choose — you control both sides; e.g. https://example.org/mrn>
```

Then run it: `docker compose up --build` — the app listens on `http://localhost:8888`
and does nothing until an EHR launches it.

## 2. Register the app in MedPlum

In your MedPlum project: **Admin → Project → Clients → New**, and set:

- **Launch URI:** `http://localhost:8888/smart-on-fhir/launch`
- **Redirect URI:** `http://localhost:8888/smart-on-fhir/callback`
- **PKCE enabled; no client secret required** (the app is a public client toward the
  EHR — its only secret is the JHE one, which never goes to MedPlum).

Copy the client's **ID** into `SMART_CLIENT_ID` in your `.env`.

## 3. Configure JHE to trust the launch

On the JHE side ([details](../jhe/provider-ehr-launch.md#configuration)):

- `auth.sof.trusted_issuers` → `["https://api.medplum.com/"]`
- `auth.sof.trusted_audience` → your MedPlum client ID from step 2
- Your JHE practitioner's `identifier` → your **MedPlum Practitioner ID** (open your
  practitioner profile in MedPlum; the id is in the URL — the `fhirUser` claim in the
  launch id_token carries the same value).

## 4. Create the demo patient (the MRN join)

The app finds the JHE patient whose **external identifier equals the EHR patient's MRN
value**. Create a MedPlum `Patient` mirroring a JHE patient with data, e.g. the seeded
CGM patient:

| Field                       | Value                                                   |
| --------------------------- | ------------------------------------------------------- |
| Name / birth date           | copy from the JHE patient (e.g. May Nguyen, 1984-07-11) |
| `Patient.identifier.system` | your `MRN_IDENTIFIER_SYSTEM` from step 1                |
| `Patient.identifier.value`  | the JHE patient's external id (e.g. `1636-69-001`)      |

Name and birth date must match the JHE record — the app verifies the two records are
the same person before showing anything (it fails closed with a lock notice if not).
Identifier *values* must be equal; the identifier *systems* on each side need not agree.

## 5. Launch

In MedPlum, open the patient → **Apps** → click your app. MedPlum redirects through
`/smart-on-fhir/launch`, asks you to authorize, and returns to the callback — at which
point the app exchanges the id_token with JHE and renders the dashboard:
the patient's demographics header and their device data, fetched from JHE **as you**,
under your normal JHE authorization. No JHE login screen ever appears.

On the JHE side you can watch it happen — the log records the exchange:

```
Token exchange: issued JHE token for Practitioner '<your id>' from issuer
https://api.medplum.com/ to client 'sof-ehr-launch'
```

If you get a lock notice or an error page instead, the
[QUICKSTART troubleshooting table](https://github.com/jupyterhealth/jupyterhealth-sof-provider-template/blob/main/docs/QUICKSTART.md#troubleshooting)
maps each symptom to its cause — the failure point (trust config, practitioner mapping,
MRN join) is always identifiable from which message you see.

## Where to go from here

- Put your own analytics in `dashboard.ipynb` (the cell marked
  `ADD YOUR ANALYTICS + VISUALIZATION`); the launch/data plumbing above it doesn't change.
- Try the richer CGM showcase: `cp examples/cgm-dashboard.ipynb dashboard.ipynb`.
- Register with a real EHR and deploy: the template's
  [ehr-registration](https://github.com/jupyterhealth/jupyterhealth-sof-provider-template/blob/main/docs/ehr-registration.md)
  and
  [deployment](https://github.com/jupyterhealth/jupyterhealth-sof-provider-template/blob/main/docs/deployment.md)
  guides cover Epic specifics (scope grammar, iframe/CSP) and hosting.
