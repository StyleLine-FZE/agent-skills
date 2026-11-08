---
name: renewclock
description: Use when an agent needs to list, add, read, renew, or delete Renew Clock renewal items through the public items API. The items API is the Business plan. The shared monitor routes are also mounted.
---

# Renew Clock

Renew Clock tracks dated renewal items such as licenses, permits, insurance, contracts, and similar records. The product API is `/api/v1/items`. The app also mounts the shared `/api/v1/monitors` routes.

Public docs: https://www.renewclock.com/docs/api

## API key

Sign in and open Settings: https://www.renewclock.com/app/settings

Create a key by POSTing `/app/settings/api-keys`. The field `name` is trimmed to 60 characters. An empty name is stored as `API key`. The full key is shown once. An account can hold 10 keys.

Revoke a key by POSTing `/app/settings/api-keys/{id}/delete`.

The key prefix is `rnc_`. Settings does not block key creation by plan. The items routes reject a plan other than Business.

## Authentication

```http
Authorization: Bearer rnc_...
```

Used by both `/api/v1/items` and `/api/v1/monitors`.

A missing bearer token, or a token that does not start with `rnc_`, is HTTP 401:

```json
{"error":"unauthorized","message":"Send Authorization: Bearer <api key>."}
```

An unknown or revoked key is HTTP 401:

```json
{"error":"unauthorized","message":"Unknown or revoked API key."}
```

If the key's user has `team_owner_id`, the call runs as that owner. If that owner row is missing, the response is HTTP 401 `{"error":"unauthorized"}` with no `message`.

## Base URL

`https://www.renewclock.com`

`/api/v1/monitors` responses set `Cache-Control: no-store`. The items routes do not set that header themselves.

## Item object

The items API returns the stored renewal item. `status` is computed when the row is read: `renewed` when archived, `lapsed` when `expiresAt` is before today UTC, otherwise `active`.

```json
{
  "id": "",
  "accountId": "",
  "type": "license",
  "name": "",
  "holder": null,
  "issuer": null,
  "referenceNumber": null,
  "notes": null,
  "issueDate": null,
  "expiresAt": "2027-01-14",
  "termMonths": 12,
  "reminderDays": [90, 60, 30, 14, 7, 1],
  "ownerEmail": "",
  "backupEmail": null,
  "escalateAfterDays": 3,
  "locationId": null,
  "customFields": {},
  "archived": false,
  "status": "active",
  "lastRenewedAt": null,
  "createdAt": "",
  "updatedAt": ""
}
```

Accepted `type` values on create: `license`, `permit`, `insurance`, `contract`, `software`, `vehicle`, `warranty`, `domain`, `certification`, `other`. Any other string is stored as `other`.

Item caps used by create: free 15, starter 150, pro 750, business is `Number.MAX_SAFE_INTEGER`. The items API still returns 402 `forbidden_plan` before that check unless the plan is Business.

## Item endpoints

### GET /api/v1/items

Lists items that are not archived.

```bash
curl -sS -H "Authorization: Bearer rnc_..." https://www.renewclock.com/api/v1/items
```

HTTP 200 `{"data":[<item>...],"plan":"business","limit":0}`.

`limit` is `monitorLimit` for the account (the shared monitor slot limit), not the item cap.

HTTP 402 when the plan is not Business:

```json
{"error":"forbidden_plan","message":"The items API is on Business."}
```

### POST /api/v1/items

JSON fields: `name` (string, min 1), `expiresAt` (`YYYY-MM-DD`), optional `type`, optional `ownerEmail` (email), optional `holder`, optional `issuer`.

Omitted `type` is `other`. Omitted `ownerEmail` is the account email. `termMonths` is stored as 12. `reminderDays` is `[90, 60, 30, 14, 7, 1]`. `escalateAfterDays` is 3. `customFields` is `{}`.

```bash
curl -sS -X POST -H "Authorization: Bearer rnc_..." -H "Content-Type: application/json" \
  -d '{"name":"City license","type":"license","expiresAt":"2027-01-14","ownerEmail":"ops@example.com"}' \
  https://www.renewclock.com/api/v1/items
```

HTTP 201 `{"data": <item>}`.

HTTP 400 `{"error":"invalid_json"}` when the body is not JSON.

HTTP 400 `{"error":"invalid_json","message":"Send name and expiresAt (YYYY-MM-DD)."}` when the object fails the schema.

HTTP 402 `{"error":"plan_limit","message":"This plan has no free item slots."}` when the active count is at the item cap.

HTTP 402 `forbidden_plan` when the plan is not Business.

### GET /api/v1/items/{id}

```bash
curl -sS -H "Authorization: Bearer rnc_..." https://www.renewclock.com/api/v1/items/ITEM_ID
```

HTTP 200 `{"data": <item>}`. HTTP 404 `{"error":"not_found"}`.

### POST /api/v1/items/{id}/renew

The handler does not read a body. It rolls `expiresAt` forward by `termMonths` (12 when that value is not positive), sets `status` to `active`, sets `lastRenewedAt` to now, and sets `archived` to false.

```bash
curl -sS -X POST -H "Authorization: Bearer rnc_..." https://www.renewclock.com/api/v1/items/ITEM_ID/renew
```

HTTP 200 `{"data": <item>}`. HTTP 404 `{"error":"not_found"}`.

### DELETE /api/v1/items/{id}

```bash
curl -sS -X DELETE -H "Authorization: Bearer rnc_..." https://www.renewclock.com/api/v1/items/ITEM_ID
```

HTTP 204 with an empty body, or HTTP 404 `{"error":"not_found"}`.

## Item rate limits

60 requests per minute per key id, window 60 seconds. This counter is separate from the monitor API counter.

After the key is recognized, the response sets `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset`.

Above 60: HTTP 429, `Retry-After: 60`, and `{"error":"rate_limited"}` with no `message`.

## Item errors

| HTTP | `error` | `message` |
| --- | --- | --- |
| 401 | `unauthorized` | Present for a bad prefix and an unknown key. Absent when the team owner is missing. |
| 400 | `invalid_json` | Absent when the body is not JSON. Present as `Send name and expiresAt (YYYY-MM-DD).` when the object fails the schema. |
| 402 | `forbidden_plan` | `The items API is on Business.` |
| 402 | `plan_limit` | `This plan has no free item slots.` |
| 404 | `not_found` | Absent. |
| 429 | `rate_limited` | Absent. `Retry-After` is `60`. |

## Shared monitor routes

These routes are mounted for every app that uses the platform app, including Renew Clock. They store a hostname and port and run a TLS check. They are not the renewal item API.

| Method | Path | Result |
| --- | --- | --- |
| GET | `/api/v1/monitors` | `{"data","plan","limit","planLimit","bonusLimit"}` |
| POST | `/api/v1/monitors` | JSON `hostname` and optional `port`. HTTP 201, 400, 402, or 409. |
| GET | `/api/v1/monitors/{id}` | One monitor. |
| GET | `/api/v1/monitors/{id}/checks` | Last 50 rows: `checked_at`, `ok`, `error_code`, `days_remaining`, `issuer`, `valid_to`. |
| POST | `/api/v1/monitors/{id}/check` | Runs a TLS check. No body. |
| DELETE | `/api/v1/monitors/{id}` | HTTP 204 or 404. |

The monitor object fields are `id`, `hostname`, `port`, `paused`, `status` (`pending`, `ok`, or `problem`), `error_code`, `days_remaining`, `valid_from`, `valid_to`, `issuer`, `subject`, `chain_valid`, `hostname_match`, `last_checked_at`, `domain`, `domain_expires_at`, `created_at`.

List `paused` follows the plan pause set. GET one and POST check return `paused: false`.

Monitor rate limit is also 60 a minute per key, with `Retry-After` set to the seconds left in the window and `message` `Limit is 60 requests per minute per key.` Duplicate message is `That hostname and port are already on the account.` Plan limit message is `The <plan> plan covers <n> monitors.` Invalid hostname message is `Send {"hostname": "example.com"}, optionally with "port".`

```bash
curl -sS -H "Authorization: Bearer rnc_..." https://www.renewclock.com/api/v1/monitors
```

## Signed webhooks

A webhook delivery POSTs JSON and sets `X-RenewClock-Event` and `X-RenewClock-Signature: t=<unix seconds>,v1=<hex>`.

`v1` is HMAC-SHA256, hex, of `timestamp + "." + raw body`, using the monitor signing secret.

Default body fields: `event`, `id`, `sent_at`, `monitor` (`id`, `hostname`, `port`, `url`), `subject`, `text`, `data`.
