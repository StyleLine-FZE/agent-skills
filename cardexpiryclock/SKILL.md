---
name: cardexpiryclock
description: Use when an agent needs to list, add, read, scan, or delete Card Expiry Clock monitors through the public REST API. A scan writes the soonest card expiry onto the monitor row. Connecting Stripe is done in the dashboard, not by this JSON body.
---

# Card Expiry Clock

Card Expiry Clock scans a connected Stripe account for cards that expire on active subscriptions. The public API uses the shared monitor routes. POST stores a public hostname. It does not attach a Stripe account. POST check runs the product scan and then returns the monitor object.

Public docs: https://www.cardexpiryclock.com/docs/api

## API key

Sign in and open Settings: https://www.cardexpiryclock.com/app/settings

Create a key by POSTing `/app/settings/api-keys`. The field `name` is trimmed to 60 characters. An empty name is stored as `API key`. The full key is shown once. An account can hold 10 keys.

Revoke a key by POSTing `/app/settings/api-keys/{id}/delete`.

The key prefix is `cec_`. The handler has no plan gate for creating keys or calling `/api/v1`.

## Authentication

```http
Authorization: Bearer cec_...
```

A missing bearer token, or a token that does not start with `cec_`, is HTTP 401:

```json
{"error":"unauthorized","message":"Send Authorization: Bearer <api key>."}
```

An unknown or revoked key is HTTP 401:

```json
{"error":"unauthorized","message":"Unknown or revoked API key."}
```

If the key's user has `team_owner_id`, the call runs as that owner. If that owner row is missing, the response is HTTP 401 `{"error":"unauthorized"}` with no `message`.

## Base URL

`https://www.cardexpiryclock.com`

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

A saved scan sets `issuer` to `card-expiry`, `days_remaining` to the soonest card expiry in days (`soonestDays`), `subject` to `{n} cards in the next 30 days`, and `error_code` to `tls_error` when the scan result has an error. `chain_valid` and `hostname_match` are stored true.

On the list, `paused` follows the plan pause set. GET one and POST check return `paused: false`.

## Endpoints

### GET /api/v1/monitors

```bash
curl -sS -H "Authorization: Bearer cec_..." https://www.cardexpiryclock.com/api/v1/monitors
```

HTTP 200 `{"data":[],"plan":"free","limit":0,"planLimit":0,"bonusLimit":0}`.

`limit` is `monitorLimit` for the account. `planLimit` is `planLimit(plan, "monitors").total`. `bonusLimit` is `planLimit(user, "monitors").bonus`.

### POST /api/v1/monitors

JSON `hostname` and optional `port`. The value must parse as a dotted hostname, IPv4, `host:port`, or URL. A single word such as `production` does not parse. The path of a URL is dropped. Omitted port is 443.

```bash
curl -sS -X POST -H "Authorization: Bearer cec_..." -H "Content-Type: application/json" \
  -d '{"hostname":"billing.example","port":443}' \
  https://www.cardexpiryclock.com/api/v1/monitors
```

HTTP 201 `{"data": <monitor>}`.

HTTP 400 `{"error":"invalid_json"}`.

HTTP 400 `{"error":"invalid_hostname","message":"Send {\"hostname\": \"example.com\"}, optionally with \"port\"."}`.

HTTP 402 `{"error":"plan_limit","message":"The <plan> plan covers <n> monitors."}`.

HTTP 409 `{"error":"duplicate","message":"That hostname and port are already on the account."}`.

### GET /api/v1/monitors/{id}

```bash
curl -sS -H "Authorization: Bearer cec_..." https://www.cardexpiryclock.com/api/v1/monitors/MONITOR_ID
```

HTTP 200 `{"data": <monitor>}`. HTTP 404 `{"error":"not_found"}`.

### GET /api/v1/monitors/{id}/checks

The last 50 scans.

```bash
curl -sS -H "Authorization: Bearer cec_..." https://www.cardexpiryclock.com/api/v1/monitors/MONITOR_ID/checks
```

Each row: `checked_at`, `ok`, `error_code`, `days_remaining`, `issuer`, `valid_to`.

### POST /api/v1/monitors/{id}/check

Runs the card-expiry scan for that monitor id, then returns the monitor object. No JSON body is read.

```bash
curl -sS -X POST -H "Authorization: Bearer cec_..." https://www.cardexpiryclock.com/api/v1/monitors/MONITOR_ID/check
```

HTTP 200 `{"data": <monitor>}` or `{"data": null}`. HTTP 404 `{"error":"not_found"}`.

### DELETE /api/v1/monitors/{id}

```bash
curl -sS -X DELETE -H "Authorization: Bearer cec_..." https://www.cardexpiryclock.com/api/v1/monitors/MONITOR_ID
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

A webhook delivery POSTs JSON and sets `X-CardExpiryClock-Event` and `X-CardExpiryClock-Signature: t=<unix seconds>,v1=<hex>`.

`v1` is HMAC-SHA256, hex, of `timestamp + "." + raw body`, using the monitor signing secret.

Default body fields: `event`, `id`, `sent_at`, `monitor` (`id`, `hostname`, `port`, `url`), `subject`, `text`, `data`.
