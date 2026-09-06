---
name: diebold-authenticate-transaction-middleware
description: Obtain and refresh a bearer token for a Diebold Nixdorf Vynamic Transaction Middleware installation, and retrieve the public key needed for JWE-encrypted bearer tokens.
api: DN TM Authorization API / DN Open Backend API
operations:
  - tm-authenticate
  - exchangeToken
  - refreshToken
  - tm-get-keys
  - tm-logout
  - getToken
  - exchangeKey
  - getRsaKey
  - getEcKey
generated: '2026-09-06'
method: generated
source: openapi/diebold-dn-tm-authorization-api-openapi.yml, openapi/diebold-dn-open-backend-api-openapi.yml, openapi/diebold-dn-online-mobile-api-openapi.yml
---

# Authenticate against a Vynamic Transaction Middleware installation

Every other Diebold Nixdorf skill in this repository depends on this one. Do it first.

**Before you start.** There is no Diebold Nixdorf hosted endpoint. The base URL is the financial
institution's own Vynamic installation — the published specifications name `localhost:8080` because
the software runs inside the bank. Get the real host, the path prefix (`/oauth-api/v1`,
`/OB-API-REST/v2`, `/tm-om-api/v4`) and the credentials from the institution before writing anything.

## Steps

1. **Pick the token source for your surface.**
   - Transaction Middleware generally: `POST /tm-authorization/authenticate` (`tm-authenticate`) on
     the DN TM Authorization API.
   - DN Open Backend API on its own: `GET /getToken` (`getToken`).
   - Payment Initiation or Secure Business Processing: do not use either. Those declare OpenID
     Connect and expect a token from the institution's configured OIDC provider — for Payment
     Initiation, the Microsoft Entra ID tenant named in the spec's `openIdConnect` scheme, whose
     discovery document is saved at `well-known/diebold-openid-configuration.json`.

2. **Authenticate.** Send the credentials with HTTP Basic (`basicAuth` / `basic`). The specification
   says plainly that Basic "should only be used for testing purposes during development time" — for
   anything else, exchange it immediately for a bearer token and stop sending the password.

3. **Read the response.** You get a token plus, on `tm-authenticate`, a `TmAuthenticationResponse`
   carrying `TmRight` entries. Those rights are what the middleware will enforce; check them before
   attempting an operation rather than discovering a 403 mid-transaction.

4. **Refresh rather than re-authenticate.** `POST /token` (`refreshToken`) takes a
   `RefreshTokenRequest`; `GET /token` (`exchangeToken`) exchanges a token from an external provider
   for a middleware token.

5. **Set up JWE if the installation requires it.** The Open Backend and Assist APIs accept a third
   scheme, `jweBearer`: a JWT encrypted with the backend endpoint's public key. Fetch the key first —
   `GET /getRsaKey` or `GET /getEcKey` on the Online & Mobile API, `POST /exchangeKey` on the Open
   Backend API, `GET /pkiKey` on the Payment Initiation API — then encrypt and send the result as
   `Authorization: Bearer <encryptedJWT>`.

6. **Log out when you are done.** `DELETE /tm-authorization/logout` (`tm-logout`) or `POST /logout`
   on the Open Backend API. Sessions on a self-service channel are short-lived by design.

## Rules that apply to everything downstream

- Read `responseCode` (`OK` | `FAIL` | `RESUBMIT`) as well as the HTTP status. `RESUBMIT` is an
  instruction from the core banking host to send the request again; the contract gives no backoff
  guidance, so pick a conservative one.
- On failure, read `extendedResponseCode` — `errors/diebold-decline-codes.yml` lists all 40 values
  and what each one means.
- `GET /test` (`sandboxTest`) on the TM Authorization API verifies that your token carries the
  rights an annotation requires. It is the only test affordance in the contract, and it tests
  authorization, not business logic.
