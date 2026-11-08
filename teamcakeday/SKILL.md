---
name: teamcakeday
description: Use when an agent needs to list Team Cake Day members through the public members API, or use the shared monitor routes that the app also mounts. The members route returns month and day, never a birth year.
---

# Team Cake Day

Team Cake Day keeps people and the month and day the team celebrates. `GET /api/v1/members` is the product read API. The app also mounts the shared `/api/v1/monitors` routes. The members route does not create, update, or delete people.

Public docs: https://www.teamcakeday.com/docs/api

## API key

Sign in and open Settings: https://www.teamcakeday.com/app/settings

Create a key by POSTing `/app/settings/api-keys`. The field `name` is trimmed to 60 characters. An empty name is stored as `API key`. The full key is shown once. An account can hold 10 keys.

Revoke a key by POSTing `/app/settings/api-keys/{id}/delete`.

The key prefix in business config is `tcd_`. The members route checks that prefix as the literal `tcd_`.

The handler has no plan gate for creating keys, calling `/api/v1/members`, or calling `/api/v1/monitors`.

## Authentication

```http
Authorization: Bearer tcd_...
```

### Members route

A missing bearer token, a token that does not start with `tcd_`, or an unknown key is HTTP 401:

```json
{"error":"unauthorized"}
```

There is no `message`. This route does not look up a team owner and does not set rate-limit headers.

### Monitor routes

A missing bearer token, or a token that does not start with `tcd_`, is HTTP 401:

```json
{"error":"unauthorized","message":"Send Authorization: Bearer <api key>."}
```

An unknown or revoked key is HTTP 401:

```json
{"error":"unauthorized","message":"Unknown or revoked API key."}
```

If the key's user has `team_owner_id`, monitor calls run as that owner. If that owner row is missing, the response is HTTP 401 `{"error":"unauthorized"}` with no `message`.

## Base URL

`https://www.teamcakeday.com`

`/api/v1/monitors` responses set `Cache-Control: no-store`. The members route does not set that header.

## GET /api/v1/members

```bash
curl -sS -H "Authorization: Bearer tcd_..." https://www.teamcakeday.com/api/v1/members
```

HTTP 200:

```json
{"data":[{"id":"","name":"","birthday_month":1,"birthday_day":1,"start_date":null,"celebrate":""}]}
```

`name` is the member display name. `birthday_month` and `birthday_day` are the stored month and day. `start_date` is the stored start date. `celebrate` is the stored celebrate value. The handler does not include a birth year.

The members route defines no rate limit and no other error code.

## Shared monitor routes

These routes are mounted as well. Team Cake Day wires the product check to a function that returns immediately, so POST check reloads the monitor and does not run a certificate or calendar check.

| Method | Path | Result |
| --- | --- | --- |
| GET | `/api/v1/monitors` | `{"data","plan","limit","planLimit","bonusLimit"}` |
| POST | `/api/v1/monitors` | JSON `hostname` and optional `port`. HTTP 201, 400, 402, or 409. |
| GET | `/api/v1/monitors/{id}` | One monitor. |
| GET | `/api/v1/monitors/{id}/checks` | Last 50 rows: `checked_at`, `ok`, `error_code`, `days_remaining`, `issuer`, `valid_to`. |
| POST | `/api/v1/monitors/{id}/check` | Returns the monitor after the no-op check. No body. |
| DELETE | `/api/v1/monitors/{id}` | HTTP 204 or 404. |

The monitor object fields are `id`, `hostname`, `port`, `paused`, `status` (`pending`, `ok`, or `problem`), `error_code`, `days_remaining`, `valid_from`, `valid_to`, `issuer`, `subject`, `chain_valid`, `hostname_match`, `last_checked_at`, `domain`, `domain_expires_at`, `created_at`.

List `paused` follows the plan pause set. GET one and POST check return `paused: false`.

```bash
curl -sS -H "Authorization: Bearer tcd_..." https://www.teamcakeday.com/api/v1/monitors
```

POST create parses `hostname` as a dotted hostname, IPv4, `host:port`, or URL. The path is dropped. Omitted port is 443.

HTTP 400 `{"error":"invalid_json"}` when the body is not JSON.

HTTP 400 `{"error":"invalid_hostname","message":"Send {\"hostname\": \"example.com\"}, optionally with \"port\"."}`.

HTTP 402 `{"error":"plan_limit","message":"The <plan> plan covers <n> monitors."}`.

HTTP 409 `{"error":"duplicate","message":"That hostname and port are already on the account."}`.

HTTP 404 `{"error":"not_found"}` with no `message`.

## Monitor rate limits

The monitor routes allow 60 requests per minute per key id, window 60 seconds. The members route is not part of this counter.

After the key is recognized, a monitor response sets `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset`. A missing or unknown key does not set them.

Above 60: HTTP 429, `Retry-After` seconds until the window ends, and:

```json
{"error":"rate_limited","message":"Limit is 60 requests per minute per key."}
```

## Signed webhooks

A webhook delivery POSTs JSON and sets `X-TeamCakeDay-Event` and `X-TeamCakeDay-Signature: t=<unix seconds>,v1=<hex>`.

`v1` is HMAC-SHA256, hex, of `timestamp + "." + raw body`, using the monitor signing secret.

Default body fields: `event`, `id`, `sent_at`, `monitor` (`id`, `hostname`, `port`, `url`), `subject`, `text`, `data`.
