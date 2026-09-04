---
name: viridi-moxion-aemp-telematics
description: Page through Moxion AEMP equipment telematics and fault records from the Viridi ViSTA platform.
api: Viridi ViSTA Platform API
base_url: https://vista.viridiparente.com
operations:
  - getMoxionAEMP
  - getMoxionAEMPFaults
generated: '2026-09-04'
method: generated
source: openapi/viridi-parente-vista-openapi.json
---

# Pull Moxion AEMP telematics from Viridi ViSTA

Viridi took over the Moxion mobile power line and exposes its equipment telematics on the
ViSTA platform under an AEMP-named surface — the mixed-fleet telematics convention
(AEMP 2.0 / ISO 15143-3) that fleet-management systems consume.

## 1. Authenticate

`POST /api/auth/login`, then send `X-Authorization: Bearer <jwt>` on every call. See
`viridi-parente-sbr30-telemetry.md` step 1.

## 2. Page the telematics feed

`getMoxionAEMP` — `GET /api/moxion_aemp/{page}?pageSize=100`

- `page` is a required **path** parameter (int32), not a query parameter. This is unusual on
  this API: every other paged surface takes `page` as a query parameter.
- `pageSize` is a query parameter, default 100.
- Start at `page=0` and walk until the returned page is empty.

## 3. Page the fault feed

`getMoxionAEMPFaults` — `GET /api/moxion_aemp_faults/{page}?pageSize=100`

Same shape. Correlate faults against the telematics records by equipment identifier.

## 4. What you cannot assume

The contract declares both responses as a bare `object` with no schema, so the payload is
not confirmed AEMP-conformant from the published contract. Do not hand these records to a
downstream AEMP consumer without inspecting the real shape first, and do not map fields by
guessing from the standard — read one page and confirm.

## 5. Rules

- Read-only. Neither operation writes.
- No rate limits are published and no rate-limit headers are returned. Page serially with
  backoff rather than in parallel.
- On 401 re-authenticate once and retry. On 500 retry with backoff. On 400 fix the request.
