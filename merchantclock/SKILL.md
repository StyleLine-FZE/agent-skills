---
name: merchantclock
description: Use when an agent needs to manage Merchant Clock store monitors through the public REST API, run an anonymous store scan, or verify signed Merchant Clock alert webhooks. API keys use the sslc_ prefix.
---

# Merchant Clock

Merchant Clock scans a storefront and stores a score. The authenticated API uses the shared monitor routes. POST check runs a store scan and then returns the monitor object. `POST /api/scan` is a public scan and does not use an API key.

Public docs: https://www.merchantclock.com/docs/api

## API key

Sign in and open Settings: https://www.merchantclock.com/app/settings

Create a key by POSTing `/app/settings/api-keys`. The field `name` is trimmed to 60 characters. An empty name is stored as `API key`. The full key is shown once. An account can hold 10 keys.

Revoke a key by POSTing `/app/settings/api-keys/{id}/delete`.

Merchant Clock does not set `apiKeyPrefix`. The prefix is the platform default `sslc_`.

The handler has no plan gate for creating keys or calling `/api/v1`.

## Authentication

```http
Authorization: Bearer sslc_...
```

A missing bearer token, or a token that does not start with `sslc_`, is HTTP 401:

```json
{"error":"unauthorized","message":"Send Authorization: Bearer <api key>."}
```

An unknown or revoked key is HTTP 401:

```json
{"error":"unauthorized","message":"Unknown or revoked API key."}
```

If the key's user has `team_owner_id`, the call runs as that owner. If that owner row is missing, the response is HTTP 401 `{"error":"unauthorized"}` with no `message`.

`POST /api/scan` does not use this header.

## Base URL

`https://www.merchantclock.com`

Every `/api/v1/*` response sets `Cache-Control: no-store`. Anonymous scan responses set `Cache-Control: no-store` as well.

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

A saved scan sets `issuer` to `store-scan`, `subject` to the scanned URL, and `days_remaining` to the score. A blocked scan stores `days_remaining` null and `error_code` `tls_error`. A score below 50 also stores `error_code` `tls_error`. `chain_valid` and `hostname_match` are stored true. The check is `ok` when the scan is not blocked and the score is at least 50.

On the list, `paused` follows the plan pause set. GET one and POST check return `paused: false`.

## Endpoints

### GET /api/v1/monitors

```bash
curl -sS -H "Authorization: Bearer sslc_..." https://www.merchantclock.com/api/v1/monitors
```

HTTP 200 `{"data":[],"plan":"free","limit":0,"planLimit":0,"bonusLimit":0}`.

`limit` is `monitorLimit` for the account. `planLimit` is `planLimit(plan, "monitors").total`. `bonusLimit` is `planLimit(user, "monitors").bonus`.

### POST /api/v1/monitors

JSON `hostname` and optional `port`. A URL is reduced to hostname and port. The path is dropped. Omitted port is 443. The hostname must be a dotted hostname or IPv4.

```bash
curl -sS -X POST -H "Authorization: Bearer sslc_..." -H "Content-Type: application/json" \
  -d '{"hostname":"https://store.example"}' \
  https://www.merchantclock.com/api/v1/monitors
```

That example stores hostname `store.example` and port 443.

HTTP 201 `{"data": <monitor>}`.

HTTP 400 `{"error":"invalid_json"}`.

HTTP 400 `{"error":"invalid_hostname","message":"Send {\"hostname\": \"example.com\"}, optionally with \"port\"."}`.

HTTP 402 `{"error":"plan_limit","message":"The <plan> plan covers <n> monitors."}`.

HTTP 409 `{"error":"duplicate","message":"That hostname and port are already on the account."}`.

### GET /api/v1/monitors/{id}

```bash
curl -sS -H "Authorization: Bearer sslc_..." https://www.merchantclock.com/api/v1/monitors/MONITOR_ID
```

HTTP 200 `{"data": <monitor>}`. HTTP 404 `{"error":"not_found"}`.

### GET /api/v1/monitors/{id}/checks

The last 50 scans.

```bash
curl -sS -H "Authorization: Bearer sslc_..." https://www.merchantclock.com/api/v1/monitors/MONITOR_ID/checks
```

Each row: `checked_at`, `ok`, `error_code`, `days_remaining` (the score), `issuer`, `valid_to`.

### POST /api/v1/monitors/{id}/check

Runs a store scan for that monitor, then returns the monitor object. No JSON body is read. The saved monitor's hostname is what gets scanned.

```bash
curl -sS -X POST -H "Authorization: Bearer sslc_..." https://www.merchantclock.com/api/v1/monitors/MONITOR_ID/check
```

HTTP 200 `{"data": <monitor>}` or `{"data": null}`. HTTP 404 `{"error":"not_found"}`.

### DELETE /api/v1/monitors/{id}

```bash
curl -sS -X DELETE -H "Authorization: Bearer sslc_..." https://www.merchantclock.com/api/v1/monitors/MONITOR_ID
```

HTTP 204 with an empty body, or HTTP 404 `{"error":"not_found"}`.

### POST /api/scan

Anonymous scan. No API key. JSON fields, all optional strings: `url`, `target`, `hostname`, `feedUrl`, `planId`. At least one of `url`, `target`, or `hostname` must parse as a store URL. A value without a scheme is treated as `https://`. The host must look like a dotted hostname.

`feedUrl` is used only for a signed-in caller whose plan allows the feed feature, and only when that URL also parses.

```bash
curl -sS -X POST -H "Content-Type: application/json" \
  -d '{"url":"https://store.example"}' \
  https://www.merchantclock.com/api/scan
```

HTTP 200 is the scan object: `url`, `score`, `pagesCrawled`, `jsRendered`, optional `blocked`, and `results`. Each result has `id`, `category`, `status` (`pass`, `warn`, `fail`, or `skip`), `weight`, `pointsLost`, `evidence` (`url` and `excerpt`, or null), and `fix`.

HTTP 400 `{"error":"Enter a store URL like https://yourstore.com"}` when the body fails the schema or the URL does not parse.

HTTP 400 `{"error":"That host resolves to a private, loopback, link-local or reserved address, so it can't be checked."}` when the host is blocked.

HTTP 500 `{"error":"<message>"}` when the scan throws. The fallback message is `The scan failed`.

Per client address: 10 scans per hour, then HTTP 429 `{"error":"Too many scans. Try again in an hour."}`. A signed-in caller also has a separate cap of 20 scans per hour for that user id, with the same 429 body.

## Rate limits

`/api/v1` allows 60 requests per minute per key id, window 60 seconds.

After the key is recognized, the response sets `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset`. A missing or unknown key does not set them.

Above 60: HTTP 429, `Retry-After` seconds until the window ends, and:

```json
{"error":"rate_limited","message":"Limit is 60 requests per minute per key."}
```

`POST /api/scan` uses the hourly caps above. Those 429 responses do not set `Retry-After`.

## Errors

| HTTP | `error` | Notes |
| --- | --- | --- |
| 401 | `unauthorized` | Prefix or unknown key includes `message`. Missing team owner has no `message`. |
| 400 | `invalid_json` | Monitor create, no `message`. |
| 400 | `invalid_hostname` | Monitor create. |
| 400 | string message | Anonymous scan, `error` is the sentence, not a code. |
| 402 | `plan_limit` | `The <plan> plan covers <n> monitors.` |
| 409 | `duplicate` | `That hostname and port are already on the account.` |
| 404 | `not_found` | No `message`. |
| 429 | `rate_limited` | `/api/v1` includes the limit `message`. |
| 429 | `Too many scans. Try again in an hour.` | Anonymous scan. The `error` value is that sentence. |
| 500 | scan message | Anonymous scan failure. |

## Signed webhooks

A webhook delivery POSTs JSON and sets `X-MerchantClock-Event` and `X-MerchantClock-Signature: t=<unix seconds>,v1=<hex>`.

`v1` is HMAC-SHA256, hex, of `timestamp + "." + raw body`, using the monitor signing secret.

Default body fields: `event`, `id`, `sent_at`, `monitor` (`id`, `hostname`, `port`, `url`), `subject`, `text`, `data`.

Store alerts that go through this path use `store.policy_gone` and `store.score_drop`.
