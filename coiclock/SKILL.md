---
name: coiclock
description: Use when an agent needs to list, add, read, recheck, or delete COI Clock monitors through the public REST API. A monitor stores a hostname. Vendor name and email are collected in the dashboard. Recheck writes certificate compliance onto the monitor row.
---

# COI Clock

COI Clock tracks vendor certificates of insurance. The public API uses the shared monitor routes. POST creates a hostname monitor. It does not accept a vendor name or contact email. Those fields are on the dashboard vendor form. POST `/api/v1/monitors/{id}/check` runs the vendor evaluation and then returns the monitor object.

Public docs: https://www.coiclock.com/docs/api

## API key

Sign in and open Settings: https://www.coiclock.com/app/settings

Create a key by POSTing `/app/settings/api-keys`. The field `name` is trimmed to 60 characters. An empty name is stored as `API key`. The full key is shown once. An account can hold 10 keys.

Revoke a key by POSTing `/app/settings/api-keys/{id}/delete`.

The key prefix is `coi_`. The handler has no plan gate for creating keys or calling `/api/v1`.

## Authentication

```http
Authorization: Bearer coi_...
```

A missing bearer token, or a token that does not start with `coi_`, is HTTP 401:

```json
{"error":"unauthorized","message":"Send Authorization: Bearer <api key>."}
```

An unknown or revoked key is HTTP 401:

```json
{"error":"unauthorized","message":"Unknown or revoked API key."}
```

If the key's user has `team_owner_id`, the call runs as that owner. If that owner row is missing, the response is HTTP 401 `{"error":"unauthorized"}` with no `message`.

## Base URL

`https://www.coiclock.com`

Every `/api/v1/*` response sets `Cache-Control: no-store`.

## Monitor object

```json
{
  "id": "",
  "hostname": "",
  "port": 443,
  "paused": false,
  "status": "pending",
  "error_code": null,
  "days_remaining": null,
  "valid_from": null,
  "valid_to": null,
  "issuer": null,
  "subject": null,
  "chain_valid": null,
  "hostname_match": null,
  "last_checked_at": null,
  "domain": null,
  "domain_expires_at": null,
  "created_at": ""
}
```

`status` is `pending` when `last_checked_at` is null, `problem` when `error_code` is set, and `ok` otherwise.

A saved certificate evaluation sets `issuer` to `coi`, `subject` to the vendor legal name, `days_remaining` to days until the earliest policy expiry, `valid_to` to that expiry date, and `valid_from` to the first policy effective date. `error_code` is null on that result, so `status` stays `ok` after a check even when the check row's `ok` flag is false. `ok` on a check row is true when the vendor status is `compliant` or `expiring`.

Gap codes are stored on the check and are not fields of the monitor object or the check JSON below.

On the list, `paused` follows the plan pause set. GET one and POST check return `paused: false`.

## Endpoints

### GET /api/v1/monitors

```bash
curl -sS -H "Authorization: Bearer coi_..." https://www.coiclock.com/api/v1/monitors
```

HTTP 200 `{"data":[],"plan":"free","limit":0,"planLimit":0,"bonusLimit":0}`.

`limit` is `monitorLimit` for the account. `planLimit` is `planLimit(plan, "monitors").total`. `bonusLimit` is `planLimit(user, "monitors").bonus`.

### POST /api/v1/monitors

JSON `hostname` and optional `port`, parsed as a public hostname, `host:port`, or URL. The path is dropped. Omitted port is 443.

```bash
curl -sS -X POST -H "Authorization: Bearer coi_..." -H "Content-Type: application/json" \
  -d '{"hostname":"acme.example","port":443}' \
  https://www.coiclock.com/api/v1/monitors
```

HTTP 201 `{"data": <monitor>}`.

HTTP 400 `{"error":"invalid_json"}`.

HTTP 400 `{"error":"invalid_hostname","message":"Send {\"hostname\": \"example.com\"}, optionally with \"port\"."}`.

HTTP 402 `{"error":"plan_limit","message":"The <plan> plan covers <n> monitors."}`.

HTTP 409 `{"error":"duplicate","message":"That hostname and port are already on the account."}`.

### GET /api/v1/monitors/{id}

```bash
curl -sS -H "Authorization: Bearer coi_..." https://www.coiclock.com/api/v1/monitors/MONITOR_ID
```

HTTP 200 `{"data": <monitor>}`. HTTP 404 `{"error":"not_found"}`.

### GET /api/v1/monitors/{id}/checks

The last 50 checks.

```bash
curl -sS -H "Authorization: Bearer coi_..." https://www.coiclock.com/api/v1/monitors/MONITOR_ID/checks
```

Each row: `checked_at`, `ok`, `error_code`, `days_remaining`, `issuer`, `valid_to`.

### POST /api/v1/monitors/{id}/check

Runs vendor evaluation for that monitor id, then returns the monitor object. No JSON body is read. If the monitor has no vendor row, evaluation returns without writing a check.

```bash
curl -sS -X POST -H "Authorization: Bearer coi_..." https://www.coiclock.com/api/v1/monitors/MONITOR_ID/check
```

HTTP 200 `{"data": <monitor>}` or `{"data": null}`. HTTP 404 `{"error":"not_found"}`.

### DELETE /api/v1/monitors/{id}

```bash
curl -sS -X DELETE -H "Authorization: Bearer coi_..." https://www.coiclock.com/api/v1/monitors/MONITOR_ID
```

HTTP 204 with an empty body, or HTTP 404 `{"error":"not_found"}`.

## Rate limits

60 requests per minute per key id, window 60 seconds.

After the key is recognized, the response sets `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset`. A missing or unknown key does not set them.

Above 60: HTTP 429, `Retry-After` seconds until the window ends, and:

```json
{"error":"rate_limited","message":"Limit is 60 requests per minute per key."}
```

## Errors

| HTTP | `error` | Notes |
| --- | --- | --- |
| 401 | `unauthorized` | Prefix or unknown key includes `message`. Missing team owner has no `message`. |
| 400 | `invalid_json` | No `message`. |
| 400 | `invalid_hostname` | Message tells the caller to send `hostname` and optional `port`. |
| 402 | `plan_limit` | `The <plan> plan covers <n> monitors.` |
| 409 | `duplicate` | `That hostname and port are already on the account.` |
| 404 | `not_found` | No `message`. |
| 429 | `rate_limited` | Includes the limit `message`. |

## Signed webhooks

A webhook delivery POSTs JSON and sets `X-CoiClock-Event` and `X-CoiClock-Signature: t=<unix seconds>,v1=<hex>`.

`v1` is HMAC-SHA256, hex, of `timestamp + "." + raw body`, using the monitor signing secret.

Default body fields: `event`, `id`, `sent_at`, `monitor` (`id`, `hostname`, `port`, `url`), `subject`, `text`, `data`.
