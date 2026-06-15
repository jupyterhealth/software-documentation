---
title: MCP Server
---

The JHE MCP Server (`jhe_mcp`) exposes JupyterHealth Exchange (JHE) data to LLM clients (Claude, Gemini, ChatGPT, and any other [Model Context Protocol](https://modelcontextprotocol.io/) client) so a user can query their studies, patients, and observations in natural language.

This page is the single onboarding doc for the MCP server: how it works, how to register it with JHE, how to configure and deploy it, and how to connect an LLM client. The server acts as an **OAuth broker** - it presents an OAuth 2.0 Authorization Server interface to MCP clients while delegating the actual login to JHE. Because JHE enforces per-user RBAC, each user only sees the studies, patients, and observations they are already authorized to access; the MCP server inherits those boundaries automatically.

The server lives in the [`mcp_server/`](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/mcp_server) directory of the JHE repository and is an **optional, standalone service** - it is deployed independently and is not part of a JHE deployment.

| What                              | Where                                                                                                                                                                                                        |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Streamable HTTP server (`/mcp`)   | [`src/jhe_mcp/server_http.py`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/mcp_server/src/jhe_mcp/server_http.py)                                                                      |
| stdio server (fallback transport) | [`src/jhe_mcp/server_stdio.py`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/mcp_server/src/jhe_mcp/server_stdio.py)                                                                    |
| OAuth broker + flow               | [`src/jhe_mcp/auth/`](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/mcp_server/src/jhe_mcp/auth)                                                                                         |
| Tool definitions                  | [`src/jhe_mcp/tools/`](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/mcp_server/src/jhe_mcp/tools)                                                                                       |
| JHE FHIR/REST client              | [`src/jhe_mcp/fhir/`](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/mcp_server/src/jhe_mcp/fhir)                                                                                         |
| OMH schema registry               | [`src/jhe_mcp/omh_registry.py`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/mcp_server/src/jhe_mcp/omh_registry.py)                                                                    |
| Configuration (env vars)          | [`src/jhe_mcp/config.py`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/mcp_server/src/jhe_mcp/config.py)                                                                                |
| Container / deploy                | [`Dockerfile`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/mcp_server/Dockerfile), [`fly.toml`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/mcp_server/fly.toml) |
| JHE-side reproducible client seed | [`core/management/commands/seed.py`](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/core/management/commands/seed.py) (`seed_mcp_broker_application`)                                     |
| Tests                             | [`tests/`](https://github.com/jupyterhealth/jupyterhealth-exchange/tree/main/mcp_server/tests)                                                                                                               |

> In the examples below, replace `<your-mcp-host>` with the public host of your MCP server. The current reference/dev deployment runs on [Fly.io](https://fly.io) at `https://jhe-mcp.fly.dev`; treat that URL as an example, not a stable public service.

## Architecture

The MCP server sits between an LLM client and JHE. It never holds long-lived credentials for the end user: the user logs in at JHE, JHE issues tokens, and the server forwards those tokens to JHE's REST/FHIR APIs on the user's behalf.

```
LLM Client (e.g. Claude Desktop)
        │  MCP over Streamable HTTP (/mcp)
        ▼
┌─────────────────────────────────────┐
│         JHE MCP Server               │
│  OAuth Broker (Authorization façade) │
│  MCP Tools (studies, patients, obs)  │
└────────────┬─────────────────────────┘
             │  OAuth 2.0 + REST/FHIR API calls
             ▼
┌─────────────────────────────────────┐
│       JupyterHealth Exchange         │
│  Authorization Server (django-oauth- │
│  toolkit + OIDC + PKCE)              │
│  RBAC enforced per user              │
│  Data: studies / patients / obs      │
└─────────────────────────────────────┘
```

**Flow:**

1. The LLM client connects to the MCP server's Streamable HTTP endpoint (`/mcp`) and initiates OAuth.
1. The MCP server redirects the user to **JHE's own login screen**.
1. The user enters **their own** JHE credentials; JHE issues an authorization code to the MCP server.
1. The MCP server exchanges the code for JHE tokens and hands them back to the client. The server is **stateless** - it stores no tokens; the client holds and refreshes them.
1. Subsequent MCP tool calls carry the user's token, which the server forwards to JHE REST/FHIR API requests - RBAC is enforced entirely by JHE.

> **Two distinct OAuth identities - don't conflate them:**
>
> - **MCP server ↔ JHE:** the MCP server is a **confidential** OAuth client of JHE, holding `JHE_CLIENT_ID` + `JHE_CLIENT_SECRET`. These live **only in the server deployment** (e.g. Fly secrets) and are never seen by end users.
> - **LLM client ↔ MCP server:** the LLM client authenticates to the broker as a **public** client - a `client_id` only, **no secret**. The end user's actual identity is established by logging in at JHE with their own credentials.

The server speaks the modern MCP **Streamable HTTP transport** at `/mcp` and implements **OAuth 2.0 Dynamic Client Registration (DCR, RFC 7591)** plus discovery metadata (RFC 9728 / RFC 8414). Clients **connect directly to the URL and register themselves** - no bridge and no manually-issued client ID required.

## Registering the OAuth Client in JHE

The MCP server must be registered as an OAuth 2.0 **confidential** client in the JHE instance it will talk to. There are two ways to do this. In the URLs below, replace `<jhe-host>` with your JHE instance's host.

> **Do not use JHE's portal "Clients" page for this.** That page creates a public client with a fixed `{SITE_URL}/auth/callback` redirect URI and cannot issue a `client_secret`. Use one of the two admin paths below instead.

### Option A - Self-service (any admin/staff user)

Navigate to `https://<jhe-host>/o/applications/register/`, log in with a staff or superuser account, fill in the fields from the table below, and save.

### Option B - Django admin (superuser)

Navigate to `https://<jhe-host>/admin/oauth2_provider/application/add/` and fill in the same fields. The admin form also exposes a **User** field - set it to the admin user creating the record, or leave it blank.

### Field values

| Field                     | Value                                    |
| ------------------------- | ---------------------------------------- |
| Name                      | `JHE MCP Server`                         |
| Client type               | `Confidential`                           |
| Authorization grant type  | `Authorization code`                     |
| Redirect URIs             | `https://<your-mcp-host>/oauth/callback` |
| Algorithm                 | `RSA with SHA-256 (RS256)`               |
| Post logout redirect URIs | *(leave blank)*                          |
| Allowed origins           | *(leave blank)*                          |
| User *(admin form only)*  | the admin user creating it, or blank     |

> **Copy the `client_secret` immediately after saving.** django-oauth-toolkit 3.x hashes the secret on save and never displays it again. If you lose it, you must regenerate a new one.

> **PKCE (S256)** is enforced globally via the `PKCE_REQUIRED` setting in JHE - it is not a per-application field and does not need to be configured here.

### Reproducible JHE-side seeding (optional)

Manual registration above is a one-time setup. To make the broker's JHE OAuth application **reproducible** - so a fresh `python manage.py seed` recreates it (as a confidential client with `skip_authorization=True`, i.e. no consent prompt) instead of needing the admin UI - set the **same** credentials on the **JHE deployment** too:

```bash
# on the JHE deployment
MCP_OAUTH_CLIENT_ID=<client_id>
MCP_OAUTH_CLIENT_SECRET=<client_secret>
# optional: set to match your MCP host's /oauth/callback.
# Defaults to https://jhe-mcp.fly.dev/oauth/callback, so non-reference deployments must set it.
MCP_OAUTH_REDIRECT_URI=https://<your-mcp-host>/oauth/callback
```

`seed.py::seed_mcp_broker_application` reads these and creates/updates the `JHE MCP Server` application; when they are unset (local/CI seeds) it is skipped. They must match the `JHE_CLIENT_ID` / `JHE_CLIENT_SECRET` set on the MCP server (see [Configuration](#configuration)) so the broker can authenticate against the seeded record.

## Configuration

The server is configured entirely via environment variables (or secrets in production).

| Variable                | Required | Purpose                                                                                                                                                                                                                                                                                                                                                                                     |
| ----------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `JHE_BASE_URL`          | Yes      | Base URL of the JHE instance (e.g. `https://jhe.fly.dev`). This is what aims the server at a particular JHE.                                                                                                                                                                                                                                                                                |
| `JHE_CLIENT_ID`         | Yes      | The broker's **confidential** OAuth client ID at JHE (server-side only; never given to end users or LLM clients).                                                                                                                                                                                                                                                                           |
| `JHE_CLIENT_SECRET`     | No       | The broker's confidential client secret at JHE. You do not pick this value - JHE generates it: django-oauth-toolkit pre-fills the **Client secret** field on the registration form (see [Registering the OAuth Client](#registering-the-oauth-client-in-jhe)), and copy it immediately because it is hashed on save and never shown again. Required for a confidential client; recommended. |
| `MCP_RESOURCE_URL`      | Yes      | Public URL of this MCP server (e.g. `https://<your-mcp-host>`). Must be the server's own public URL. **If unset it silently defaults to the reference host `https://jhe-mcp.fly.dev`** rather than erroring - so every non-reference deployment must set it, or its discovery metadata (RFC 9728 / RFC 8414) will advertise the reference host instead of itself.                           |
| `MCP_BROKER_KEY`        | Yes      | Random secret used to encrypt OAuth state and authorization codes. **Must be at least 32 characters** or the server refuses to start. Generate with `python -c "import secrets; print(secrets.token_urlsafe(32))"` (yields ~43 characters).                                                                                                                                                 |
| `MCP_ALLOWED_REDIRECTS` | No       | Comma-separated list of non-loopback redirect URIs to allow (for additional MCP client types).                                                                                                                                                                                                                                                                                              |
| `MCP_HTTP_PORT`         | No       | Port the HTTP server listens on (default: `8401`).                                                                                                                                                                                                                                                                                                                                          |
| `MCP_REQUIRE_AUDIENCE`  | No       | When `true`, reject tokens whose audience can't be confirmed via JHE introspection (fail closed). Default `false` (dev): fall back to userinfo-only validation. Set `true` in production.                                                                                                                                                                                                   |

## Connecting an LLM Client

Clients connect directly to the server URL and register themselves via DCR. On first connect the user is sent to **JHE's own login page**; after they sign in, the client caches and refreshes the JHE-issued token automatically. **The end user supplies nothing but their JHE login** - in every client below the only value you provide is the server URL, `https://<your-mcp-host>/mcp`.

### Claude Code

```bash
claude mcp add --transport http jhe https://<your-mcp-host>/mcp
```

The first tool call opens JHE's login in your browser. Verify with `claude mcp list` / `claude mcp get jhe`.

### Claude Desktop

Add a connector via **Settings → Connectors → Add custom connector**, with URL `https://<your-mcp-host>/mcp`. (Recent desktop builds support remote connectors with OAuth + DCR directly.)

### Google Gemini (Gemini CLI)

In `~/.gemini/settings.json`, use the `httpUrl` field for Streamable HTTP:

```json
{
  "mcpServers": {
    "jhe": {
      "httpUrl": "https://<your-mcp-host>/mcp"
    }
  }
}
```

### ChatGPT (OpenAI)

- **ChatGPT app "Connectors" (developer mode):** add a connector with URL `https://<your-mcp-host>/mcp`. The server advertises the discovery metadata and DCR that ChatGPT expects.
- **Responses / Agents API:** the API does not run the OAuth flow for you - obtain a JHE access token out-of-band and pass it on the `mcp` tool:
  ```json
  {"type": "mcp", "server_label": "jhe", "server_url": "https://<your-mcp-host>/mcp", "authorization": "<JHE access token>", "require_approval": "never"}
  ```

### Fallback: stdio-only clients

For a client that cannot speak remote Streamable HTTP at all, bridge with [`mcp-remote`](https://github.com/geelen/mcp-remote) (it will DCR automatically - no client ID needed):

```bash
npx -y mcp-remote https://<your-mcp-host>/mcp
```

## Deploying

The MCP server ships as a standard **container** (the `Dockerfile` in the `mcp_server/` directory) and runs on any container platform - a plain Docker host, Kubernetes, Cloud Run, ECS, Fly.io, etc. It connects to JHE over the network as an OAuth client (via `JHE_BASE_URL`), so one JHE instance can have zero, one, or several MCP servers pointed at it. Deploying JHE never deploys the MCP server, and JHE is fully functional without it - the only thing absent is LLM/MCP access.

To stand it up against a running JHE instance:

1. **Register the OAuth client** in that JHE - see [Registering the OAuth Client in JHE](#registering-the-oauth-client-in-jhe).
1. **Provide the required environment variables / secrets** via your platform's mechanism - `JHE_BASE_URL`, `JHE_CLIENT_ID`, `JHE_CLIENT_SECRET`, `MCP_RESOURCE_URL`, `MCP_BROKER_KEY` (full list in [Configuration](#configuration)).
1. **Build and run the container**, exposing its HTTP port (`8401`) behind TLS. The image builds reproducibly from the committed `uv.lock`:
   ```bash
   docker build -t jhe-mcp .
   docker run -p 8401:8401 --env-file .env jhe-mcp   # or your platform's run/secret mechanism
   ```
1. **Verify** it is healthy:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' https://<your-mcp-host>/health   # expect 200
   ```

To run against a different JHE, change `JHE_BASE_URL` (and the matching client credentials) and redeploy.

### Reference deployment (Fly.io)

The hosted instance runs on [Fly.io](https://fly.io) (app `jhe-mcp`), using the `fly.toml` in the `mcp_server/` directory. Set the secrets from [Configuration](#configuration), then:

```bash
fly secrets set -a jhe-mcp \
  JHE_CLIENT_ID=<client_id> \
  JHE_CLIENT_SECRET=<client_secret> \
  MCP_BROKER_KEY=$(python -c "import secrets; print(secrets.token_urlsafe(32))") \
  MCP_RESOURCE_URL=https://jhe-mcp.fly.dev

fly deploy -a jhe-mcp    # build from uv.lock + deploy
fly logs   -a jhe-mcp    # tail logs
fly status -a jhe-mcp    # machine / image status
```

## Tools

The MCP server exposes the following tools to LLM clients. Every tool runs as the authenticated user and only returns data that user is authorized to see.

**Studies:**

- **`get_study_count`** - Returns the total number of studies the authenticated user can access.
- **`list_studies`** - Lists all studies visible to the authenticated user, with key metadata.
- **`get_study_metadata`** - Retrieves detailed metadata for a specific study by ID.
- **`list_study_patients`** - Lists patients enrolled in a specific study, returning ID, name, and email for each.

**Patients:**

- **`get_patient_demographics`** - Returns demographic information for a specific patient by patient ID.
- **`get_patient_date_range`** - Returns the earliest and latest observation dates and total count for a patient (first/last-data answers without paging).

**Observations:**

- **`count_patient_observations`** - Returns the exact observation count for a patient, optionally filtered by OMH data type and date range, without returning records.
- **`count_study_observations`** - Returns the observation count across a whole study in one call; with `by_patient=True` returns a `{patient_id: count}` map.
- **`summarize_patient_observations`** - Returns a compact per-data-type digest for a patient (`{type: {count, earliest, latest}}`) - the "show me everything" overview.
- **`get_patient_observations`** - Fetches one page of a patient's observations (with total/pagination), optionally filtered by OMH data type and date range; defaults to compact records, with `verbosity="full"` for the raw OMH body.

**OMH schemas:**

- **`get_omh_schema`** - Returns the full OMH JSON schema for a data type by short name (e.g. `heart-rate`, `blood-glucose`). Schemas are also browsable as resources at `omh://schema/<name>`.

## Local Development

### Setup

```bash
cd mcp_server
uv venv --python 3.12 .venv
source .venv/bin/activate
uv pip install -e ".[dev]"
```

### Running tests

```bash
uv run pytest tests/unit
```

### Running the server

Set the required environment variables (see [Configuration](#configuration)) before starting, or create a `.env` file and load it into your shell, then:

```bash
jhe-mcp-http
```
