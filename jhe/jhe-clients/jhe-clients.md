---
title: Bundled Clients
---

JHE ships with the clients listed in the table below.

| Data Source | Client | Auth |
| --- | --- | --- |
| **CareX**<br>`omh:blood-pressure:4.0`<br>`omh:heart-rate:2.0`<br><br>**Questionnaire**<br>`QuestionnaireResponse` | **CareX** | Invitation Link |
| **Dexcom Stelo**<br>`omh:blood-glucose:4.0`<br><br>**iHealth**<br>`omh:body-temperature:4.0`<br>`omh:heart-rate:2.0` | **CommonHealth** | Invitation Link |
| **EHR Patient Portal**<br>`*` (all FHIR resources) | **EHR Patient Portal** | Invitation Link |
| – | **JHE Admin** | User Credentials<br/>(Username/Password) |
| **Oura**<br>`ieee:sleep-episode:1.0`<br>`omh:heart-rate:2.0` | **Open Wearables** | Invitation Link |
| – | – | Patient Access<br/>(E-mail one-time code) |



## EHR Patient Portal

This JHE Client allows a Patient to upload their patient chart records into JHE via FHIR by connecting to a supported Patient Portal using the patient-facing SMART on FHIR launch flow.

### Configuration

For this example we will use the Epic MyChart Sandbox.

#### EHR
- Log in to the JHE Admin UI with super user permissions and click on the "EHRs" menu. There should be an EHR Vendor for "Epic Sandbox" - click on the update icon and set the EHR Client ID that is [provided by Epic](https://fhir.epic.com/Documentation?docId=patientfacingfhirapps). Select all the scopes for this example test.

#### Data Source
- Ensure a Data Source is configured with the "All FHIR Resources" scope and label it something appropirate, eg "EHR Patient Portal"

#### JHE Client
- Ensure a Client is configured with the Invitation URL `https://jhe.fly.dev/clients/ehr-patient-portal/?code=CODE`, associate it with th above Data Source and label it something appropirate, eg "EHR Patient Portal".

#### Study
- Create a Study that includes the requested scope "All FHIR Resources" and attach the Data Source and Client from above.

### Test the flow

1. Add a Patient to the Study, view the Patient and then below the "EHR Patient Portal" Client click on the "Generate Invitation Link" button
1. Copy and paste this link into a new Incognito browser window
1. The JHE Web UI will take you through the flow to provde consent for the "All FHIR Resources" scope
1. The JHE Web UI will then redirect you to the Epic MyChart login, enter `username: fhircamila` and `password: epicepic1`
1. The Epic Web UI will then ask you to consent the requested scopes
1. The EPic Web UI will then redirect you back to the JHE Web UI that will import the records into JHE
1. Once complete, return to the JHE Admin UI and click on the FHIR Resource menu
1. Take a ote of the JHE Patient ID from (1) above
1. Choose the corresponding Organization and Study from the dropdowns, select a Resource that you expect from the chart (eg Condition), choose "External" from the Source and then enter the numeric JHE Patient ID from above (eg 40001). You should now see the associated records displayed.


