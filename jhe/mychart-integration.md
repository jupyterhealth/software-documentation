---
title: MyChart (Epic) Integration
---

JHE can pull a patient's clinical data from [Epic MyChart](https://www.epic.com/software/mychart/) (and any SMART on FHIR server) using the **SMART on FHIR standalone patient launch**.

This page is the single onboarding doc for the MyChart client: setup, configuration, end-to-end test. The flow is **vendor-agnostic** - only the FHIR endpoint (`iss`) and brand differ per provider; the Epic work generalizes to other SMART servers.

| What                                    | Where                                                                                                                                                                      |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Page views + identifier proxy           | [`core/views/mychart.py`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/mychart.py)                                                         |
| Patient-facing connect / callback pages | [`core/templates/clients/mychart/`](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/templates/clients/mychart)                                      |
| Browser SMART flow (vanilla JS)         | [`core/static/clients/mychart/js/client-mychart.js`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/static/clients/mychart/js/client-mychart.js)   |
| Vendored SMART library                  | [`core/static/clients/mychart/js/fhir-client.min.js`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/static/clients/mychart/js/fhir-client.min.js) |
| Client + config seed                    | [`core/management/commands/seed.py`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/management/commands/seed.py) (`seed_clients`)                  |
| Tests                                   | [`tests/backend/test_mychart.py`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/tests/backend/test_mychart.py)                                         |

## Architecture

Unlike the [OW integration](./ow-integration.md) (a server-side poller), the MyChart flow runs **client-side in the browser**: the vanilla-JS page hosted on JHE redeems the invitation for a JHE token, runs the Epic PKCE OAuth, pulls the patient's FHIR data, and writes it back into JHE. JHE's backend only serves the pages, holds the Epic config in `JheClient.aux_data`, and exposes one additive identifier endpoint.

```
Patient browser ──▶ /clients/mychart/?code=<invitation>   (connect.html)
                     │
                     ├─ POST /api/v1/invitation/<token>   ──▶ JHE: redeem invitation
                     └─ POST /o/token/                    ──▶ JHE: JHE access token (PKCE)
                     │
                     └─ FHIR.oauth2.authorize(iss, clientId, scope, redirectUri)
                            │
              Epic MyChart  ◀────────── login + consent ──┘
                            │
                            ▼
   /clients/mychart/callback (callback.html)
       │  FHIR.oauth2.ready()  ──▶ Epic access token + patient id
       │
       ├─ POST /api/v1/mychart/identifier       ──▶ PatientIdentifier (Epic patient id)
       ├─ POST /api/v1/fhir_sources             ──▶ FhirSource id
       ├─ GET  <Epic>/Observation?category=...  ──▶ pull Labs
       └─ POST /FHIR/R5/Observation             ──▶ FhirAuxResource
              (header X-JHE-FHIR-Source-ID)         "Labs: N records"
```

The Epic patient id is stored as a `PatientIdentifier` (`system` = the Epic `iss`, `value` = the Epic patient id). The pulled Observations are non-OMH FHIR resources, so they are stored as [auxiliary FHIR resources](./fhir/fhir-api.md) (`FhirAuxResource`), linked to a `FhirSource` the patient registers on the fly. Re-running attaches new aux rows.

## Configuration

### Epic config (`JheClient.aux_data`)

The seeded **MyChart** client carries the Epic config in its `aux_data` JSON blob. The connect/callback views inject it into the page; `fhir-client.js` discovers the authorize/token endpoints from `{iss}/.well-known/smart-configuration`.

```json
{
  "iss": "https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4",
  "client_id": "<Epic non-production client id>",
  "scopes": "openid profile launch/patient patient/Patient.read patient/Observation.read"
}
```

### Epic developer app (one-time, Epic side)

Epic performs the OAuth, so the **redirect URI is registered on the Epic app** at [fhir.epic.com](https://fhir.epic.com), not in JHE.

1. In the Epic app, open the redirect URIs (under the app's **Edit** page).
1. Add, byte-for-byte, the MyChart **callback** URL for each environment:
   - Local: `http://localhost:8010/clients/mychart/callback`
   - Fly: `https://jhe.fly.dev/clients/mychart/callback`
1. Ensure the app's **Incoming APIs** include `Patient.Read (R4)` and `Observation.Search/Read (R4)`, audience **Patients**, public client (PKCE).
1. Save.

> Epic can take up to ~1-12h to sync a new redirect URI before it works. If the callback errors with a `redirect_uri` mismatch right after editing, wait and retry.

The Epic redirect is the **only** thing registered with Epic - JHE's own `Application.redirect_uris` is the JHE OAuth callback (`/auth/callback`), used by the invitation -> `/o/token/` exchange, and is unrelated to Epic.

### Local port (temporary: 8010)

The Epic redirect registered for the Phase 1 PoC is on port **8010**, so the JHE MyChart pages must be served on `:8010` locally to match Epic byte-for-byte. This is temporary - revert to JHE's normal `:8001` once the Epic redirect is updated. (See the canonical local-test ports in the [OW integration](./ow-integration.md#local-setup-docker) page; the MyChart client is the reason 8010 is reserved.)

## Local Setup (Docker)

Seed and serve **everything on `:8010`**. Seeding with `SITE_URL=http://localhost:8010`
registers the JHE Admin UI app's OAuth redirect as `:8010/auth/callback`, so the
admin login works on `:8010` with no manual step. The seed also creates the
**Epic MyChart** `DataSource` (every `FhirSource` needs one) and wires the MyChart
client to the example patient **Peter** (`ll_patient_peter@example.com`), so the
end-to-end flow is testable out of the box.

```bash
cd jupyterhealth-exchange

docker compose up -d db

# Seed the DB on :8010 (creates the MyChart client + config, links it to Peter,
# and registers the :8010 admin OAuth redirect). --flush-db for a clean DB.
docker compose run --rm -e DB_HOST=db -e SITE_URL=http://localhost:8010 \
  web python manage.py seed --flush-db

# Serve the JHE stack on host :8010 with SITE_URL set to match
docker compose run --rm -d --name jhe-web-8010 -p 8010:8010 \
  -e DB_HOST=db -e SITE_URL=http://localhost:8010 -e DEBUG=True \
  web python manage.py runserver 0.0.0.0:8010
```

> If you already have a seeded DB on `:8001` and don't want to re-seed, instead set
> the `site.url` to `:8010` (invitation links embed this host), add the `:8010` admin
> redirect, and link the MyChart client to Peter, by hand:
>
> ```bash
> docker compose run --rm -e DB_HOST=db web python manage.py shell -c "
> from oauth2_provider.models import get_application_model as G
> from core.models import StudyClient, Study, JheSetting
> s=JheSetting.objects.get(key='site.url'); s.set_value('string','http://localhost:8010'); s.save()
> a=G().objects.get(name='JHE Admin UI'); u=a.redirect_uris.split()
> x='http://localhost:8010/auth/callback'
> a.redirect_uris=' '.join(u+[x]) if x not in u else a.redirect_uris; a.save()
> StudyClient.objects.get_or_create(study=Study.objects.get(name='Lifespan Study on BP & HR'), client=G().objects.get(name='MyChart'))
> "
> ```
>
> After changing `site.url`, generate a **fresh** invitation link - older links have
> the previous host baked in.

Verify:

| Check                                                                    | Expected          |
| ------------------------------------------------------------------------ | ----------------- |
| `GET http://localhost:8010/clients/mychart/`                             | 200, connect page |
| `GET http://localhost:8010/static/clients/mychart/js/fhir-client.min.js` | 200               |
| Page source contains `MYCHART_CONFIG` with the Epic `iss`                | yes               |

Stop the test server with `docker rm -f jhe-web-8010`.

## End-to-End Test

| Step             | Command / Action                                                                                                                                                                                | Expected                                                                      |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| 1. Unit tests    | `python -m pytest tests/backend/test_mychart.py`                                                                                                                                                | 7 passed                                                                      |
| 2. Practitioner  | Log in at `http://localhost:8010/` as `manager_mary@example.com` / `Jhe1234!`, open patient **Peter** (`ll_patient_peter@example.com`), **Generate Invitation Link** for the **MyChart** client | Link points at `http://localhost:8010/clients/mychart/?code=...`              |
| 3. Connect       | Open the invitation link in a new tab                                                                                                                                                           | "Redeeming invitation..." → "JHE access token received" → redirect to MyChart |
| 4. MyChart login | Log in as an Epic sandbox MyChart test patient (e.g. Camila Lopez `fhircamila`)                                                                                                                 | Epic consent screen, then return to `/clients/mychart/callback`               |
| 5. Import        | (automatic on the callback page)                                                                                                                                                                | Shows the MyChart patient id, "Stored ... in JHE", and **"Labs: N records"**  |
| 6. Identifier    | JHE Admin UI → Peter                                                                                                                                                                            | A `PatientIdentifier` with `system` = Epic `iss`, `value` = Epic patient id   |
| 7. Data          | JHE Admin UI → Peter's observations (or query `FhirAuxResource` for `Observation`)                                                                                                              | N Observation aux rows linked to Peter's `FhirSource`                         |

The MyChart patient id is whatever the Epic sandbox account maps to; it is stored on the JHE patient (Peter) you invited, regardless of name.

Epic sandbox MyChart test patients (those with a "MyChart Login") are listed at <https://fhir.epic.com/Documentation?docId=testpatients>.

## Troubleshooting

| Symptom                                                      | Likely cause                                                                                                                                                                                                                                                                             |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `redirect_uri` mismatch on the Epic callback                 | The served origin/port ≠ the URI registered on the Epic app, or < ~1-12h since the edit. Use `:8010`.                                                                                                                                                                                    |
| "no invitation code in URL"                                  | Opened `/clients/mychart/` directly; start from a real invitation link.                                                                                                                                                                                                                  |
| Redeeming invitation 400s against `localhost:8000` (uvicorn) | `site.url` doesn't match the served host, so the invitation code embeds the wrong host. Set `site.url` to `http://localhost:8010` and generate a fresh link.                                                                                                                             |
| "failed to exchange invitation for JHE token"                | The MyChart client's `Application.redirect_uris` isn't the JHE `/auth/callback`. Re-run `seed_clients`.                                                                                                                                                                                  |
| Blank / no patient context after MyChart login               | Missing `launch/patient` scope in `aux_data.scopes`.                                                                                                                                                                                                                                     |
| "(N records could not be saved)"                             | A record failed the JHE FHIR write validation. Common cause: Epic ("Unconstrained FHIR IDs") emits a resource `id` over the FHIR 64-char limit; the client relocates an over-long `id` into an `identifier` before writing. Inspect the `OperationOutcome` response for the exact field. |
| `invalid scope`                                              | A `patient/<Resource>.read` scope is requested but that API isn't enabled on the Epic app.                                                                                                                                                                                               |

## Gotchas (lessons from first end-to-end)

The flow touches three OAuth layers (JHE invitation, JHE token, Epic SMART) plus the
FHIR write, so a wrong value in any one stops the chain. The snags hit while wiring it,
in flow order:

1. **Serve and seed on the same port.** Everything (admin UI, invitation link, redirects)
   must agree on `:8010`. Seed with `SITE_URL=http://localhost:8010` so the JHE Admin UI
   OAuth redirect is `:8010/auth/callback`; otherwise the admin login fails with
   "Mismatching redirect URI".
1. **`site.url` must match the served host.** Invitation links embed the host from the
   `site.url` setting. If `site.url` is `:8000` but you serve on `:8010`, the browser
   redeems the invitation against `localhost:8000` (the Open Wearables backend) and gets a
   400\. Seeding on `:8010` sets this correctly.
1. **Use the right Epic app's client id.** The redirect URIs must be registered on the
   **same** Epic app whose non-production client id is in `aux_data.client_id`. A mismatch
   gives Epic authorize `error=4` ("request is invalid"). Verify the client id on the app
   that actually lists `localhost:8010/clients/mychart/callback`.
1. **A `FhirSource` needs a `DataSource`.** The seed creates the "Epic MyChart"
   `DataSource`; the views inject its id and the client sends it. Missing it →
   `{"dataSource":["This field is required."]}`.
1. **FHIR `id` 64-char limit.** Epic ("Unconstrained FHIR IDs") emits ids over 64 chars,
   which the JHE write rejects. The client relocates an over-long `id` into an `identifier`
   before writing.

## Out of scope (follow-up)

- Epic Brands / hospital-search provider directory (seeded into `JheClient.aux_data.locations`) so a patient can pick their own hospital.
- Production (non-sandbox) Epic app and promotion.
- Multi-vendor (Cerner/Oracle Health, Athena) and refresh-token handling.
