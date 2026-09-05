---
name: XRHealth server-side patient login
description: >-
  Sign a patient in to XRHealth from a confidential server-side application using the passwordless
  one-time email code flow, then keep the session alive with refresh-token rotation and end it with
  an explicit revoke.
api: openapi/xrhealth-platform-openapi.yml
base_url: https://api.xr.health/v1
operations:
  - startPatientPasswordlessLogin
  - verifyPatientPasswordlessLogin
  - getCurrentPatient
  - refreshPatientToken
  - revokePatientToken
generated: '2026-09-04'
method: generated
source: >-
  Grounded in openapi/xrhealth-platform-openapi.yml, harvested verbatim from
  https://api.xr.health/v1/openapi.json on 2026-09-04. Every operationId above appears in that
  document.
---

# XRHealth server-side patient login

Use this flow when your code runs on a server and can hold a secret. You need an
**XRHealth application token**, issued to you by XRHealth — access to the XRH Developer portal at
<https://developer.xr.health/> is by invitation, so obtain the token from XRHealth before starting.

Send it on **every** call in this skill as the header:

```
X-XRHealth-Application-Token: <your application token>
```

## Steps

1. **Request a one-time code — `startPatientPasswordlessLogin`**
   `POST /auth/passwordless/start` with `{"email": "<patient email>"}`.
   The body schema is `PasswordlessStartRequest`; `email` is the only permitted field and
   `additionalProperties` is `false`, so any extra key is a `400`.
   A success is **`202`**, not `200` — the code was accepted for sending, nothing more.

   **Do not retry this call blindly.** There is no idempotency key anywhere in this API, so a retry
   sends the patient a *second* email. If you get a `429`, back off; XRHealth publishes no
   `RateLimit-*` or `Retry-After` headers, so choose your own backoff.

2. **Exchange the code — `verifyPatientPasswordlessLogin`**
   `POST /auth/passwordless/verify` with `PasswordlessVerifyRequest`: `request_id` (uuid),
   `email`, and `code` (4–8 digits, pattern `^\d{4,8}$`). All three are required.
   You get a `TokenResponse`: `token_type` (`Bearer`), `access_token`, `expires_in`,
   `refresh_token`, `refresh_token_expires_in`, `subject` (the opaque patient id) and `scope`
   (a space-delimited string).

   Store `subject`, never the email, as your patient key — it is the identifier XRHealth intends
   you to hold.

3. **Confirm who you are acting as — `getCurrentPatient`**
   `GET /me` with **both** credentials: the application token header *and*
   `Authorization: Bearer <access_token>`. The `MeResponse` returns `subject`, `application` and
   `scopes[]`. Check `scopes[]` before attempting anything privileged — a `403` from this API means
   the credential is valid but not scoped for the resource.

4. **Keep the session alive — `refreshPatientToken`**
   `POST /auth/token/refresh` with `{"refresh_token": "<token>"}` (min 16 chars).
   Refresh tokens **rotate**: the response is a fresh `TokenResponse` and the token you sent should
   be treated as spent. Persist the new one atomically; if you lose it, the patient must log in
   again from step 1.

5. **End the session — `revokePatientToken`**
   `POST /auth/token/revoke` with `{"refresh_token": "<token>"}`. A `200` means the refresh token is
   no longer usable.

## Rules this API enforces

- **Errors are coarse.** The body is `{"error": "request_error", "request_id": "<uuid>"}` — *not*
  RFC 9457 problem+json — and the same `error` code was observed on both a `401` and a `404`.
  Branch on the HTTP status, not the body.
- **Always log `request_id`.** It is on every response as the `x-request-id` header and repeated in
  error bodies. It is the only handle XRHealth support can act on.
- **Nothing here is idempotent.** No `Idempotency-Key` exists. Steps 1 and 2 have real side effects
  on retry.
- **Revocation is terminal.** There is no un-revoke. Recovery is a new login from step 1.
- **`503` means the auth module, not the whole platform.** Retry with backoff.
