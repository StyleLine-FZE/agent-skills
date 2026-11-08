---
name: firecertclock
description: Use when an agent needs to manage FireCert Clock sites and assets, log a service visit, or download the asset CSV through the public REST API. API access is the Business plan. The shared monitor routes are also mounted.
---

# FireCert Clock

FireCert Clock tracks fire-safety sites and assets and the next due date for each asset. The product API is under `/api/v1/sites`, `/api/v1/assets`, and `/api/v1/export.csv`. The app also mounts the shared `/api/v1/monitors` routes.

Public docs: https://www.firecertclock.com/docs/api

## API key

Sign in and open Settings: https://www.firecertclock.com/app/settings

Create a key by POSTing `/app/settings/api-keys`. The field `name` is trimmed to 60 characters. An empty name is stored as `API key`. The full key is shown once. An account can hold 10 keys.

Revoke a key by POSTing `/app/settings/api-keys/{id}/delete`.

The key prefix is `fcc_`. Settings does not add a FireCert plan gate of its own. The product routes return HTTP 402 unless the plan is Business (`apiAndCsv` is true only on Business).

## Authentication

```http
Authorization: Bearer fcc_...
```

A missing bearer token, or a token that does not start with `fcc_`, is HTTP 401:

```json
{"error":"unauthorized","message":"Send Authorization: Bearer <api key>."}
```

An unknown or revoked key is HTTP 401:

```json
{"error":"unauthorized","message":"Unknown or revoked API key."}
```

If the key's user has `team_owner_id`, the call runs as that owner. If that owner row is missing, the response is HTTP 401 `{"error":"unauthorized"}` with no `message`.

When the plan is not Business, the product routes return HTTP 402 before the handler:

```json
{"error":"plan_required","message":"API access starts on Business."}
```

## Base URL

`https://www.firecertclock.com`

`/api/v1/monitors` responses set `Cache-Control: no-store`. The site and asset routes do not set that header themselves.

## Site object

```json
{
  "id": "",
  "name": "",
  "address": "",
  "client_email": null,
  "asset_count": 0,
  "overdue_count": 0,
  "created_at": ""
}
```

`overdue_count` is the number of that site's assets whose `next_due_on` is before today's UTC date (`YYYY-MM-DD`). `client_name` can be sent on create and is not included in this object.

## Asset object

```json
{
  "id": "",
  "site_id": "",
  "name": "",
  "kind": "extinguisher",
  "location": "",
  "preset_id": "",
  "interval_days": 365,
  "last_service_on": "",
  "next_due_on": "",
  "created_at": ""
}
```

`kind` is `extinguisher`, `fire_alarm`, `sprinkler`, `emergency_light`, or `custom`.

## Endpoints

### GET /api/v1/sites

```bash
curl -sS -H "Authorization: Bearer fcc_..." https://www.firecertclock.com/api/v1/sites
```

HTTP 200 `{"data":[<site>...]}`.

### POST /api/v1/sites

JSON: `name` (1 to 120 characters), optional `address` (max 200), optional `client_email` (email or `""`), optional `client_name` (max 120).

Jurisdiction is stored as `generic`.

```bash
curl -sS -X POST -H "Authorization: Bearer fcc_..." -H "Content-Type: application/json" \
  -d '{"name":"Warehouse A","address":"12 Dock Rd"}' \
  https://www.firecertclock.com/api/v1/sites
```

HTTP 201 `{"data": <site>}` with `asset_count` 0 and `overdue_count` 0. If the row cannot be reloaded, `data` is `{"id":"..."}`.

HTTP 400 `{"error":"invalid_json","message":"Send {\"name\": \"Warehouse A\"}."}` when the body is missing, is not JSON, or fails the schema.

HTTP 402 `{"error":"plan_limit","message":"The current plan does not cover another site."}`.

HTTP 409 `{"error":"duplicate","message":"That site name is already on the account."}`.

### GET /api/v1/sites/{id}

```bash
curl -sS -H "Authorization: Bearer fcc_..." https://www.firecertclock.com/api/v1/sites/SITE_ID
```

HTTP 200 `{"data": { ...site, "assets": [<asset>...] }}`. HTTP 404 `{"error":"not_found"}`.

### DELETE /api/v1/sites/{id}

Removes the site and its assets.

```bash
curl -sS -X DELETE -H "Authorization: Bearer fcc_..." https://www.firecertclock.com/api/v1/sites/SITE_ID
```

HTTP 204 with an empty body, or HTTP 404 `{"error":"not_found"}`.

### GET /api/v1/assets

Optional query `status=overdue` keeps assets whose `next_due_on` is before today UTC. Any other `status` value returns every asset.

```bash
curl -sS -H "Authorization: Bearer fcc_..." "https://www.firecertclock.com/api/v1/assets?status=overdue"
```

HTTP 200 `{"data":[<asset>...]}`.

### POST /api/v1/assets

JSON: `site_id` (min 1), `kind` (one of the five kinds), `last_service_on` (non-empty string). Optional: `name` (max 120), `location` (max 120), `preset_id`, `interval_days` (positive integer, max 5000), `technician_id`.

When `interval_days` is sent, `preset_id` is stored as `custom-interval` and the interval is the sent number. Otherwise a `preset_id` that matches a known preset is kept, and an unknown `preset_id` is replaced by the default preset for that kind.

Omitted `name` becomes the kind label.

```bash
curl -sS -X POST -H "Authorization: Bearer fcc_..." -H "Content-Type: application/json" \
  -d '{"site_id":"SITE_ID","kind":"extinguisher","location":"East stair","last_service_on":"2026-01-15","preset_id":"nfpa10-annual-maintenance"}' \
  https://www.firecertclock.com/api/v1/assets
```

HTTP 201 `{"data": <asset>}` or `{"data":{"id":"..."}}`.

HTTP 400 `{"error":"invalid_json","message":"Send site_id, kind and last_service_on."}` when the body fails the schema.

HTTP 402 `{"error":"plan_limit","message":"The current plan does not cover another asset."}`.

HTTP 404 `{"error":"not_found"}` when the site id is not on the account.

HTTP 400 `{"error":"invalid_json"}` with no `message` for the other create failures.

### POST /api/v1/assets/{id}/visits

JSON: `service_on` must be a real `YYYY-MM-DD` date (the calendar day must match the UTC date). Optional `next_due_on` is used only when it is also a real `YYYY-MM-DD` date. Otherwise the next due date is `service_on` plus the asset `interval_days`.

```bash
curl -sS -X POST -H "Authorization: Bearer fcc_..." -H "Content-Type: application/json" \
  -d '{"service_on":"2026-10-01"}' \
  https://www.firecertclock.com/api/v1/assets/ASSET_ID/visits
```

HTTP 200 `{"data": <asset>}`.

HTTP 400 `{"error":"invalid_json","message":"Send {\"service_on\":\"YYYY-MM-DD\"}."}`.

HTTP 404 `{"error":"not_found"}` when the asset is not on the account.

### GET /api/v1/export.csv

```bash
curl -sS -H "Authorization: Bearer fcc_..." https://www.firecertclock.com/api/v1/export.csv
```

HTTP 200, `content-type: text/csv; charset=utf-8`, `content-disposition: attachment; filename="assets.csv"`.

Header row: `site,name,asset_type,location,last_service_date,next_due_date,status,preset`.

## Product rate limits

60 requests per minute per key id, window 60 seconds, on the site, asset, visit, and CSV routes. This counter is separate from the monitor API counter.

After the key is recognized, the response sets `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset`.

Above 60: HTTP 429, `Retry-After` seconds until the window ends, and:

```json
{"error":"rate_limited","message":"Limit is 60 requests per minute per key."}
```

The plan check runs after the rate-limit check.

## Product errors

| HTTP | `error` | Notes |
| --- | --- | --- |
| 401 | `unauthorized` | Prefix or unknown key includes `message`. Missing team owner has no `message`. |
| 402 | `plan_required` | `API access starts on Business.` |
| 400 | `invalid_json` | `message` depends on the route, as listed above. |
| 402 | `plan_limit` | Site or asset cap message above. |
| 409 | `duplicate` | Site name already exists. |
| 404 | `not_found` | No `message`. |
| 429 | `rate_limited` | Includes the limit `message`. |

## Shared monitor routes

The platform also mounts `/api/v1/monitors`, `/api/v1/monitors/{id}`, `/api/v1/monitors/{id}/checks`, `/api/v1/monitors/{id}/check`, and `DELETE /api/v1/monitors/{id}`.

Those routes use the same `fcc_` key and the shared monitor JSON (`id`, `hostname`, `port`, `paused`, `status`, `error_code`, `days_remaining`, `valid_from`, `valid_to`, `issuer`, `subject`, `chain_valid`, `hostname_match`, `last_checked_at`, `domain`, `domain_expires_at`, `created_at`). POST check runs a TLS check. They do not create sites or assets.

Their rate limit is also 60 a minute per key, with `message` `Limit is 60 requests per minute per key.`

## Signed webhooks

A webhook delivery POSTs JSON and sets `X-FireCertClock-Event` and `X-FireCertClock-Signature: t=<unix seconds>,v1=<hex>`.

`v1` is HMAC-SHA256, hex, of `timestamp + "." + raw body`, using the monitor signing secret.

Default body fields: `event`, `id`, `sent_at`, `monitor` (`id`, `hostname`, `port`, `url`), `subject`, `text`, `data`.
