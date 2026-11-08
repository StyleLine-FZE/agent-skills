---
name: mentionclock
description: Use when an agent needs to manage Mention Clock projects through the public REST API, run the public visibility check, or verify signed Mention Clock alert webhooks.
---

# Mention Clock

Mention Clock stores a project as a monitor and a share of mentions from the latest run. The authenticated API uses the shared monitor routes. POST check runs a saved project and then returns the monitor object. `POST /api/visibility` is a public check and does not use an API key.

Public docs: https://www.mentionclock.com/docs/api

## API key

Sign in and open Settings: https://www.mentionclock.com/app/settings

Create a key by POSTing `/app/settings/api-keys`. The field `name` is trimmed to 60 characters. An empty name is stored as `API key`. The full key is shown once. An account can hold 10 keys.

Revoke a key by POSTing `/app/settings/api-keys/{id}/delete`.

The key prefix is `mcl_`. The handler has no plan gate for creating keys or calling `/api/v1`.

## Authentication

```http
Authorization: Bearer mcl_...
```

A missing bearer token, or a token that does not start with `mcl_`, is HTTP 401:

```json
{"error":"unauthorized","message":"Send Authorization: Bearer <api key>."}
```

An unknown or revoked key is HTTP 401:

```json
{"error":"unauthorized","message":"Unknown or revoked API key."}
```

If the key's user has `team_owner_id`, the call runs as that owner. If that owner row is missing, the response is HTTP 401 `{"error":"unauthorized"}` with no `message`.

`POST /api/visibility` does not use this header.

## Base URL

`https://www.mentionclock.com`

Every `/api/v1/*` response sets `Cache-Control: no-store`. The public visibility response sets `Cache-Control: no-store` as well.

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

A saved run sets `issuer` to `ai-visibility`, `subject` to the brand, and `days_remaining` to `share` from that run. `error_code` is null when `share` is greater than 0, and `tls_error` otherwise. The check `ok` flag is true when `share` is greater than 0. `chain_valid` and `hostname_match` are stored true.

On the list, `paused` follows the plan pause set. GET one and POST check return `paused: false`.

## Endpoints

### GET /api/v1/monitors

```bash
curl -sS -H "Authorization: Bearer mcl_..." https://www.mentionclock.com/api/v1/monitors
```

HTTP 200 `{"data":[],"plan":"free","limit":0,"planLimit":0,"bonusLimit":0}`.

`limit` is `monitorLimit` for the account. `planLimit` is `planLimit(plan, "monitors").total`. `bonusLimit` is `planLimit(user, "monitors").bonus`.

### POST /api/v1/monitors

JSON `hostname` and optional `port`, parsed as a dotted hostname, IPv4, `host:port`, or URL. A URL is reduced to hostname and port. The path is dropped. Omitted port is 443. This route does not accept brand, category, or competitor fields. Those belong to the dashboard project form and to `POST /api/visibility`.

```bash
curl -sS -X POST -H "Authorization: Bearer mcl_..." -H "Content-Type: application/json" \
  -d '{"hostname":"brand.example"}' \
  https://www.mentionclock.com/api/v1/monitors
```

HTTP 201 `{"data": <monitor>}`.

HTTP 400 `{"error":"invalid_json"}`.

HTTP 400 `{"error":"invalid_hostname","message":"Send {\"hostname\": \"example.com\"}, optionally with \"port\"."}`.

HTTP 402 `{"error":"plan_limit","message":"The <plan> plan covers <n> monitors."}`.

HTTP 409 `{"error":"duplicate","message":"That hostname and port are already on the account."}`.

### GET /api/v1/monitors/{id}

```bash
curl -sS -H "Authorization: Bearer mcl_..." https://www.mentionclock.com/api/v1/monitors/MONITOR_ID
```

HTTP 200 `{"data": <monitor>}`. HTTP 404 `{"error":"not_found"}`.

### GET /api/v1/monitors/{id}/checks

The last 50 runs.

```bash
curl -sS -H "Authorization: Bearer mcl_..." https://www.mentionclock.com/api/v1/monitors/MONITOR_ID/checks
```

Each row: `checked_at`, `ok`, `error_code`, `days_remaining` (the share), `issuer`, `valid_to`.

### POST /api/v1/monitors/{id}/check

Runs the saved project for that monitor id, then returns the monitor object. No JSON body is read.

```bash
curl -sS -X POST -H "Authorization: Bearer mcl_..." https://www.mentionclock.com/api/v1/monitors/MONITOR_ID/check
```

HTTP 200 `{"data": <monitor>}` or `{"data": null}`. HTTP 404 `{"error":"not_found"}`.

### DELETE /api/v1/monitors/{id}

```bash
curl -sS -X DELETE -H "Authorization: Bearer mcl_..." https://www.mentionclock.com/api/v1/monitors/MONITOR_ID
```

HTTP 204 with an empty body, or HTTP 404 `{"error":"not_found"}`.

### POST /api/visibility

Public check. No API key. JSON:

- `brand`: string, 1 to 80 characters, required
- `website`: optional string, 1 to 300 characters
- `hostname`: optional string, 1 to 300 characters, used when `website` is absent
- `category`: optional string, 1 to 80 characters. Omitted category is stored as `software`
- `competitor`: optional string, max 80 characters
- `kind`: optional `visibility` or `description`. Any other value is treated as `visibility` only after a successful parse. The schema rejects other strings.

```bash
curl -sS -X POST -H "Content-Type: application/json" \
  -d '{"brand":"Example","website":"https://example.com","category":"software"}' \
  https://www.mentionclock.com/api/visibility
```

HTTP 200 is the run object plus `resultId` when a public result row was saved. Run fields: `brand`, `website`, `answers`, `mentionRate`, `share`, `byEngine`, `citedSources`, `failures`, `costUsd`, `capped`.

HTTP 400 `{"error":"Enter a brand name, website and category."}` when the body fails the schema. `brand` is the required field in the schema. `website` and `category` are optional there.

HTTP 429 `{"error":"Too many free checks. Try again tomorrow."}` after 3 calls in 24 hours for the client address.

HTTP 500 `{"error":"<message>"}` when the run throws. The fallback message is `The check failed`.

## Rate limits

`/api/v1` allows 60 requests per minute per key id, window 60 seconds.

After the key is recognized, the response sets `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset`. A missing or unknown key does not set them.

Above 60: HTTP 429, `Retry-After` seconds until the window ends, and:

```json
{"error":"rate_limited","message":"Limit is 60 requests per minute per key."}
```

`POST /api/visibility` uses the daily cap above and does not set `Retry-After`.

## Errors

| HTTP | `error` | Notes |
| --- | --- | --- |
| 401 | `unauthorized` | Prefix or unknown key includes `message`. Missing team owner has no `message`. |
| 400 | `invalid_json` | Monitor create, no `message`. |
| 400 | `invalid_hostname` | Monitor create. |
| 400 | `Enter a brand name, website and category.` | Public check schema failure. |
| 402 | `plan_limit` | `The <plan> plan covers <n> monitors.` |
| 409 | `duplicate` | `That hostname and port are already on the account.` |
| 404 | `not_found` | No `message`. |
| 429 | `rate_limited` | `/api/v1` includes the limit `message`. |
| 429 | `Too many free checks. Try again tomorrow.` | Public check. |
| 500 | thrown message | Public check. |

## Signed webhooks

A webhook delivery POSTs JSON and sets `X-MentionClock-Event` and `X-MentionClock-Signature: t=<unix seconds>,v1=<hex>`.

`v1` is HMAC-SHA256, hex, of `timestamp + "." + raw body`, using the monitor signing secret.

Default body fields: `event`, `id`, `sent_at`, `monitor` (`id`, `hostname`, `port`, `url`), `subject`, `text`, `data`.
