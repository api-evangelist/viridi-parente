---
name: viridi-sbr30-telemetry
description: Read SBR30 battery energy storage data and bulk telemetry from the Viridi ViSTA platform, including generator and chiller latest values.
api: Viridi ViSTA Platform API
base_url: https://vista.viridiparente.com
operations:
  - getSBR30Data
  - getSBR30DataV1_2
  - getSBR30Telem
  - getGeneratorData
  - getChillerData
generated: '2026-09-04'
method: generated
source: openapi/viridi-parente-vista-openapi.json
---

# Read SBR30 battery telemetry from Viridi ViSTA

All operations require an authenticated ViSTA account. There is no sandbox and no
self-service signup — credentials are issued by Viridi sales at
https://viridiparente.com/create-a-vista-account/.

## 1. Authenticate

`POST /api/auth/login` with `{"username": ..., "password": ...}`. Take the returned JWT and
send it on every subsequent call as:

```
X-Authorization: Bearer <jwt>
```

Refresh with `POST /api/auth/token` before expiry. A 401 with
`{"status":401,"message":"Authentication failed","errorCode":10}` means the token is
missing, invalid or expired — re-authenticate, then retry once.

If the tenant uses single sign-on, `POST /api/noauth/oauth2Clients` (anonymous) lists the
configured identity providers and their `/oauth2/authorization/<uuid>` entry URLs.

## 2. Read the latest SBR30 page

- `getSBR30Data` — `GET /api/data?page=0&pageSize=10`
- `getSBR30DataV1_2` — `GET /api/data/v1_2?page=0&pageSize=10`

Both take `page` (default 0) and `pageSize` (default 10). Pick the v1_2 variant when the
caller needs the newer payload; the contract does not say which is preferred and neither is
marked deprecated, so pin the one you tested against.

The response body is typed as a bare `object` in the contract — Viridi publishes no schema
for it. Do not assume field names; read the first page and inspect before mapping.

## 3. Read a time range in bulk

`getSBR30Telem` — `POST /api/telem` with a `TelemRequest` body:

```json
{
  "assets": ["<asset name>"],
  "keys": {"<group>": ["<key>", "<key>"]},
  "start_ts": 1780000000000,
  "end_ts": 1780086400000,
  "report": false,
  "limit": 1000,
  "interval": "1h"
}
```

`start_ts` / `end_ts` are epoch milliseconds. The response is a byte payload, not JSON —
handle it as a stream. Keep ranges bounded: no rate limits are published and no rate-limit
headers are returned, so there is no runtime signal to back off on. Use conservative
concurrency and your own backoff.

## 4. Generator and chiller values

- `getGeneratorData` — `GET /api/data/gen`
- `getChillerData` — `GET /api/data/chiller`
- `getGenTelem` — `POST /api/genTelem` for a bulk generator read

## 5. Rules

- This skill is read-only. It calls no write operation.
- Errors are NOT RFC 9457. The envelope is `{status, message, errorCode, timestamp}` on
  `application/json`. See `errors/viridi-parente-problem-types.yml`.
- A 500 is safe to retry with backoff. A 400 is not — fix the payload.
- There is no idempotency key anywhere in this API, so never assume a retry is free on a
  write path. Reads are safe.
