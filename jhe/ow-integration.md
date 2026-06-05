---
title: Open Wearables Integration
---

JHE can ingest observations from an [Open Wearables](https://github.com/the-momentum/open-wearables) (OW) backend.

This page is the single onboarding doc for the OW PoC: setup, configuration, end-to-end test.

| What                                   | Where                                                                                                                                          |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| OAuth proxy + user creation views      | [`core/views/ow.py`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/views/ow.py)                                       |
| Patient-facing launch / complete pages | [`core/templates/clients/ow/`](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/core/templates/clients/ow)                    |
| Polling command (every 15 min)         | [`core/management/commands/ow_poll.py`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/management/commands/ow_poll.py) |
| Raw S3 reader (raw mode only)          | [`core/services/ow_ingest.py`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/services/ow_ingest.py)                   |
| Cron sidecar schedule                  | [`deploy/crontab`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/deploy/crontab)                                           |
| Tests                                  | [`tests/backend/test_ow_poll.py`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/tests/backend/test_ow_poll.py)             |

## Architecture

```
Patient browser  ──▶  /ow/?code=<invitation>     (launch.html)
                       │
                       ├─ POST /api/v1/ow/users           ──▶ OW: create user
                       └─ GET  /api/v1/ow/oauth/oura/...  ──▶ OW: provider auth URL
                                                                │
                Oura  ◀────────── OAuth ──────────────────────┘
                  │
                  └─▶ /api/v1/oauth/oura/callback (proxied to OW)

cron (15 min) ──▶ manage.py ow_poll
                   │
                   ├─ normalized: GET <OW>/api/v1/users/<id>/timeseries
                   └─ raw:        list S3 bucket → fetch JSON
                   │
                   └─ omh_shim.convert(...) ─▶ Observation + ObservationIdentifier
```

Dedup: every ingested record gets a paired `ObservationIdentifier` row keyed
by `system="ow:normalized"` (or `"ow:raw"`) and `value=<omh-header.uuid>`.
The unique constraint on `(system, value)` makes re-runs idempotent.

## Configuration

### Environment variables (`.env`)

```env
OW_API_URL="http://localhost:8001"
OW_API_KEY="sk-<your-OW-admin-API-key>"
```

Read by `jhe/settings.py` and used by both `core/views/ow.py` (proxy) and
`core/management/commands/ow_poll.py` (poller).

### Oura developer app (one-time, OW side)

OW performs the Oura OAuth, so the Oura developer app is configured against
**OW's** public URL, not JHE's. In the Oura Cloud developer portal for the app:

1. Open **Redirect URIs**.
1. Add, byte-for-byte, `<OW_PUBLIC_URL>/api/v1/oauth/oura/callback`
   (Fly PoC: `https://ow-poc.fly.dev/api/v1/oauth/oura/callback`).
1. Save.

If the redirect URI does not match exactly, Oura rejects the callback and no
data ever reaches OW. For the PoC you can reuse the existing Oura developer
account's `OURA_CLIENT_ID` / `OURA_CLIENT_SECRET` (set on the OW backend, not
JHE) rather than registering a new app - a dedicated dev app per environment is
a later step.

### Runtime toggles (JheSettings)

| Key                   | Type   | Default      | Purpose                                                                                                                                                                                  |
| --------------------- | ------ | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `module.ow`           | bool   | `false`      | Master switch. `ow_poll` no-ops when false.                                                                                                                                              |
| `ow.ingest_mode`      | string | `normalized` | `normalized` (HTTP) or `raw` (S3).                                                                                                                                                       |
| `ow.sync_in_progress` | string | `""`         | Lock auto-managed by `ow_poll`. Stores the acquiring tick's ISO timestamp (empty = unlocked). Locks older than 30 minutes are treated as abandoned and force-reclaimed by the next tick. |

Raw-mode S3 settings (required when `ow.ingest_mode=raw` - **no defaults**; the
poller raises a clear error if any of the four below are unset):

| Key                       | Default                          |
| ------------------------- | -------------------------------- |
| `ow.s3.endpoint_url`      | (required)                       |
| `ow.s3.access_key_id`     | (required)                       |
| `ow.s3.secret_access_key` | (required)                       |
| `ow.s3.bucket_name`       | (required)                       |
| `ow.s3.key_prefix`        | `raw-payloads/oura/api_response` |

Toggle from the Django shell:

```python
from core.models import JheSetting

s, _ = JheSetting.objects.update_or_create(
    key="module.ow", setting_id=None, defaults={"value_type": "bool"}
)
s.set_value("bool", True)
s.save()
```

## Local Setup (from scratch)

```bash
# 1. JHE
cp dot_env_example.txt .env       # then fill OW_API_URL / OW_API_KEY
docker compose up -d db
pipenv install --dev
pipenv shell
python manage.py migrate
python manage.py seed             # creates "Oura" data source + "OW Local" client
python manage.py runserver 0.0.0.0:8000

# 2. OW backend (separate repo)
#    Clone https://github.com/the-momentum/open-wearables and follow its README.
#    Default port 8001. Generate an admin API key, paste into JHE's .env.

# 3. Enable OW module + first poll
python manage.py shell -c "
from core.models import JheSetting
s,_ = JheSetting.objects.update_or_create(key='module.ow', setting_id=None, defaults={'value_type':'bool'})
s.set_value('bool', True); s.save()
"
python manage.py ow_poll
```

## Production Setup

1. Set `OW_API_URL` and `OW_API_KEY` in your deployment env (Fly secrets, K8s Secret, etc.).

1. Apply migrations: `python manage.py migrate`.

1. Run `python manage.py seed` once, or otherwise ensure the `Oura` DataSource + `omh:heart-rate:2.0` CodeableConcept exist.

1. Run the `jhe_cron` sidecar so [`deploy/crontab`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/deploy/crontab) fires `ow_poll` every 15 minutes (the Dockerfile installs `supercronic`):

   ```yaml
   # docker-compose / k8s sidecar
   command: ["supercronic", "/code/deploy/crontab"]
   ```

1. Flip `module.ow=true` in JheSettings when ready to start ingesting.

## End-to-End Test

| Step             | Command / Action                                                                                                           | Expected                                                                                                                   |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| 1. Unit tests    | `python -m pytest tests/test_ow_poll.py`                                                                                   | 11 passed                                                                                                                  |
| 2. Smoke poll    | `python manage.py ow_poll`                                                                                                 | `OW poll complete (mode=normalized). Created N observations.`                                                              |
| 3. Practitioner  | Log in as an org manager (e.g. `manager_mary@example.com`), attach `Oura` data source to a study, generate invitation link | URL contains `?code=...`; host part is your real JHE host, not `localhost`                                                 |
| 4. Patient       | Open invitation in incognito, sign up, agree to consents (Heart Rate checked), complete Oura OAuth                         | Lands on "Successfully Connected"                                                                                          |
| 5. JheUser check | Django admin → JheUser of the patient                                                                                      | `identifier = "ow:<oura-user-id>"`                                                                                         |
| 6. Consent gate  | Revoke Heart Rate scope, run `ow_poll`                                                                                     | 0 observations created for that patient                                                                                    |
| 7. Ingest        | Re-grant Heart Rate, run `ow_poll`                                                                                         | ≥1 Observation created                                                                                                     |
| 8. Dedup         | Run `ow_poll` again immediately                                                                                            | `Created 0 observations.`                                                                                                  |
| 9. Provenance    | Open new Observation in admin                                                                                              | `data_source=Oura`, `coding_code=omh:heart-rate:2.0`, paired `ObservationIdentifier(system="ow:normalized", value=<uuid>)` |

`ow_poll` looks back one day per run and resumes from a high-water-mark, so a
patient's older Oura history is not backfilled by default. For a one-time
backfill on first connect, widen the window for a single run:

```bash
python manage.py shell -c "
from datetime import timedelta
from core.management.commands import ow_poll
ow_poll.POLL_WINDOW = timedelta(days=90)
from django.core.management import call_command
call_command('ow_poll')
"
```

## Troubleshooting

| Symptom                                                   | Fix                                                                                                                                                                                                 |
| --------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OW poll skipped: module.ow=false`                        | Set `module.ow=true` in JheSettings (see above).                                                                                                                                                    |
| `ow_poll aborted: OW_API_URL / OW_API_KEY not configured` | Add both vars to `.env` and restart the server.                                                                                                                                                     |
| `401 Invalid or missing API key`                          | Stale `OW_API_KEY`; regenerate in OW admin and update `.env`.                                                                                                                                       |
| `ow_poll skipped: ow.sync_in_progress since <iso>`        | A previous tick is still running. If the timestamp is older than 30 minutes the next tick will auto-reclaim the lock and proceed; to force an immediate reclaim, set `ow.sync_in_progress` to `""`. |
| `CodeableConcept 'omh:heart-rate:2.0' not found`          | Run `python manage.py seed`.                                                                                                                                                                        |
| Patient skipped silently                                  | Either `JheUser.identifier` doesn't start with `ow:`, or patient hasn't consented to `omh:heart-rate:2.0`.                                                                                          |
