---
title: Auth
---

## Authentication & Authorization

The OAuth 2.0 Authorization Code flow with PKCE is used to issue access and refresh tokens, and OIDC is used to issue ID tokens, for both Practitioners (via web login) and Patients (via JHE clients through a secure invitation link).

Endpoints and configuration details can be discovered from the OIDC metadata endpoint:
`/o/.well-known/openid-configuration`

FHIR/SMART clients can equivalently discover the `authorize` and `token` endpoints from the [SMART App Launch discovery document](../reference/exchange-apis.md) at `/fhir/r5/.well-known/smart-configuration` (or the CapabilityStatement at `/fhir/r5/metadata`), so public PKCE clients need no hardcoded endpoint configuration.

Separate OAuth clients are created for the Web UI (Practitioners) and for individual JHE (Patient) Clients.

### JHE Client Auth Flow

> A JHE Client is the application component that communicates directly with JHE to manage consents and upload data.

#### 1. Patient receives Invitation Link

The patient receives a link by E-mail or SMS that looks like this example: `https://app.tcp.org/invitation/jhe.tcp.org_0wYuXvhoyRfko9yFYl9inpBiNkHLVBMy`

- `https://app.tcp.org/` is the URL used to launch the app or web browser
- `jhe.tcp.org_0wYuXvhoyRfko9yFYl9inpBiNkHLVBMy` is the invitation code and it can be configured how/where it's included into the URL

#### 2. Client is launched and consumes the invitation code

The patient clicks the link to launch the client which consumes the invitation code

- `jhe.tcp.org_0wYuXvhoyRfko9yFYl9inpBiNkHLVBMy` is the invitation code
  - The client first splits the invitation code on "`_`"
    - `jhe.tcp.org` is the JHE host (may also include port eg `jhe.tcp.org:8080`)
    - `0wYuXvhoyRfko9yFYl9inpBiNkHLVBMy` is the invitation token

#### 3. Client swaps token for grant authorization code

The client makes a POST request to the following URL:

`https://{JHE host}/api/v1/invitation/{token}`

eg: `POST https://jhe.tcp.org/api/v1/invitation/0wYuXvhoyRfko9yFYl9inpBiNkHLVBMy`

JHE responds with the grant, eg:

```
{
  "grant": {
    "grant_type": "authorization_code",
    "redirect_uri": "https://jhe.tcp.org/auth/callback",
    "client_id": "hxngPvsCo7TR1IgijzqFChfEtZr3Kb3JPEKfM1Rk",
    "code": "aTvkFAndYkObdV5CjrygTlc8YpyE4o"
  },
  "token_endpoint": "http://jhe.tcp.org/o/token/",
  "expires": "2026-05-24T12:21:33.471879Z"
}
```

#### 3. Client adds code_verifier and swaps grant for access token

- The client first computes the code verifier by Base64URL encoding (without padding) on the original invitation token.

`code_verifier = base64.urlsafe_b64encode(invitation_token.encode()).decode().rstrip("=")`

eg: `0wYuXvhoyRfko9yFYl9inpBiNkHLVBMy` has code_verifier `MHdZdVh2aG95UmZrbzl5RllsOWlucEJpTmtITFZCTXk`

- The client constructs a POST body by adding the `code_verifier` to the grant from step 3 above and urlencoding

eg: `grant_type=authorization_code&redirect_uri=http%3A%2F%2Fjhe.tcp.org%2Fauth%2Fcallback&client_id=hxngPvsCo7TR1IgijzqFChfEtZr3Kb3JPEKfM1Rk&code=aTvkFAndYkObdV5CjrygTlc8YpyE4o&code_verifier=MHdZdVh2aG95UmZrbzl5RllsOWlucEJpTmtITFZCTXk`

- The client makes a final POST request to the `token_endpoint` returned in the grant with the `content-type: application/x-www-form-urlencoded` header. JHE responds with the access and id tokens.

```
{
  "access_token": "x3VdsqpjayuOQ08G9EnWyAf7LDUor6",
  "expires_in": 60,
  "token_type": "Bearer",
  "scope": "openid email",
  "refresh_token": "WTtBhOIOTwYh4Z0x5UrvQYT7Hk6xKW",
  "id_token": "eyJ0eXAiOiAiSldUIiwgImFsZyI6ICJSUzI1NiIsICJraWQiOiAiNFZkLVBpU3pPMzVLcjhFaUlxMk1EVzVndkc4X2JNeFplVzFmeTBoVkR6VSJ9.eyJhdWQiOiAiaHhuZ1B2c0NvN1RSMUlnaWp6cUZDaGZFdFpyM0tiM0pQRUtmTTFSayIsICJpYXQiOiAxNzc4NDE1NzM4LCAiYXRfaGFzaCI6ICJsNFBkMU9uSTRDMldBSk9vRVM2SVhnIiwgInN1YiI6ICIxMDAwNyIsICJlbWFpbCI6ICJsbF9wYXRpZW50X3BhbWVsYUBleGFtcGxlLmNvbSIsICJpc3MiOiAiaHR0cDovL2xvY2FsaG9zdDo4MDAwL28iLCAiZXhwIjogMTc3ODQ1MTczOCwgImF1dGhfdGltZSI6IDE3Nzg0MTU2OTMsICJqdGkiOiAiMTQ1MDU4NTAtN2I1Mi00NWZkLTlmMzQtMGUzNDUxZThhYWJjIn0.qeFh61mQrin45RpWzhXw1S0s13sM8TXkBWaWulzP3hcJJd5HC4Z_J-BHqp6G--UwG69bs8Pn7pdhyaaMdFDlft72mwwr_ZPuy7jp78l9nbpeu_2FAjxKPeE0bPyHHs9sdri_65uFyZG_B9PPZcYWsXUJwj-u_UHS58x3Ef-anTVQxCihfiozEteCSdomWkSGEuNfszjFGTXwlo6tsg_LlF6ZKmFkC0uHTVqWQsorBbmnomuZqf-znw7tjpP3xFN5lHqSwy5ds2i_TTue5q__Ls7QOPgLvRFVAcGYzmwql7ZGZDQlXm00nmY5xvD9vB1fubZkyrqfPMfQhRiolXiZcGbHgtq5sOrHnXxAlqq88i9EYRDmupC8zY7U1hyX4Bl-5jOQmP4roxrJvELMzregpDtLxPBAUYGYdhQc1LqMMY2cISXeDfrBh4UjjueF7-XbODuumzrDZpXMDD1YoaO81tUuDDU9ONpNFPO9oGutKnC4l3iWcZAimINyXet0zdFHQa8w_MwpgTGLspW460xrJUxRqt3RqSfIHW_vtYPQjD-zaP9uj_p9uzWBsQZk8gqLjnqncr8dsSRcdoNFvBnJHvVZGDCsXaWQU0ThzDiNhc-kVibQLVXuGHmIL5by4vW73ddswX04KHobuZiJUaoMC8E2qXs_rbm3Id-hCkl1-3Q"
}
```

The returned `access_token` must be included in the `Authorization` header for all subsequent API requests with the prefix `Bearer `

### Single Sign-On (SSO) with SAML2

The [django-allauth SAML provider](https://docs.allauth.org/en/latest/socialaccount/providers/saml.html) is included to support SSO with SAML2.

Users provisioned via SAML are created as Practitioners; existing Patient accounts are rejected from SAML login.

#### System Settings

| Key                      | Value Type | Value                     | Notes                                                            |
| ------------------------ | ---------- | ------------------------- | ---------------------------------------------------------------- |
| `auth.sso.saml2`         | int        | `1`                       | Set to 0 to disable SAML SSO                                     |
| `auth.sso.valid_domains` | string     | `example.com,example.org` | Comma-separated email domains, restricts first-time sign-up only |

#### IdP Configuration

The IdP is configured as a Social Application in the Django admin (`/admin/socialaccount/socialapp/`). Exactly **one** SAML Social Application must exist (along with the `auth.sso.saml2` setting enabled) for the SSO login button to be displayed.

- **Provider**: `saml`
- **Name**: any display name, eg `Mock SAML`
- **Client id**: the URL slug for the provider's endpoints, eg `mocksaml`
- **Settings**: JSON configuring the IdP, eg:

```json
{
  "idp": {
    "metadata_url": "https://mocksaml.com/api/saml/metadata"
  },
  "verified_email": true,
  "email_authentication": true,
  "advanced": {
    "authn_request_signed": true,
    "want_assertion_signed": true,
    "private_key": "<PEM>",
    "x509cert": "<PEM>"
  }
}
```

- `verified_email` trusts e-mail addresses asserted by the IdP as verified
- `email_authentication` allows an existing password account with the same e-mail address to log in via SAML
  - Both are per-IdP settings and should only be enabled for IdPs that verify mailbox ownership
- The `advanced` signing flags default to off in allauth; enable them (with a key pair) for IdPs that require signed requests/assertions
- The IdP must assert a **stable user identifier**: either the `urn:oasis:names:tc:SAML:attribute:subject-id` attribute (allauth's default `uid` mapping) or an `attribute_mapping` entry mapping `uid` to a persistent attribute (eg `eduPersonPrincipalName`). Without one, allauth falls back to the SAML NameID — a *transient* NameID creates a new linked social account on every login. Verify at onboarding: log in twice and confirm the user still has exactly one entry under `/admin/socialaccount/socialaccount/`.

With a Social Application slug of `mocksaml`:

- The ACS URL is `http://localhost:8000/allauth/saml/mocksaml/acs/`
- The SP metadata is served at `http://localhost:8000/allauth/saml/mocksaml/metadata/`

#### Example SAML2 Flow with Mock SAML

##### Test Flow

1. Visit https://mocksaml.com/saml/login and enter the fields below:

   - ACS URL: `http://localhost:8000/allauth/saml/mocksaml/acs/` (or substitute your hostname) **End with a trailing slash**

   - Audience: `http://localhost:8000/allauth/saml/mocksaml/metadata/` (the SP entity ID, which allauth defaults to the SP metadata URL — not the ACS URL; it can be overridden with `sp.entity_id` in the Social Application settings JSON)

1. Enter any email name `@example.com`

1. Enter any password

1. Click Sign in

1. The JHE portal should be displayed with the user in the matching user name in the bottom left hand corner

#### Disabling SAML SSO (back-out)

1. Set the `auth.sso.saml2` System Setting to `0` — this only hides the login button; the `/allauth/saml/<slug>/` endpoints stay live.
1. Delete the SAML Social Application in the Django admin — this is the real kill switch: every SAML endpoint for that slug then returns 404.
1. If backing out permanently, also delete the provider's rows under `/admin/socialaccount/socialaccount/`. They are inert without the Social Application, but a future re-enable would silently re-link users under the old IdP's identifiers.
1. Users who signed up via SSO have no password; if SSO is being retired they can set one via the password reset flow at `/allauth/password/reset/`. (Use that URL — the login page's "Forgot Password" link skips accounts that have no usable password.)
