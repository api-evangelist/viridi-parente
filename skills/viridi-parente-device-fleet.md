---
name: viridi-device-fleet
description: Inventory devices and assets, read latest timeseries, and triage alarms on the Viridi ViSTA platform, without touching physical actuation.
api: Viridi ViSTA Platform API
base_url: https://vista.viridiparente.com
operations:
  - getTenantDevices_1
  - getTenantDeviceInfos
  - getDeviceById
  - getCustomerDevices
  - getTimeseriesKeys
  - getLatestTimeseries_1
  - getAllAlarms
  - clearAlarm
generated: '2026-09-04'
method: generated
source: openapi/viridi-parente-vista-openapi.json
---

# Inventory and monitor a Viridi ViSTA fleet

ViSTA is a Viridi-operated ThingsBoard 3.7.0 deployment, so the fleet model is
Tenant → Customer → Device / Asset, with alarms and timeseries hanging off entities. See
`data-model/viridi-parente-data-model.yml`.

## 1. Authenticate

`POST /api/auth/login`, then `X-Authorization: Bearer <jwt>` on every call.

## 2. List the fleet

- `getTenantDevices_1` — `GET /api/tenant/devices?page=0&pageSize=100`
- `getTenantDeviceInfos` — `GET /api/tenant/deviceInfos?page=0&pageSize=100` when you also
  want the customer/profile names resolved
- `getCustomerDevices` — `GET /api/customer/{customerId}/devices` to scope to one customer

Pagination is `page`, `pageSize`, `sortProperty`, `sortOrder`, `textSearch`. Responses carry
`data`, `totalPages`, `totalElements`, `hasNext` — walk on `hasNext`, do not compute offsets.

## 3. Read one device

`getDeviceById` — `GET /api/device/{deviceId}`. A `Device` carries typed `EntityId` objects
(`{id, entityType}`) for `tenantId`, `customerId`, `deviceProfileId`, `firmwareId` and
`softwareId` — dereference those, never treat them as plain strings.

## 4. Read telemetry

- `getTimeseriesKeys` — `GET /api/plugins/telemetry/{entityType}/{entityId}/keys/timeseries`
  to discover which keys the device actually reports.
- `getLatestTimeseries_1` — `GET /api/plugins/telemetry/{entityType}/{entityId}/values/timeseries?keys=<csv>`
  for the latest values.

Discover keys first. Key names are per-device-profile and are not in the contract.

## 5. Triage alarms

- `getAllAlarms` — `GET /api/alarms?page=0&pageSize=100`
- `clearAlarm` — `POST /api/alarm/{alarmId}/clear` is the reversal for an acknowledged alarm.

Clearing an alarm is the only write this skill permits, and it is reversible in the sense
that the alarm can re-fire; no clearing window is documented.

## 6. Operations this skill must never call

These actuate physical battery, generator and radio hardware. There is no idempotency key
and no documented reversal for any of them, so a retry after a timeout can act twice:

- `handleTwoWayDeviceRPCRequest` — `POST /api/rpc/twoway/{deviceId}`
- `setConfig`, `setCert`, `registerAll`, `deregisterAll` (CBRS radio surface)
- `deleteDevice` — a hard delete with no restore operation
- `deleteEntityTimeseries` — a hard delete of historical data

Escalate any of these to a human. See `conventions/viridi-parente-conventions.yml`
(`idempotency.coverage: none`, `reversibility.grade: documented`).
