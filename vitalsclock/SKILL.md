---
name: vitalsclock
description: Use when an agent needs to manage Vitals Clock page monitors through the public REST API, run an anonymous page check or bulk check, or verify signed Vitals Clock alert webhooks.
---

# Vitals Clock

Vitals Clock stores a page URL and a lab performance score. The authenticated API uses the shared monitor routes, with page URLs stored in the hostname field. POST check runs a page check and then returns the monitor object. `POST /api/vitals/check` and `POST /api/vitals/bulk` are public and do not use an API key.

Public docs: https://www.vitalsclock.com/docs/api

## API key

Sign in and open Settings: https://www.vitalsclock.com/app/settings

Create a key by POSTing `/app/settings/api-keys`. The field `name` is trimmed to 60 characters. An empty name is stored as `API key`. The full key is shown once. An account can hold 10 keys.

Revoke a key by POSTing `/app/settings/api-keys/{id}/delete`.

The key prefix is `vcl_`. The handler has no plan gate for creating keys or calling `/api/v1`.

## Authentication

```http
Authorization: Bearer vcl_...
```

A missing bearer token, or a token that does not start with `vcl_`, is HTTP 401:

```json
{"error":"unauthorized","message":"Send Authorization: Bearer <api key>."}
```

An unknown or revoked key is HTTP 401:

```json
{"error":"unauthorized","message":"Unknown or revoked API key."}
```

If the key's user has `team_owner_id`, the call runs as that owner. If that owner row is missing, the response is HTTP 401 `{"error":"unauthorized"}` with no `message`.

The anonymous vitals routes do not use this header.

## Base URL

`https://www.vitalsclock.com`

Every `/api/v1/*` response sets `Cache-Control: no-store`. Anonymous vitals responses set `Cache-Control: no-store` as well.

## Monitor object

Because the engine type is `vitals`, create stores the canonical page URL string in `hostname` and always stores port 443.

```json
{
  "id": "",
  "hostname": "https://example.com/pricing",
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

A saved check sets `issuer` to `vitals`, `days_remaining` to the mobile lab performance score (null when the check has an error), and `subject` to the desktop lab performance score as a string (null when that score is null). `error_code` is `unreachable` when the check has an error, `tls_error` when there are failing metrics, and null otherwise. `chain_valid` and `hostname_match` are stored true. The check `ok` flag is true when there is no error and no failing metrics.

On the list, `paused` follows the plan pause set. GET one and POST check return `paused: false`.

## Endpoints

### GET /api/v1/monitors

```bash
curl -sS -H "Authorization: Bearer vcl_..." https://www.vitalsclock.com/api/v1/monitors
```

HTTP 200 `{"data":[],"plan":"free","limit":0,"planLimit":0,"bonusLimit":0}`.

`limit` is `monitorLimit` for the account. `planLimit` is `planLimit(plan, "monitors").total`. `bonusLimit` is `planLimit(user, "monitors").bonus`.

### POST /api/v1/monitors

JSON `{"hostname":"https://example.com/pricing"}`. `port` in the body is ignored. `parsePageTarget` accepts an `http` or `https` URL, or a host that it prefixes with `https://`. The stored `hostname` is the full URL string, including path. Userinfo in the URL is rejected. The value cannot contain whitespace and cannot be longer than 2048 characters. The host must be a dotted hostname or IPv4.

```bash
curl -sS -X POST -H "Authorization: Bearer vcl_..." -H "Content-Type: application/json" \
  -d '{"hostname":"https://example.com/pricing"}' \
  https://www.vitalsclock.com/api/v1/monitors
```

HTTP 201 `{"data": <monitor>}`.

HTTP 400 `{"error":"invalid_json"}`.

HTTP 400 `{"error":"invalid_hostname","message":"Send {\"hostname\": \"example.com\"}, optionally with \"port\"."}`.

HTTP 402 `{"error":"plan_limit","message":"The <plan> plan covers <n> monitors."}`.

HTTP 409 `{"error":"duplicate","message":"That hostname and port are already on the account."}`. The conflict key is the stored URL plus port 443.

### GET /api/v1/monitors/{id}

```bash
curl -sS -H "Authorization: Bearer vcl_..." https://www.vitalsclock.com/api/v1/monitors/MONITOR_ID
```

HTTP 200 `{"data": <monitor>}`. HTTP 404 `{"error":"not_found"}`.

### GET /api/v1/monitors/{id}/checks

The last 50 checks.

```bash
curl -sS -H "Authorization: Bearer vcl_..." https://www.vitalsclock.com/api/v1/monitors/MONITOR_ID/checks
```

Each row: `checked_at`, `ok`, `error_code`, `days_remaining` (mobile lab score), `issuer`, `valid_to`.

### POST /api/v1/monitors/{id}/check

Runs a page check for that monitor, then returns the monitor object. No JSON body is read.

```bash
curl -sS -X POST -H "Authorization: Bearer vcl_..." https://www.vitalsclock.com/api/v1/monitors/MONITOR_ID/check
```

HTTP 200 `{"data": <monitor>}` or `{"data": null}`. HTTP 404 `{"error":"not_found"}`.

### DELETE /api/v1/monitors/{id}

```bash
curl -sS -X DELETE -H "Authorization: Bearer vcl_..." https://www.vitalsclock.com/api/v1/monitors/MONITOR_ID
```

HTTP 204 with an empty body, or HTTP 404 `{"error":"not_found"}`.

### POST /api/vitals/check

Anonymous check. No API key. JSON fields, all optional strings with min length 1: `url`, `target`, `hostname`. The first one present is parsed as a page URL.

```bash
curl -sS -X POST -H "Content-Type: application/json" \
  -d '{"url":"https://example.com/pricing"}' \
  https://www.vitalsclock.com/api/vitals/check
```

HTTP 200 is the check object: `url`, `fetchedAt`, `mobile`, `desktop`, `pageWeight`, `error`. `mobile` and `desktop` each include `lab`, `field`, and `fieldHistory`.

HTTP 400 `{"error":"Enter a page URL like https://example.com/pricing"}` when the body fails the schema or the URL does not parse.

HTTP 400 `{"error":"That host resolves to a private, loopback, link-local or reserved address, so it cannot be checked."}` when the host is blocked.

HTTP 503 `{"error":"<check.error>"}` when the check returns an error string.

HTTP 500 `{"error":"<message>"}` when the check throws. The fallback message is `The check failed`.

Per client address: 10 checks per hour, then HTTP 429 `{"error":"Too many checks. Try again in an hour."}`.

### POST /api/vitals/bulk

Anonymous bulk check. JSON `{"urls":["https://a.example/","https://b.example/"]}`. `urls` is an array of strings with at most 5 items.

```bash
curl -sS -X POST -H "Content-Type: application/json" \
  -d '{"urls":["https://a.example/","https://b.example/"]}' \
  https://www.vitalsclock.com/api/vitals/bulk
```

HTTP 200 `{"results":[...]}`. A bad URL in the list becomes `{"url":"<raw>","error":"Not a public http or https URL."}`. A check error becomes `{"url":"<url>","error":"<message>"}`. A successful entry is the same check object as the single route.

HTTP 400 `{"error":"Send up to five URLs."}` when the body fails the schema, including more than five URLs.

Per client address: 6 bulk calls per hour, then HTTP 429 `{"error":"Too many bulk tests. Try again in an hour."}`.

## Rate limits

`/api/v1` allows 60 requests per minute per key id, window 60 seconds.

After the key is recognized, the response sets `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset`. A missing or unknown key does not set them.

Above 60: HTTP 429, `Retry-After` seconds until the window ends, and:

```json
{"error":"rate_limited","message":"Limit is 60 requests per minute per key."}
```

The anonymous routes use the hourly caps above and do not set `Retry-After`.

## Errors

| HTTP | `error` | Notes |
| --- | --- | --- |
| 401 | `unauthorized` | Prefix or unknown key includes `message`. Missing team owner has no `message`. |
| 400 | `invalid_json` | Monitor create, no `message`. |
| 400 | `invalid_hostname` | Monitor create. |
| 400 | sentence | Anonymous routes. The `error` value is the sentence. |
| 402 | `plan_limit` | `The <plan> plan covers <n> monitors.` |
| 409 | `duplicate` | `That hostname and port are already on the account.` |
| 404 | `not_found` | No `message`. |
| 429 | `rate_limited` | `/api/v1` includes the limit `message`. |
| 429 | hourly sentence | Anonymous check or bulk. |
| 503 | check error string | Single anonymous check. |
| 500 | thrown message | Single anonymous check. |

## Signed webhooks

A webhook delivery POSTs JSON and sets `X-VitalsClock-Event` and `X-VitalsClock-Signature: t=<unix seconds>,v1=<hex>`.

`v1` is HMAC-SHA256, hex, of `timestamp + "." + raw body`, using the monitor signing secret.

Default body fields: `event`, `id`, `sent_at`, `monitor` (`id`, `hostname`, `port`, `url`), `subject`, `text`, `data`.
