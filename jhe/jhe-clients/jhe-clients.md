---
title: Bundled Clients
---

JHE ships with the clients listed in the table below.

| Data Source                                                                                                          | Client                 | Auth                                      |
| -------------------------------------------------------------------------------------------------------------------- | ---------------------- | ----------------------------------------- |
| **CareX**<br>`omh:blood-pressure:4.0`<br>`omh:heart-rate:2.0`<br><br>**Questionnaire**<br>`QuestionnaireResponse`    | **CareX**              | Invitation Link                           |
| **Dexcom Stelo**<br>`omh:blood-glucose:4.0`<br><br>**iHealth**<br>`omh:body-temperature:4.0`<br>`omh:heart-rate:2.0` | **CommonHealth**       | Invitation Link                           |
| **EHR Patient Portal**<br>`*` (all FHIR resources)                                                                   | **EHR Patient Portal** | Invitation Link                           |
| –                                                                                                                    | **JHE Admin**          | User Credentials<br/>(Username/Password)  |
| **Oura**<br>`ieee:sleep-episode:1.0`<br>`omh:heart-rate:2.0`                                                         | **Open Wearables**     | Invitation Link                           |
| –                                                                                                                    | –                      | Patient Access<br/>(E-mail one-time code) |

## EHR Patient Portal

This JHE Client allows a Patient to upload their patient chart records into JHE via FHIR by connecting to a supported Patient Portal using the patient-facing SMART on FHIR launch flow.

### Overview

#### Data model

- An **EhrVendor** is an EHR product, eg "Epic Production" or "Epic Sandbox". It holds the OAuth Client ID and the supported scopes.
- An **EhrBrand** is a health system under a vendor. It holds the FHIR base URL (the SMART `iss`), which is unique per brand.
- An **EhrBrandLocation** is a facility under a brand. This is what the Patient searches for and picks. All locations of a brand share the brand's FHIR base URL.

#### Seeding and the Admin UI

- `manage.py seed` imports a small sample list of brands (`core/data/ehr_brands.sample.json`) and fills in the Epic Sandbox Client ID and scopes if they are blank.
- In the JHE Admin UI, a superuser can open the "EHRs" menu to see the vendors and edit their Client ID and scopes.
- Click the view icon on a vendor to see its brands, with each brand's locations listed beneath it. Hover over a brand for its URL and over a location for its address.
- Brands and locations are read-only in the Admin UI. They are loaded by the import command below.

#### Import the full Epic production list

The full list is about 95 MB, so it is not bundled in the JHE repository.

1. Open the Epic [Endpoints](https://open.epic.com/MyApps/Endpoints) page and download the "User-access Brands Bundle" JSON file
1. Save it somewhere outside version control, eg `local/epic_brands.json` (the `local` folder is git-ignored)
1. Run `uv run python manage.py import_ehr_brands --file local/epic_brands.json`

- The import is safe to repeat. Brands are matched on their FHIR base URL and locations on brand, name and address.
- Brands with "sandbox" in the name go under the "Epic Sandbox" vendor. All others go under "Epic Production".
- "Epic Production" has no Client ID. Set it in the "EHRs" menu once Epic has registered your app.

#### Patient external ID

- When a Patient signs in to the EHR, the EHR returns its own ID for that Patient.
- JHE saves it as an [External ID](../patient-identifiers.md) on the JHE Patient. The system is the brand's FHIR base URL and the value is the EHR's Patient ID.
- It is shown under "External Identifiers" when you update the Patient in the Admin UI.
- An External ID can belong to only one JHE Patient. Connecting the same Patient again is fine.

#### FHIR Sources and repeat syncs

- Each connection is recorded as a [FhirSource](../fhir/fhir-engine.md#fhirsource) for the Patient. It references the EhrBrandLocation the Patient picked, which in turn identifies the brand.
- Its label defaults to `<vendor name> - <brand name>`.
- A Patient has one FhirSource per brand and Data Source. Connecting again returns the existing source instead of creating another, even if the Patient picks a different location of the same brand.
- Records are stored under the source and are unique per EHR record ID, so a repeat sync updates them in place and does not create duplicates.

### Configuration

For this example we will use the Epic MyChart Sandbox.

#### EHR

- Log in to the JHE Admin UI with superuser permissions and click on the "EHRs" menu. There should be an EHR Vendor for "Epic Sandbox" - click on the update icon and set the EHR Client ID that is [provided by Epic](https://fhir.epic.com/Documentation?docId=patientfacingfhirapps). Select all the scopes for this example test.

#### Data Source

- Ensure a Data Source is configured with the "All FHIR Resources" scope and label it something appropriate, eg "EHR Patient Portal".

#### JHE Client

- Ensure a Client is configured with the Invitation URL `https://jhe.fly.dev/clients/ehr-patient-portal/?code=CODE`, associate it with the above Data Source and label it something appropriate, eg "EHR Patient Portal".

#### Study

- Create a Study that includes the requested scope "All FHIR Resources" and attach the Data Source and Client from above.

### Test the flow

```{important}
The External ID for the EHR Patient must be unique in JHE. If you test with the same Epic test account across several JHE Patients, delete the External ID from the previous Patient as you go. Update the Patient in the Admin UI and remove the entry under "External Identifiers". Otherwise the import stops with "failed to store EHR Patient Portal patient id".
```

1. Add a Patient to the Study, view the Patient, and below the "EHR Patient Portal" Client click the "Generate Invitation Link" button
1. Copy and paste this link into a new Incognito browser window
1. The JHE Web UI will take you through the flow to provide consent for the "All FHIR Resources" scope
1. The JHE Web UI will then redirect you to the Epic MyChart login, enter `username: fhircamila` and `password: epicepic1`
1. The Epic Web UI will then ask you to consent the requested scopes
1. The Epic Web UI will then redirect you back to the JHE Web UI that will import the records into JHE
1. Once complete, return to the JHE Admin UI and click on the FHIR Resource menu
1. Take a note of the JHE Patient ID from step 1
1. Choose the corresponding Organization and Study from the dropdowns, select a Resource that you expect from the chart (eg Condition), choose "External" from the Source and then enter the numeric JHE Patient ID from step 8 (eg 40001). You should now see the associated records displayed.
