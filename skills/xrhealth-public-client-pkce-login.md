---
name: XRHealth public-client PKCE login
description: >-
  Sign a patient in from a native or browser client that cannot hold a secret, using XRHealth's
  registered-client passwordless flow with a PKCE S256 challenge and a short-lived authorization
  code.
api: openapi/xrhealth-platform-openapi.yml
base_url: https://api.xr.health/v1
operations:
  - startPublicPatientPasswordlessLogin
  - verifyPublicPatientPasswordlessLogin
  - exchangePublicPatientToken
  - revokePublicPatientToken
  - getPatientApiJwks
generated: '2026-09-04'
method: generated
source: >-
  Grounded in openapi/xrhealth-platform-openapi.yml, harvested verbatim from
  https://api.xr.health/v1/openapi.json on 2026-09-04. Every operationId above appears in that
  document.
---

# XRHealth public-client PKCE login

Use this flow from a mobile app, a headset app, or a single-page web app — anywhere you **cannot**
safely store the `X-XRHealth-Application-Token`. None of the operations in this skill takes that
header; your identity is the registered `client_id` plus proof of possession via PKCE.

You need a `client_id` (8–128 characters) registered with XRHealth.

## Steps

1. **Generate the PKCE pair (locally, before any call).**
   Create a high-entropy `code_verifier` of 43–128 characters, then
   `code_challenge = BASE64URL(SHA256(code_verifier))`.
   XRHealth accepts **only** `code_challenge_method: "S256"` — the schema enum has one member.
   Keep the verifier in memory; it never leaves the device until step 3.

2. **Request a one-time code — `startPublicPatientPasswordlessLogin`**
   `POST /auth/public/passwordless/start` with `PublicPasswordlessStartRequest`:
   `client_id`, `email`, `code_challenge`, `code_challenge_method`. All four are required and
   `additionalProperties` is `false`.
   Success is **`202`**. This emails the patient a code — it is a real side effect, it has no
   idempotency key, and it is one of only two operations in the whole API that declare a `429`.

3. **Exchange the emailed code for an authorization code — `verifyPublicPatientPasswordlessLogin`**
   `POST /auth/public/passwordless/verify` with `PublicPasswordlessVerifyRequest`:
   `client_id`, `request_id`, `email`, `code`.
   You get an `AuthorizationCodeResponse`: `authorization_code` and `expires_in`. The published
   example for `expires_in` is **60 seconds** — treat this code as short-lived and single-use, and
   move straight to step 4.

4. **Exchange the authorization code for tokens — `exchangePublicPatientToken`**
   `POST /auth/public/token` with `PublicTokenRequest`:
   `client_id`, `grant_type: "authorization_code"`, `authorization_code`, `code_verifier`.
   You get a `TokenResponse`.

   To rotate later, call the **same** operation with `grant_type: "refresh_token"` and
   `refresh_token`. There is no separate refresh operation for public clients —
   `refreshPatientToken` requires the application token you do not have.

5. **Verify the access token — `getPatientApiJwks`**
   `GET /.well-known/jwks.json` (relative to the `/v1` base, i.e.
   `https://api.xr.health/v1/.well-known/jwks.json`) returns the JWKS. Cache the keys and verify the
   JWT signature before trusting any claim. Probed 2026-09-04: `200`, RSA keys.

6. **Sign out — `revokePublicPatientToken`**
   `POST /auth/public/token/revoke` with `{"client_id": ..., "refresh_token": ...}`.

## Rules this API enforces

- **`grant_type` has exactly two members**: `authorization_code` and `refresh_token`. Anything else
  is a `400`.
- **Do not retry step 4.** The authorization code is single-use; a retry is a failure, not a no-op.
- **Log `x-request-id`** from every response — it is CORS-exposed via
  `access-control-expose-headers`, so browser clients can read it too.
- **`403` is a scope problem, not a credential problem.** Call `getCurrentPatient` (server side) or
  read `scope` on the `TokenResponse` to see what you actually hold.
- **Revocation is terminal**; recovery is a new login from step 2.
