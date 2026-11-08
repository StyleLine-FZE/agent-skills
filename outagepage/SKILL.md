---
name: outagepage
description: Use when an agent needs to manage Outage Page monitors through the public REST API, send a heartbeat for a status component, or verify signed Outage Page alert webhooks.
---

# Outage Page

Outage Page is a status page product. The authenticated API stores monitors and runs the product check for a monitor id. Heartbeat components use a separate public POST that carries a token in the path.

Public docs: https://www.outagepage.com/docs/api

## API key

Sign in and open Settings: https://www.outagepage.com/app/settings

Create a key by POSTing `/app/settings/api-keys`. The field `name` is trimmed to 60 characters. An empty name is stored as `API key`. The full key is shown once. An account can hold 10 keys. A further create returns HTTP 400 on the settings page with `Up to 10 API keys per account. Revoke one first.`

Revoke a key by POSTing `/app/settings/api-keys/{id}/delete`.

The key prefix is `opg_`. The handler has no plan gate for creating keys or calling `/api/v1`.

## Authentication

```http
Authorization: Bearer opg_...
```

A missing bearer token, or a token that does not start with `opg_`, is HTTP 401:

```json
{"error":"unauthorized","message":"Send Authorization: Bearer <api key>."}
```

An unknown or revoked key is HTTP 401:

```json
{"error":"unauthorized","message":"Unknown or revoked API key."}
```

If the key's user has `team_owner_id`, the call runs as that owner. If that owner row is missing, the response is HTTP 401 `{"error":"unauthorized"}` with no `message`.

`POST /api/heartbeat/{token}` does not use this header.

## Base URL

`https://www.outagepage.com`

Every `/api/v1/*` response sets `Cache-Control: no-store`.

## Monitor object

The JSON builder is the shared monitor object:

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

`status` is `pending` when `last_checked_at` is null, `problem` when `error_code` is set, and `ok` otherwise. `chain_valid` and `hostname_match` are null until stored, then booleans.

On the list, `paused` follows the plan pause set. GET one and POST check return `paused: false`.

## Endpoints

### GET /api/v1/monitors

```bash
curl -sS -H "Authorization: Bearer opg_..." https://www.outagepage.com/api/v1/monitors
```

HTTP 200 `{"data":[],"plan":"free","limit":0,"planLimit":0,"bonusLimit":0}`.

`limit` is `monitorLimit` for the account. `planLimit` is `planLimit(plan, "monitors").total`. `bonusLimit` is `planLimit(user, "monitors").bonus`.

### POST /api/v1/monitors

JSON `{"hostname":"...","port":443}`. `port` is optional.

The engine is `status-page`, so the target is parsed with `parseCheckTarget`, not as a full page URL. A URL is reduced to hostname and port. The path is dropped. Omitted port is 443. The hostname must be a dotted hostname or IPv4.

```bash
curl -sS -X POST -H "Authorization: Bearer opg_..." -H "Content-Type: application/json" \
  -d '{"hostname":"api.example","port":443}' \
  https://www.outagepage.com/api/v1/monitors
```

HTTP 201 `{"data": <monitor>}`.

HTTP 400 `{"error":"invalid_json"}` when the body is not JSON.

HTTP 400 `{"error":"invalid_hostname","message":"Send {\"hostname\": \"example.com\"}, optionally with \"port\"."}`.

HTTP 402 `{"error":"plan_limit","message":"The <plan> plan covers <n> monitors."}`.

HTTP 409 `{"error":"duplicate","message":"That hostname and port are already on the account."}`.

### GET /api/v1/monitors/{id}

```bash
curl -sS -H "Authorization: Bearer opg_..." https://www.outagepage.com/api/v1/monitors/MONITOR_ID
```

HTTP 200 `{"data": <monitor>}`. HTTP 404 `{"error":"not_found"}`.

### GET /api/v1/monitors/{id}/checks

The last 50 checks.

```bash
curl -sS -H "Authorization: Bearer opg_..." https://www.outagepage.com/api/v1/monitors/MONITOR_ID/checks
```

HTTP 200. Each row: `checked_at`, `ok` (stored flag `1`), `error_code`, `days_remaining`, `issuer`, `valid_to`.

### POST /api/v1/monitors/{id}/check

Calls the product check for that monitor id, then returns the monitor object. No JSON body is read.

```bash
curl -sS -X POST -H "Authorization: Bearer opg_..." https://www.outagepage.com/api/v1/monitors/MONITOR_ID/check
```

HTTP 200 `{"data": <monitor>}` or `{"data": null}`. HTTP 404 `{"error":"not_found"}`.

### DELETE /api/v1/monitors/{id}

```bash
curl -sS -X DELETE -H "Authorization: Bearer opg_..." https://www.outagepage.com/api/v1/monitors/MONITOR_ID
```

HTTP 204 with an empty body, or HTTP 404 `{"error":"not_found"}`.

### POST /api/heartbeat/{token}

Public heartbeat for a status component. No API key. Idempotent: each accepted call sets that component's `status` to `operational` and `heartbeat_last_at` to the current time.

```bash
curl -sS -X POST https://www.outagepage.com/api/heartbeat/HEARTBEAT_TOKEN
```

HTTP 200 `{"ok":true}`.

HTTP 404 `{"error":"not_found"}` when the token does not match a component.

The client address is the first `X-Forwarded-For` hop, or `local`. The route allows 60 calls per 60 seconds for `hb:{address}`. The next call is HTTP 429 `{"error":"rate_limited"}` with no `Retry-After` header from this handler.

## Rate limits

`/api/v1` allows 60 requests per minute per key id, window 60 seconds.

After the key is recognized, the response sets `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset`. A missing or unknown key does not set them.

Above 60: HTTP 429, `Retry-After` seconds until the window ends, and:

```json
{"error":"rate_limited","message":"Limit is 60 requests per minute per key."}
```

The heartbeat route uses the separate limit above.

## Errors

| HTTP | `error` | Notes |
| --- | --- | --- |
| 401 | `unauthorized` | Prefix or unknown key includes `message`. Missing team owner has no `message`. |
| 400 | `invalid_json` | No `message`. |
| 400 | `invalid_hostname` | Message tells the caller to send `hostname` and optional `port`. |
| 402 | `plan_limit` | `The <plan> plan covers <n> monitors.` |
| 409 | `duplicate` | `That hostname and port are already on the account.` |
| 404 | `not_found` | No `message`. |
| 429 | `rate_limited` | `/api/v1` includes the limit `message`. Heartbeat 429 has no `message`. |

## Signed webhooks

A webhook delivery POSTs JSON and sets `X-OutagePage-Event` and `X-OutagePage-Signature: t=<unix seconds>,v1=<hex>`.

`v1` is HMAC-SHA256, hex, of `timestamp + "." + raw body`, using the monitor signing secret.

Default body fields: `event`, `id`, `sent_at`, `monitor` (`id`, `hostname`, `port`, `url`), `subject`, `text`, `data`.
