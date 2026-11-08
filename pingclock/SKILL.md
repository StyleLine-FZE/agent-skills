---
name: pingclock
description: Use when an agent needs to create or manage Ping Clock heartbeat checks, read ping history, send a ping to a check URL, or verify signed Ping Clock alert webhooks. The REST API starts on Pro.
---

# Ping Clock

Ping Clock expects a job to ping a URL on a period or cron schedule. A late or failed ping is the alert. The REST API manages checks. The ping URL is public and does not use the API key.

Public docs: https://www.pingclock.com/docs/api

## API key

Sign in and open Settings: https://www.pingclock.com/app/settings

Create a key by POSTing `/app/settings/api-keys`. The field `name` is trimmed to 60 characters. An empty name is stored as `API key`. The full key is shown once. An account can hold 10 keys.

On Free or Starter, key creation returns HTTP 403 on the settings page with `The REST API starts on Pro.`

Revoke a key by POSTing `/app/settings/api-keys/{id}/delete`.

The key prefix is `pngc_`.

## Authentication

```http
Authorization: Bearer pngc_...
```

A missing bearer token, or a token that does not start with `pngc_`, is HTTP 401:

```json
{"error":"unauthorized","message":"Send Authorization: Bearer <api key>."}
```

An unknown or revoked key is HTTP 401:

```json
{"error":"unauthorized","message":"Unknown or revoked API key."}
```

If the key's user has `team_owner_id`, the call runs as that owner. If that owner row is missing, the response is HTTP 401 `{"error":"unauthorized"}` with no `message`.

When the plan is not Pro or Business, `/api/v1` returns HTTP 403:

```json
{"error":"plan_required","message":"The REST API starts on Pro."}
```

That 403 is built as a new response after the rate-limit headers are set on the framework context. Ping URLs do not use this plan check.

## Base URL

`https://www.pingclock.com`

Every `/api/v1/*` response sets `Cache-Control: no-store`.

## Check object

When a heartbeat row exists:

```json
{
  "id": "",
  "name": "",
  "hostname": "",
  "paused": false,
  "status": "new",
  "heartbeat_status": "new",
  "ping_url": "https://www.pingclock.com/ping/TOKEN",
  "ping_token": "",
  "email_address": "TOKEN@ping.pingclock.com",
  "schedule": {"kind": "period", "every_seconds": 86400, "timezone": "UTC", "grace_seconds": 1800},
  "last_ping_at": null,
  "next_expected_at": null,
  "last_duration_ms": null,
  "created_at": ""
}
```

`name` and `hostname` are both the check name. `ping_url` is `{baseUrl}/ping/{pingToken}` with one trailing slash stripped from the base URL.

`heartbeat_status` is `paused` when the list marks the monitor paused. Otherwise a stored `down` is returned as `late`, a stored `fail` as `down`, a stored `running` as `up`, and `new` or `up` stay as stored.

`status` is `new` when `heartbeat_status` is `new`, `ok` when it is `up`, `paused` when it is `paused`, and `problem` otherwise.

A cron schedule is `{"kind":"cron","expression":"","timezone":"","grace_seconds":0}`. A period schedule includes `every_seconds` and does not include `expression`.

`last_duration_ms` is set on Pro and Business when a start ping is consumed by the next success, fail, or exit ping. Other plans store null. The REST API itself is only available on Pro and Business.

If the heartbeat row is missing, the object is only `id`, `name`, `hostname`, `paused`, `status` (`paused` or `pending`), and `created_at`.

On the list, `paused` follows the plan pause set. GET one and POST check pass `paused: false`.

## Endpoints

### GET /api/v1/monitors

```bash
curl -sS -H "Authorization: Bearer pngc_..." https://www.pingclock.com/api/v1/monitors
```

HTTP 200 `{"data":[<check>...],"plan":"pro","limit":0,"planLimit":0,"bonusLimit":0}`.

`limit` is `monitorLimit` for the account. `planLimit` is `planLimit(plan, "monitors").total`. `bonusLimit` is `planLimit(user, "monitors").bonus`.

### POST /api/v1/monitors

JSON object with `name` (or `hostname`) and `schedule`.

`name` is trimmed, 1 to 80 characters. A bad name returns HTTP 400 `{"error":"invalid_check","message":"Give the check a name, up to 80 characters."}`. A body that is not a JSON object at the product parser returns `message` `Send a JSON object with name and schedule.` A body that is not JSON at all is HTTP 400 `{"error":"invalid_json"}` with no `message`.

Period schedule:

```json
{"name":"Nightly backup","schedule":{"kind":"period","every_seconds":86400,"grace_seconds":1800,"timezone":"UTC"}}
```

`every_seconds` is an integer from 1 to 2592000 (30 days). `grace_seconds` is an integer from 1 to 172800 (48 hours). `timezone` is an IANA name. Omitted timezone is `UTC`. Omitted `grace_seconds` is stored as 1800. Omitted `every_seconds` is rejected unless the schedule object also has `every`, in which case the stored period is 86400.

Cron schedule:

```json
{"name":"Nightly backup","schedule":{"kind":"cron","expression":"*/5 * * * *","timezone":"Europe/Berlin","grace_seconds":300}}
```

`expression` must parse and must have a next run. A bad timezone message is `Pick an IANA time zone, such as Europe/Berlin.`

```bash
curl -sS -X POST -H "Authorization: Bearer pngc_..." -H "Content-Type: application/json" \
  -d '{"name":"Nightly backup","schedule":{"kind":"period","every_seconds":86400,"grace_seconds":1800}}' \
  https://www.pingclock.com/api/v1/monitors
```

HTTP 201 `{"data": <check>}`.

HTTP 402 `{"error":"plan_limit","message":"The <plan> plan covers <n> monitors."}`.

HTTP 409 `{"error":"duplicate","message":"A check with that name is already on the account."}`. The stored hostname is the name and the port is 443.

### GET /api/v1/monitors/{id}

```bash
curl -sS -H "Authorization: Bearer pngc_..." https://www.pingclock.com/api/v1/monitors/MONITOR_ID
```

HTTP 200 `{"data": <check>}`. HTTP 404 `{"error":"not_found"}`.

### GET /api/v1/monitors/{id}/checks

Ping events, not a fixed 50.

```bash
curl -sS -H "Authorization: Bearer pngc_..." https://www.pingclock.com/api/v1/monitors/MONITOR_ID/checks
```

HTTP 200. Each row:

```json
{"received_at":"","kind":"success","exit_code":0,"duration_ms":null,"method":"GET","body":null,"source_ip":null}
```

Pro returns up to 200 events from the last 30 days. Business returns up to 500 events from the last 365 days. `body` is the POST body, capped at 10240 bytes, and only when the plan allows logs (Pro and Business). `duration_ms` is null unless the plan allows duration.

### POST /api/v1/monitors/{id}/check

Evaluates due state for that check and returns the check object. No JSON body is read.

```bash
curl -sS -X POST -H "Authorization: Bearer pngc_..." https://www.pingclock.com/api/v1/monitors/MONITOR_ID/check
```

HTTP 200 `{"data": <check>}` or `{"data": null}`. HTTP 404 `{"error":"not_found"}`.

### DELETE /api/v1/monitors/{id}

```bash
curl -sS -X DELETE -H "Authorization: Bearer pngc_..." https://www.pingclock.com/api/v1/monitors/MONITOR_ID
```

HTTP 204 with an empty body, or HTTP 404 `{"error":"not_found"}`.

### Ping URLs

No API key. GET and POST are both accepted.

| Path | Kind |
| --- | --- |
| `/ping/{token}` | success, exit code 0 |
| `/ping/{token}/start` | start |
| `/ping/{token}/fail` | fail, exit code 1 |
| `/ping/{token}/{code}` | `{code}` is 0 to 255. 0 is success. 1 to 255 is an exit ping. |

```bash
curl -sS https://www.pingclock.com/ping/PING_TOKEN
curl -sS https://www.pingclock.com/ping/PING_TOKEN/start
curl -sS -X POST --data 'disk full' https://www.pingclock.com/ping/PING_TOKEN/fail
```

POST body text is the log. Success is HTTP 200 with the text `ok`.

A paused monitor returns HTTP 200 with the text `paused` and does not record a ping.

HTTP 404 `{"error":"unknown_ping"}` when the suffix is not start, fail, or a code 0 to 255.

HTTP 404 `{"error":"unknown_check"}` when the token is unknown.

Per client address: 60 requests a minute, then HTTP 429 `{"error":"rate_limited"}`. Per check: 120 pings an hour, then HTTP 429 `{"error":"fair_use"}`.

`email_address` on the check object is `{emailToken}@ping.pingclock.com`. That address is for mail into the product. It is not a REST call.

## Rate limits

`/api/v1` allows 60 requests per minute per key id, window 60 seconds.

After the key is recognized, the response sets `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset`. A missing or unknown key does not set them.

Above 60: HTTP 429, `Retry-After` seconds until the window ends, and:

```json
{"error":"rate_limited","message":"Limit is 60 requests per minute per key."}
```

Ping URL limits are separate and are listed above.

## Errors

| HTTP | `error` | Notes |
| --- | --- | --- |
| 401 | `unauthorized` | Prefix or unknown key includes `message`. Missing team owner has no `message`. |
| 400 | `invalid_json` | Body was not JSON. No `message`. |
| 400 | `invalid_check` | `message` is the parser error. |
| 402 | `plan_limit` | `The <plan> plan covers <n> monitors.` |
| 403 | `plan_required` | `The REST API starts on Pro.` |
| 409 | `duplicate` | `A check with that name is already on the account.` |
| 404 | `not_found` | REST id missing. No `message`. |
| 404 | `unknown_check` or `unknown_ping` | Ping URL. |
| 429 | `rate_limited` | REST includes the limit `message`. Ping URL has no `message`. |
| 429 | `fair_use` | Ping URL, 120 an hour for that check. |

## Signed webhooks

A webhook delivery POSTs JSON and sets `X-PingClock-Event` and `X-PingClock-Signature: t=<unix seconds>,v1=<hex>`.

`v1` is HMAC-SHA256, hex, of `timestamp + "." + raw body`, using the monitor signing secret.

Default body fields: `event`, `id`, `sent_at`, `monitor` (`id`, `hostname`, `port`, `url`), `subject`, `text`, `data`.

Ping failures and recoveries send `heartbeat.fail` and `heartbeat.recovered`.
