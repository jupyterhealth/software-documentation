---
title: Patient Access Integration
---

JHE can pull a patient's clinical data from any **SMART on FHIR** patient-access API
(Epic, Cerner/Oracle Health, Athena, ...) using the **SMART App Launch standalone patient
launch**. Epic's sandbox is the seeded example, but nothing in the client is Epic-specific:
a patient searches for their hospital, picks it, is sent to that hospital's login, and their
records are converted and stored in JHE.

This is the single onboarding doc for the **Patient Access** client: hospital branding,
setup, configuration, and an end-to-end test. The flow is **vendor-agnostic** - only the FHIR
endpoint (`iss`) and brand differ per provider.

| What                                          | Where                                                                                                                                    |
| --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Page views + brands search + identifier proxy | `core/views/patient_access.py`                                                                                                           |
| Hospital-brand models                         | `core/models/ehr_brand.py`                                                                                                               |
| Brands importer + sample bundle               | `core/management/commands/import_ehr_brands.py`                                                                                          |
| Connect / callback pages                      | `core/templates/clients/patient-access/`                                                                                                 |
| Browser SMART flow (vanilla JS)               | `core/static/clients/patient-access/js/client-patient-access.js`                                                                         |
| Client + config seed                          | [`core/management/commands/seed.py`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/management/commands/seed.py) |
| Tests                                         | `tests/backend/test_patient_access*.py`, `tests/frontend/tests/patient-access-*.test.js`                                                 |

## Architecture

Unlike the [OW integration](./ow-integration.md) (a server-side poller), the Patient Access
flow runs **client-side in the browser**: a vanilla-JS page hosted on JHE redeems the
invitation for a JHE token, lets the patient pick their hospital, runs the EHR's PKCE OAuth,
pulls the patient's FHIR data, and writes it back into JHE. The backend serves the pages,
holds the EHR config in `JheClient.aux_data`, searches the hospital-brand tables, converts
each pulled R4 resource to R5, and stores it.

```
Patient browser ─▶ /clients/patient-access/?code=<invitation>   (connect.html)
   ├─ POST /api/v1/invitation/<token>          ─▶ redeem invitation
   ├─ POST /o/token/                           ─▶ JHE access token (PKCE)
   ├─ GET  /api/v1/patient-access/brands?q=    ─▶ hospital search (pick one -> its iss)
   └─ FHIR.oauth2.authorize(iss, clientId, scope, redirectUri)
          │
     EHR login ◀───── login + consent ─────────┘
          │
          ▼
   /clients/patient-access/callback   (callback.html)
       │  FHIR.oauth2.ready()  ─▶ EHR access token + patient id
       ├─ POST /api/v1/patient-access/identifier  ─▶ PatientIdentifier (EHR patient id)
       ├─ POST /api/v1/fhir_sources               ─▶ FhirSource id
       ├─ GET  <EHR>/{Patient,Condition,...}      ─▶ pull each USCDI type (R4)
       └─ POST /fhir-import/R4/{type}             ─▶ convert R4->R5, store as FhirAuxResource
              (header X-JHE-FHIR-Source-ID)          "Conditions: N", "Labs: N", ...
```

The EHR patient id is stored as a `PatientIdentifier` (`system` = the EHR `iss`, `value` =
the EHR patient id). The client pulls the demo USCDI set (Patient demographics, Condition,
MedicationRequest, AllergyIntolerance, lab Observations); each type is fetched and written
independently so one failure does not abort the rest. Because the EHR serves **R4** and JHE
stores **R5**, each resource is POSTed to the [`/fhir-import/R4/` endpoint](./fhir/fhir-engine.md),
which converts it via the HL7 cross-version maps and then runs the normal create - landing
non-OMH resources in `FhirAuxResource`, linked to a `FhirSource` the patient registers on the fly.

## Hospital branding / selection (SMART App Launch 2.2)

A JHE invitation doesn't say which hospital the patient uses, so the connect page shows a
**search box** first. This implements SMART 2.2 "user-access brands".

- **Data model** (`ehr_brand.py`):
  `EhrBrand` (one per health system: `name`, `vendor`, unique `fhir_base_url` = the SMART
  `iss`, `npi`, `logo_url`) and `EhrBrandLocation` (one per facility: `name`, `address_text`,
  `city`, `state`, `postal_code`, FK to brand). Vendor-neutral by design - no Epic-only columns.
- **Search API**: `GET /api/v1/patient-access/brands?q=&state=&postal=` (authenticated). `q`
  matches facility name, city, or brand name. Returns facilities with the brand's `fhir_base_url`.
- **Selection**: picking a facility calls `FHIR.oauth2.authorize({ iss: <brand.fhir_base_url> })`.
  `fhir-client.js` discovers the authorize/token endpoints from
  `{iss}/.well-known/smart-configuration` - so JHE routes purely on the `iss`, never on
  hardcoded endpoints.

### Seeding + refreshing the brand list

`manage.py seed` loads a small curated sample
(`core/data/ehr_brands.sample.json`)
so the picker has data out of the box. For a full list, download a vendor's **user-access
Brands Bundle** and import it:

```bash
# Epic publishes ~94k organizations / ~800 endpoints:
curl -sSL -o epic_brands.json https://open.epic.com/Endpoints/Brands
docker compose exec web python manage.py import_ehr_brands --file epic_brands.json
```

The importer is idempotent (brands keyed on `fhir_base_url`); re-run to refresh.

### Extending to other EHR vendors

The importer parses any SMART user-access Brands Bundle, so **adding Cerner/Athena/etc. is
importing that vendor's bundle - no code change** for search or selection. The one future
code change for real multi-vendor auth is a **per-vendor `client_id`** (today the single EHR
`client_id` lives in the Patient Access client's `aux_data`). If CMS ships its planned national
brands directory, point the importer at that instead.

## Configuration

### EHR config (`JheClient.aux_data`)

The seeded **Patient Access** client carries the EHR config in its `aux_data` JSON blob. The
connect/callback views inject it into the page; `fhir-client.js` discovers the
authorize/token endpoints from `{iss}/.well-known/smart-configuration`.

```json
{
  "iss": "https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4",
  "client_id": "<EHR non-production client id>",
  "scopes": "openid profile launch/patient patient/Patient.read patient/Observation.read patient/Condition.read patient/MedicationRequest.read patient/AllergyIntolerance.read"
}
```

> The picker overrides `iss` per selected hospital at authorize time; the `client_id` and
> `scopes` in `aux_data` still apply.

### Register the EHR app (one-time, Epic example)

Each deploying institution registers its **own** app at [fhir.epic.com](https://fhir.epic.com);
the redirect URI and APIs live on the EHR app, not in JHE.

1. Sign in at [fhir.epic.com](https://fhir.epic.com), **Build Apps -> Create**. Audience
   **Patients**, **SMART on FHIR standalone** launch, public client (PKCE, no secret).
1. **Incoming APIs (R4)**: `Patient.Read`, `Observation.Search/Read`, `Condition.Search/Read`,
   `MedicationRequest.Search/Read`, `AllergyIntolerance.Search/Read`.
1. **Redirect URIs**: add, byte-for-byte, your **callback** URL per environment:
   - Local: `http://localhost:8001/clients/patient-access/callback`
   - Fly: `https://jhe.fly.dev/clients/patient-access/callback`
1. Save; copy the issued **non-production client id** into the JHE config below.

> Epic can take ~1-12h to sync a new redirect URI. A `redirect_uri` mismatch right after
> editing usually just means it hasn't propagated yet.

### Point JHE at your EHR app

```bash
docker compose exec web python manage.py shell -c "
from oauth2_provider.models import get_application_model as G
c = G().objects.get(name='Patient Access').jhe_client
c.aux_data['client_id'] = '<your EHR non-production client id>'
c.aux_data['iss'] = '<your EHR FHIR base URL>'
c.save()
"
```

## Local setup (Docker)

The default Compose stack serves `:8001` with `SITE_URL=http://localhost:8001`. The seed also
creates the **Patient Access API** `DataSource` (every `FhirSource` needs one), seeds the
sample hospital brands, and wires the Patient Access client to the example patient **Peter**
(`ll_patient_peter@example.com`), so the flow is testable out of the box.

```bash
cd jupyterhealth-exchange
docker compose up --build -d
# wait for the web container's migrate to finish, then seed a clean DB:
docker compose exec web python manage.py seed --flush-db
```

Verify:

| Check                                                                                 | Expected                                     |
| ------------------------------------------------------------------------------------- | -------------------------------------------- |
| `GET http://localhost:8001/clients/patient-access/`                                   | 200, connect page with a hospital search box |
| `GET http://localhost:8001/static/clients/patient-access/js/client-patient-access.js` | 200                                          |
| Page source contains `PATIENT_ACCESS_CONFIG` with the EHR `iss`                       | yes                                          |

## End-to-end test

| Step             | Action                                                                                                                                                                | Expected                                                                                                      |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| 1. Unit tests    | `python -m pytest tests/backend/test_patient_access.py tests/backend/test_patient_access_brands.py`                                                                   | pass                                                                                                          |
| 2. Practitioner  | Log in at `http://localhost:8001/` as `manager_mary@example.com` / `Jhe1234!`, open patient **Peter**, **Generate Invitation Link** for the **Patient Access** client | Link at `.../clients/patient-access/?code=...`                                                                |
| 3. Connect       | Open the link in a new tab                                                                                                                                            | "Redeeming invitation..." → "Choose your hospital" + search box                                               |
| 4. Pick hospital | Search (e.g. `mount`, `san jose`) and click a result                                                                                                                  | Redirect to that hospital's login                                                                             |
| 5. EHR login     | Log in as an Epic sandbox test patient (e.g. Camila Lopez `fhircamila` / `epicepic1`)                                                                                 | Consent, then return to `/clients/patient-access/callback`                                                    |
| 6. Import        | (automatic on the callback page)                                                                                                                                      | EHR patient id, "Stored ... in JHE", per-type counts (Demographics, Conditions, Medications, Allergies, Labs) |
| 7. Data          | JHE Admin UI → **FHIR Resources**, or query `FhirAuxResource` per type                                                                                                | Imported rows for Peter, linked to his `FhirSource`                                                           |

Epic sandbox test patients: <https://fhir.epic.com/Documentation?docId=testpatients>.

## Troubleshooting

| Symptom                                                        | Likely cause                                                                                                                                                                   |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `redirect_uri` mismatch on the EHR callback                    | Served origin/port ≠ the URI registered on the EHR app, or < ~1-12h since the edit. Use `:8001`.                                                                               |
| "no invitation code in URL"                                    | Opened `/clients/patient-access/` directly; start from a real invitation link.                                                                                                 |
| Picker never appears                                           | Invitation not redeemed (no JHE token). Check the invitation link/host.                                                                                                        |
| "failed to exchange invitation for JHE token"                  | The client's `Application.redirect_uris` isn't the JHE `/auth/callback`. Re-seed.                                                                                              |
| A few Conditions rejected with `clinicalStatus field required` | The EHR's data for that patient omits `clinicalStatus`, which R5 requires - correctly rejected, not a bug. The import returns an `OperationOutcome` per record explaining why. |
| Over-long resource `id` rejected                               | Epic ("Unconstrained FHIR IDs") emits ids over the FHIR 64-char limit; the client relocates an over-long `id` into an `identifier` before writing.                             |
| `invalid scope`                                                | A `patient/<Resource>.read` scope is requested but that API isn't enabled on the EHR app.                                                                                      |

## Out of scope (follow-up)

- **Full Epic brands import**: Epic's file uses `urn:uuid` references; the importer resolves
  the sample's `Organization/<id>` style and needs a small tweak to resolve `urn:uuid` for the
  full 94k-entry file. Bulk insert for that volume is also a follow-up.
- Production (non-sandbox) EHR app and promotion.
- Per-vendor `client_id` for simultaneous multi-vendor auth; refresh-token handling.
