---
name: infoworks-authenticate
description: Mint, use, validate and revoke an Infoworks REST API v3 bearer token against a customer-deployed Infoworks instance.
api: Infoworks REST API v3
base_url: "{protocol}://{host}:{port}/v3"
operations:
  - GET /security/authenticate
  - GET /security/token/access
  - GET /security/token/validate
  - DELETE /security/token/access
  - DELETE /security/token/refresh
generated: '2026-08-23'
method: generated
source: https://docs.infoworks.io/developer-resources/rest-api + openapi/infoworks-rest-api-v3-openapi.yml
---

# Authenticate to Infoworks

Infoworks runs in the customer's own cloud. Before anything else you need the deployment's host and
port — there is no vendor endpoint to fall back on. The Python SDK's own example uses
`protocol=https`, `port=443` for an ingress-fronted deployment; the OpenAPI defaults
(`http`/`localhost`/`3001`) are the in-cluster service address.

## Step 1 — get a bearer token

Prefer the refresh token. It is the only path that works on every deployment, including SAML.

1. Ask the operator for the refresh token from the Infoworks UI: **My Profile > Settings > Refresh Token**.
2. `GET /v3/security/token/access` with header `Authorization: Basic <refresh_token>`.
3. Read the JWT from `result.authentication_token`.

Username/password is the fallback and **fails on SAML deployments**:

1. Base64-encode `<username>:<password>`.
2. `GET /v3/security/authenticate` with header `Authorization: Basic <encoded>`.
3. Read the JWT from `result.authentication_token`.

## Step 2 — use it

Send `Authorization: Bearer <token>` on every other call. Every non-`/security` operation in the
contract declares `BearerAuth`.

## Step 3 — handle expiry

The token lives **15 minutes by default** (the deployment can configure it). Any long-running task
must re-mint, not retry.

- `IW10004` on 401/403 means the token is missing, expired, or the user lacks access. Re-mint once,
  then retry the original call once. Do not loop.
- `IW10005` on 401 means the credential itself did not resolve to a principal — re-minting will not
  help; the credential is wrong.
- A **406 Not Acceptable** with `iw_code` `IW10031` is how this API reports bad credentials on
  `/security/authenticate`. If you only branch on 401 you will miss it.
- `GET /v3/security/token/validate` tells you whether a token is still active without spending a call
  on a real operation.

## Step 4 — clean up

- `DELETE /v3/security/token/access` (`deleteAuthToken`) purges the access token when you are done.
- `DELETE /v3/security/token/refresh` regenerates the refresh token and blacklists the old one — use
  this if a refresh token may have leaked. It invalidates every client using that token.

## Cautions

- Never log the token. Release 6.1.3.1 exists partly because the platform itself was logging a
  password field (IPD-28155).
- Since 6.2.0 refresh tokens have an enforced expiry with administrator configuration. A refresh token
  that worked last quarter can be expired rather than wrong.
- Error `help` links returned by this API point at `api.infoworks.io`, a host that no longer resolves.
  Do not follow them; use `iw_code` and the support desk instead.
