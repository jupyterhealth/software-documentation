---
title: Provider EHR Launch (SMART on FHIR)
---

Clinicians work in the EHR; the remote-monitoring data their patients generate — CGM
traces, wearable vitals, sleep — lives in JupyterHealth Exchange. This flow puts that
data **inside the clinical workflow**: the provider opens a patient's chart, clicks a
JupyterHealth app, and sees that patient's device data right there — already in patient
context, already authenticated. No separate portal, no second login, no re-finding the
patient, and JHE's access control still applies (the provider sees only patients they're
authorized for in JHE).

Technically, the launched app is a **SMART on FHIR provider app** (Epic, Cerner,
MedPlum, …) that trades the EHR-issued OIDC **`id_token`** for a **JHE user-bound access
token** over OAuth 2.0 Token Exchange ([RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693)) —
that exchange is what makes the single login possible, and it's what this page documents.

JHE verifies the `id_token` **offline against the EHR's JWKS** (discovered from
`{iss}/.well-known/smart-configuration`), identifies the provider from the **`fhirUser`**
claim, and issues its own access token bound to the matching JHE `Practitioner`. No EHR
`userinfo` or introspection endpoint is required — verification relies only on
capabilities every ONC §170.315(g)(10)-certified EHR must expose (OIDC `id_token` with
`fhirUser` + published JWKS), so the same code path works across EHR vendors with no
per-vendor plumbing.

This is the **provider-side** sibling of the patient-side
[MyChart (Epic) integration](./mychart-integration.md): MyChart pulls a patient's
clinical data *into* JHE; this flow lets a provider's launched app read JHE data *out*,
as that provider, under JHE's normal access control.

<!-- TODO(flip-to-ready): the oidc_verify.py and test_token_exchange.py links below
     point at the feature branch; switch them to jupyterhealth-exchange main when PR #646
     merges and this PR is marked ready. -->

| What                                   | Where                                                                                                                                                                                      |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Token-exchange endpoint                | [`core/views/common.py`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/common.py) (`token_exchange`)                                                        |
| id_token verification (JWKS discovery) | [`core/oidc_verify.py`](https://github.com/the-commons-project/jupyterhealth-exchange/blob/ehr-launch-jhe-sso-with-client/core/oidc_verify.py)                                             |
| Client + settings seed                 | [`core/management/commands/seed.py`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/management/commands/seed.py) (`seed_sof_ehr_launch_application`, `auth.sof.*`) |
| Tests                                  | [`tests/backend/test_token_exchange.py`](https://github.com/the-commons-project/jupyterhealth-exchange/blob/ehr-launch-jhe-sso-with-client/tests/backend/test_token_exchange.py)           |
| Reference app (template)               | [jupyterhealth-sof-provider-template](https://github.com/jupyterhealth/jupyterhealth-sof-provider-template) — see also the [MedPlum tutorial](../tutorial/medplum-provider-dashboard.md)   |

## Architecture

```
                ┌──────────────────── EHR (Epic, Cerner, MedPlum, …) ───────────────────┐
                │  SMART App Launch (openid fhirUser)                                    │
  Provider ───▶ │  → issues id_token (iss = EHR, aud = app's EHR client_id, fhirUser)    │
                └────────────────────────────────────────────────────────────────────────┘
                                 │  id_token (+ the app's JHE client_id/secret)
                                 ▼
  ┌───────────────────── External SMART app (confidential backend) ───────────────────┐
  │  POST /o/token-exchange  (client auth + subject_token = id_token)                  │
  └────────────────────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
  ┌───────────────────────────── JHE /o/token-exchange ───────────────────────────────┐
  │  1. Authenticate the confidential client (RFC 6749 client auth)                    │
  │  2. Read id_token `iss`; require it ∈ auth.sof.trusted_issuers                     │
  │  3. Discover JWKS: {iss}/.well-known/smart-configuration → jwks_uri                │
  │  4. Verify signature + exp/iat + aud == auth.sof.trusted_audience                  │
  │  5. fhirUser ("Practitioner/<id>") → JheUser.identifier == <id>                    │
  │  6. Issue access token: user = the Practitioner, application = the client          │
  └────────────────────────────────────────────────────────────────────────────────────┘
                                 │  { access_token, token_type: Bearer, expires_in }
                                 ▼
                 App calls JHE FHIR API with  Authorization: Bearer <access_token>
```

### Two distinct client identities (do not conflate)

|                            | Where it lives                           | What it is                                           | Checked against               |
| -------------------------- | ---------------------------------------- | ---------------------------------------------------- | ----------------------------- |
| **EHR client_id**          | inside the `id_token` as `aud`           | the app's registration **at the EHR**                | `auth.sof.trusted_audience`   |
| **JHE client_id + secret** | the token-exchange request (client auth) | the app's registration **in JHE** ("SoF EHR Launch") | JHE's OAuth application store |

The `id_token` proves *which provider* is launching; the JHE client credential proves
*which app* is calling JHE. Both must be satisfied.

## Configuration

### Trust settings (`JheSetting` rows)

Configured at runtime via JheSettings (not env vars). Both are created by
`python manage.py seed`; update them via the JHE settings admin/API (values are cached
for ~60s, so a change takes effect within that window):

| Key                         | Type     | Meaning                                                                                    |
| --------------------------- | -------- | ------------------------------------------------------------------------------------------ |
| `auth.sof.trusted_issuers`  | `json`   | EHR OIDC issuers (`id_token.iss`) whose tokens are accepted; JWKS is discovered from each. |
| `auth.sof.trusted_audience` | `string` | The SMART app's `client_id` at the EHR (the `id_token.aud`).                               |

### The "SoF EHR Launch" confidential client

`manage.py seed` also creates a confidential OAuth `Application` named **SoF EHR
Launch** that external apps authenticate as. The seeded credentials
(`sof-ehr-launch` / `sof-ehr-launch-dev-secret`) are **dev-only placeholders — rotate
them for any real deployment** (or override at seed time via
`SOF_EHR_LAUNCH_CLIENT_ID` / `SOF_EHR_LAUNCH_CLIENT_SECRET`).

The app must run the exchange **server-side** (the template runs it in the Jupyter/Voilà
backend) so the secret never reaches a browser.

### Per-provider mapping

Each launching clinician must exist in JHE as a `Practitioner` whose
`JheUser.identifier` equals the **bare FHIR id** from the EHR's `fhirUser` claim
(`fhirUser: "Practitioner/abc123"` → `identifier == "abc123"`). Unmatched providers
get `404`.

## The request

`POST /o/token-exchange` — `application/x-www-form-urlencoded`.

| Parameter              | Required | Value                                                |
| ---------------------- | -------- | ---------------------------------------------------- |
| `grant_type`           | ✓        | `urn:ietf:params:oauth:grant-type:token-exchange`    |
| `subject_token`        | ✓        | the EHR-issued `id_token` (a JWT)                    |
| `subject_token_type`   | ✓        | `urn:ietf:params:oauth:token-type:id_token`          |
| `requested_token_type` | ✓        | `urn:ietf:params:oauth:token-type:access_token`      |
| `audience`             | ✓        | JHE's site URL (must equal JHE's `SITE_URL` exactly) |
| `scope`                | ✓        | `openid` (only value supported)                      |
| `client_id`            | ✓        | the app's JHE confidential client id                 |
| `client_secret`        | ✓        | the app's JHE confidential client secret             |

Client credentials may be sent as form fields (above) or via HTTP Basic
(`Authorization: Basic …`) — both placements are accepted.

```bash
curl -X POST 'https://jhe.example/o/token-exchange' \
  --data-urlencode 'grant_type=urn:ietf:params:oauth:grant-type:token-exchange' \
  --data-urlencode 'subject_token='"$EHR_ID_TOKEN" \
  --data-urlencode 'subject_token_type=urn:ietf:params:oauth:token-type:id_token' \
  --data-urlencode 'requested_token_type=urn:ietf:params:oauth:token-type:access_token' \
  --data-urlencode 'audience=https://jhe.example' \
  --data-urlencode 'scope=openid' \
  --data-urlencode 'client_id=sof-ehr-launch' \
  --data-urlencode 'client_secret=sof-ehr-launch-dev-secret'
```

Success (`200`):

```json
{
  "access_token": "9f8c…",
  "issued_token_type": "urn:ietf:params:oauth:token-type:access_token",
  "token_type": "Bearer",
  "expires_in": 1209600,
  "scope": "openid"
}
```

The issued token is a normal JHE bearer token: it carries the Practitioner's regular
authorization (organizations → studies → consents), and can be revoked at
`/o/revoke_token` with the same client credentials. No refresh token is issued; the app
re-exchanges per launch.

### Errors

| Status | Cause                                                                                                               |
| ------ | ------------------------------------------------------------------------------------------------------------------- |
| `400`  | Missing/invalid parameter, malformed JWT, wrong `subject_token_type`, `audience` ≠ JHE site URL, missing `fhirUser` |
| `401`  | Client authentication failed; or `id_token` signature/`exp`/`aud` invalid                                           |
| `403`  | `id_token.iss` not trusted; or `fhirUser` is not a Practitioner                                                     |
| `404`  | No `Practitioner` with `JheUser.identifier` == the `fhirUser` id                                                    |
| `500`  | Token exchange not configured (`auth.sof.*` unset)                                                                  |
| `502`  | JWKS discovery / signing-key resolution failed at the issuer                                                        |

## Security notes / known limitations

- **Offline verification, no revocation of the id_token.** A valid `id_token` is honored
  until its `exp`; an EHR-side session revocation is not reflected until then. Prefer
  short EHR `id_token` lifetimes.
- **No replay / one-time-use tracking.** A captured `id_token` can be re-exchanged within
  its validity window. Client authentication is the primary control; a leaked `id_token`
  alone is not sufficient without the client secret.
- **Identity mapping is issuer-unscoped.** `fhirUser` is matched on the bare FHIR id,
  which assumes **one trusted EHR per JHE instance**. Do not configure issuers from
  multiple EHRs whose Practitioner ids could collide; `auth.sof.trusted_audience` is
  likewise a single value.
- Issued JHE tokens have the standard JHE access-token lifetime (2 weeks by default) and
  are bound to both the Practitioner and the calling application, so they show up in —
  and can be revoked through — the normal per-client token management.
