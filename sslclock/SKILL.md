---
name: sslclock
description: Use when an agent needs to list, add, read, check, or delete SSL Clock TLS certificate monitors through the public REST API, or verify signed SSL Clock alert webhooks.
---

# SSL Clock

SSL Clock watches public TLS certificates. A monitor is a hostname and port. The checker opens a public TLS connection and stores the certificate result. Private keys, PEM bundles, and hosts that are not reachable on the public internet are outside this API.

Public docs: https://www.sslclock.com/docs/api

## API key

Sign in and open Settings: https://www.sslclock.com/app/settings

Create a key by POSTing the settings form to `/app/settings/api-keys`. The field `name` is trimmed to 60 characters. An empty name is stored as `API key`. The full key is shown once. The account stores a hash. An account can hold 10 keys. A further create returns HTTP 400 on the settings page with the text `Up to 10 API keys per account. Revoke one first.`

Revoke a key by POSTing `/app/settings/api-keys/{id}/delete`.

There is no API route that mints a key. The handler has no plan gate for creating keys or calling `/api/v1`.

## Authentication

```http
Authorization: Bearer sslc_...
```

The prefix is `sslc_` (the platform default). SSL Clock does not set `apiKeyPrefix` in business config.

A missing bearer token, or a token that does not start with `sslc_`, is HTTP 401:

```json
{"error":"unauthorized","message":"Send Authorization: Bearer <api key>."}
```

An unknown or revoked key is HTTP 401:

```json
{"error":"unauthorized","message":"Unknown or revoked API key."}
```

If the key's user has `team_owner_id`, the call runs as that owner. If that owner row is missing, the response is HTTP 401 `{"error":"unauthorized"}` with no `message`.

## Base URL

`https://www.sslclock.com`

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

`status` is `pending` when `last_checked_at` is null, `problem` when `error_code` is set, and `ok` otherwise. `chain_valid` and `hostname_match` are `null` until a check stores them, then booleans. `domain` and `domain_expires_at` are the stored domain columns (`domain_name`, `domain_expires_at`).

On the list, `paused` is true when the id is in the plan pause set. GET one monitor and POST check always return `paused: false`.

## Endpoints

### GET /api/v1/monitors

Lists monitors for the owner.

```bash
curl -sS -H "Authorization: Bearer sslc_..." https://www.sslclock.com/api/v1/monitors
```

HTTP 200:

```json
{"data":[],"plan":"free","limit":0,"planLimit":0,"bonusLimit":0}
```

`limit` is `monitorLimit` for the account (plan plus bonuses). `planLimit` is `planLimit(plan, "monitors").total` with no bonuses applied. `bonusLimit` is `planLimit(user, "monitors").bonus`.

### POST /api/v1/monitors

Adds a monitor. JSON body fields: `hostname` (string) and optional `port`.

`parseCheckTarget` accepts a hostname, an IPv4 address, `host:port`, or an `http` or `https` URL. A URL is reduced to hostname and port. The path, query, and hash are dropped. An omitted port, and an `http` URL with no explicit port, becomes 443. The port must be an integer from 1 to 65535. The hostname is lowercased and must be a dotted hostname or IPv4.

```bash
curl -sS -X POST -H "Authorization: Bearer sslc_..." -H "Content-Type: application/json" \
  -d '{"hostname":"example.com","port":443}' \
  https://www.sslclock.com/api/v1/monitors
```

HTTP 201 `{"data": <monitor>}`. The route does not read a body on the other monitor routes.

HTTP 400 `{"error":"invalid_json"}` when the body is not JSON.

HTTP 400 `{"error":"invalid_hostname","message":"Send {\"hostname\": \"example.com\"}, optionally with \"port\"."}` when the target cannot be parsed.

HTTP 402 `{"error":"plan_limit","message":"The <plan> plan covers <n> monitors."}` when unpaused monitors are already at `monitorLimit`.

HTTP 409 `{"error":"duplicate","message":"That hostname and port are already on the account."}` when that pair exists. The conflict key is `(user_id, hostname, port)`.

### GET /api/v1/monitors/{id}

```bash
curl -sS -H "Authorization: Bearer sslc_..." https://www.sslclock.com/api/v1/monitors/MONITOR_ID
```

HTTP 200 `{"data": <monitor>}`. HTTP 404 `{"error":"not_found"}` when the id is not on this owner.

### GET /api/v1/monitors/{id}/checks

The last 50 checks, newest query limit 50.

```bash
curl -sS -H "Authorization: Bearer sslc_..." https://www.sslclock.com/api/v1/monitors/MONITOR_ID/checks
```

HTTP 200:

```json
{"data":[{"checked_at":"","ok":true,"error_code":null,"days_remaining":null,"issuer":null,"valid_to":null}]}
```

`ok` is true when the stored flag is `1`. There is no query parameter for the limit.

### POST /api/v1/monitors/{id}/check

Runs a TLS check now (`checkMonitor`) and returns the monitor. The handler does not read a JSON body or query string.

```bash
curl -sS -X POST -H "Authorization: Bearer sslc_..." https://www.sslclock.com/api/v1/monitors/MONITOR_ID/check
```

HTTP 200 `{"data": <monitor>}` or `{"data": null}` if the row cannot be reloaded. HTTP 404 `{"error":"not_found"}`.

### DELETE /api/v1/monitors/{id}

```bash
curl -sS -X DELETE -H "Authorization: Bearer sslc_..." https://www.sslclock.com/api/v1/monitors/MONITOR_ID
```

HTTP 204 with an empty body, or HTTP 404 `{"error":"not_found"}`.

## Rate limits

The limit is 60 requests per minute per key id. The window is 60 seconds.

After the key is recognized, the response sets `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset` (unix seconds, `ceil(resetMs / 1000)`). A missing or unknown key does not set them.

Above 60 in the window: HTTP 429, `Retry-After` seconds until the window ends, and:

```json
{"error":"rate_limited","message":"Limit is 60 requests per minute per key."}
```

The 429 still includes the three rate-limit headers. The count increments before the check, so the request that crosses the limit is the one that returns 429.

## Errors

| HTTP | `error` | `message` |
| --- | --- | --- |
| 401 | `unauthorized` | Present for a bad prefix and for an unknown or revoked key. Absent when the team owner row is missing. |
| 400 | `invalid_json` | Absent. |
| 400 | `invalid_hostname` | `Send {"hostname": "example.com"}, optionally with "port".` |
| 402 | `plan_limit` | `The <plan> plan covers <n> monitors.` |
| 409 | `duplicate` | `That hostname and port are already on the account.` |
| 404 | `not_found` | Absent. |
| 429 | `rate_limited` | `Limit is 60 requests per minute per key.` |

There is no `plan_required` response on this API.

## Signed webhooks

Settings stores a webhook URL and secret on the monitor. A webhook delivery POSTs JSON and sets:

- `X-SSLClock-Event`: the event name
- `X-SSLClock-Signature`: `t=<unix seconds>,v1=<hex>`

`v1` is HMAC-SHA256, hex, of the string `timestamp + "." + raw body`, using the monitor signing secret.

Default JSON body fields: `event`, `id` (the dedupe key), `sent_at`, `monitor` (`id`, `hostname`, `port`, `url`), `subject`, `text`, `data`.
