---
title: Auth
---

## Authentication & Authorization

The OAuth 2.0 Authorization Code flow with PKCE is used to issue access and refresh tokens, and OIDC is used to issue ID tokens, for both Practitioners (via web login) and Patients (via JHE clients through a secure invitation link).

Endpoints and configuration details can be discovered from the OIDC metadata endpoint:
`/o/.well-known/openid-configuration`

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

The [django-saml2-auth](https://github.com/grafana/django-saml2-auth) library is included to support SSO with SAML2.

#### Example SAML2 Flow with Mock SAML

Ensure you have the following System Settings

| Key                         | Value Type | Value                                    | Notes                                 |
| --------------------------- | ---------- | ---------------------------------------- | ------------------------------------- |
| `auth.sso.saml2`            | int        | `1`                                      | Set to 0 to disable SAML SSO          |
| `auth.sso.idp_metadata_url` | string     | `https://mocksaml.com/api/saml/metadata` | SAML IdP metadata URL                 |
| `auth.sso.valid_domains`    | string     | `example.com,example.org`                | Comma-separated email domains for SSO |

##### Test Flow

1. Visit https://mocksaml.com/saml/login and enter the fields below:

   - ACS URL: `http://localhost:8000/sso/acs/` (or substitute your hostname) **End with a trailing slash**

   - Audience: `http://localhost:8000/sso/acs/`

1. Enter any email name `@example.com`

1. Enter any password

1. Click Sign in

1. The JHE portal should be displayed with the user in the matching user name in the bottom left hand corner
