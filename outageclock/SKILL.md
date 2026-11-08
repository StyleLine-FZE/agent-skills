---
name: outageclock
description: Use when an agent needs to follow vendors on Outage Clock, read vendor status polls, list the public vendor catalog, or verify signed Outage Clock alert webhooks. The account API is the Business plan.
---

# Outage Clock

Outage Clock follows vendor status feeds. A monitor hostname is a catalog vendor id, and the port is 443. This API does not watch an arbitrary website.

Public docs: https://www.outageclock.com/docs/api

## API key

Sign in and open Settings: https://www.outageclock.com/app/settings

Create a key by POSTing `/app/settings/api-keys`. The field `name` is trimmed to 60 characters. An empty name is stored as `API key`. The full key is shown once. The account stores a hash. An account can hold 10 keys. A further create returns HTTP 400 on the settings page with `Up to 10 API keys per account. Revoke one first.`

Key creation on a plan other than Business returns HTTP 402 on the settings page with `API access starts on Business.`

Revoke a key by POSTing `/app/settings/api-keys/{id}/delete`.

The key prefix is `ocl_`.

## Authentication

```http
Authorization: Bearer ocl_...
```

A missing bearer token, or a token that does not start with `ocl_`, is HTTP 401:

```json
{"error":"unauthorized","message":"Send Authorization: Bearer <api key>."}
```

An unknown or revoked key is HTTP 401:

```json
{"error":"unauthorized","message":"Unknown or revoked API key."}
```

If the key's user has `team_owner_id`, the call runs as that owner. If that owner row is missing, the response is HTTP 401 `{"error":"unauthorized"}` with no `message`.

When the owner plan is not Business, every `/api/v1` call returns HTTP 402 before the route body runs:

```json
{"error":"plan_required","message":"The REST API starts on Business."}
```

That check runs after the rate-limit check, so a request over the limit returns 429 first.

`GET /api/catalog` does not use a key.

## Base URL

`https://www.outageclock.com`

Every `/api/v1/*` response sets `Cache-Control: no-store`.

## Vendor object

```json
{
  "id": "",
  "hostname": "",
  "port": 443,
  "paused": false,
  "status": "pending",
  "error_code": null,
  "overall_status": null,
  "open_incidents": null,
  "subject": null,
  "reachability_ok": null,
  "components": [],
  "last_checked_at": null,
  "created_at": ""
}
```

`hostname` is the catalog id. `status` is `pending` when `last_checked_at` is null, `problem` when `error_code` is set, and `ok` otherwise.

After a poll, `overall_status` is the snapshot status stored in `last_issuer`: `operational`, `degraded`, `partial_outage`, `major_outage`, or `maintenance`. `open_incidents` is the count of open feed incidents. `subject` is the first open incident name, or null. `reachability_ok` is true or false. A feed marked `unavailable` is stored as reachable (`true`) so it is not labeled unreachable. `error_code` is null when the poll is ok (`operational`, reachable, and no open feed incidents). Otherwise it is `incident` when the feed was reachable and `unreachable` when it was not. A stored `tls_error` is returned as `incident`. `components` is a list of component name strings parsed from the stored names. It does not include each component's status.

On the list, `paused` follows the plan pause set. GET one monitor and POST check return `paused: false`.

## Endpoints

### GET /api/catalog

Public catalog. No key.

```bash
curl -sS https://www.outageclock.com/api/catalog
```

HTTP 200:

```json
{"data":[{"id":"","name":"","domain":"","format":""}]}
```

`format` is the catalog feed format. The route allows 60 requests per minute for the client address (`X-Forwarded-For` first hop, else `X-Real-IP`, else `local`). The next request in that 60 second window is HTTP 429 `{"error":"rate_limited"}` with no `Retry-After` and no `X-RateLimit-*` headers.

### GET /api/v1/monitors

```bash
curl -sS -H "Authorization: Bearer ocl_..." https://www.outageclock.com/api/v1/monitors
```

HTTP 200 `{"data":[<vendor>...],"plan":"business","limit":0,"planLimit":0,"bonusLimit":0}`.

`limit` is `monitorLimit` for the account. `planLimit` is `planLimit(plan, "monitors").total`. `bonusLimit` is `planLimit(user, "monitors").bonus`. Rows are hydrated from the latest vendor snapshot before JSON is built.

### POST /api/v1/monitors

Follows a vendor. JSON body: `{"hostname":"<catalog id, vendor name, or domain>"}`. `port` is ignored for a catalog match. The stored hostname is the catalog id and the port is 443.

Lookup order: exact catalog id, then the host after stripping an `http://` or `https://` scheme and path, matched as a domain, then an exact case-insensitive name, then the first catalog row whose name or id contains the lowercased query.

```bash
curl -sS -X POST -H "Authorization: Bearer ocl_..." -H "Content-Type: application/json" \
  -d '{"hostname":"github.com"}' \
  https://www.outageclock.com/api/v1/monitors
```

HTTP 201 `{"data": <vendor>}`. After insert, the handler starts a poll and waits up to 20 seconds before it reloads the row. A poll error is swallowed.

HTTP 400 `{"error":"invalid_json"}` when the body is not JSON.

HTTP 400 `{"error":"invalid_hostname","message":"That vendor is not in the catalog. Send a catalog name or domain."}` when lookup fails. This message is used when `hostname` is a string and the engine is vendor status, including a string that is not a catalog hit.

HTTP 402 `plan_limit` with `The <plan> plan covers <n> monitors.` when the account is at `monitorLimit`.

HTTP 409 `{"error":"duplicate","message":"That hostname and port are already on the account."}` when that catalog id is already followed.

### GET /api/v1/monitors/{id}

```bash
curl -sS -H "Authorization: Bearer ocl_..." https://www.outageclock.com/api/v1/monitors/MONITOR_ID
```

HTTP 200 `{"data": <vendor>}`. HTTP 404 `{"error":"not_found"}`.

### GET /api/v1/monitors/{id}/checks

The last 50 polls.

```bash
curl -sS -H "Authorization: Bearer ocl_..." https://www.outageclock.com/api/v1/monitors/MONITOR_ID/checks
```

HTTP 200. Each row is:

```json
{"checked_at":"","ok":true,"error_code":null,"overall_status":"operational","open_incidents":0,"subject":null}
```

`overall_status` is the stored issuer. `open_incidents` is the stored `days_remaining`. `error_code` maps `tls_error` to `incident`. These rows do not include `components` or `reachability_ok`.

### POST /api/v1/monitors/{id}/check

Polls that vendor now and returns the vendor object. No JSON body is read.

```bash
curl -sS -X POST -H "Authorization: Bearer ocl_..." https://www.outageclock.com/api/v1/monitors/MONITOR_ID/check
```

HTTP 200 `{"data": <vendor>}` or `{"data": null}`. HTTP 404 `{"error":"not_found"}`.

### DELETE /api/v1/monitors/{id}

```bash
curl -sS -X DELETE -H "Authorization: Bearer ocl_..." https://www.outageclock.com/api/v1/monitors/MONITOR_ID
```

HTTP 204 with an empty body, or HTTP 404 `{"error":"not_found"}`.

## Rate limits

`/api/v1` allows 60 requests per minute per key id, window 60 seconds.

After the key is recognized, the response sets `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset` (unix seconds). A missing or unknown key does not set them.

Above 60: HTTP 429, `Retry-After` seconds until the window ends, and:

```json
{"error":"rate_limited","message":"Limit is 60 requests per minute per key."}
```

The 429 still includes the three rate-limit headers.

`GET /api/catalog` uses the separate public limit described above.

## Errors

| HTTP | `error` | Where |
| --- | --- | --- |
| 401 | `unauthorized` | Bad prefix or unknown key, with `message`. Missing team owner, without `message`. |
| 400 | `invalid_json` | No `message`. |
| 400 | `invalid_hostname` | Catalog miss, with the catalog message above. |
| 402 | `plan_required` | Plan is not Business. Message: `The REST API starts on Business.` |
| 402 | `plan_limit` | `The <plan> plan covers <n> monitors.` |
| 409 | `duplicate` | `That hostname and port are already on the account.` |
| 404 | `not_found` | No `message`. |
| 429 | `rate_limited` | `/api/v1` includes `message` `Limit is 60 requests per minute per key.` Catalog 429 has no `message`. |

## Signed webhooks

A webhook delivery POSTs JSON and sets `X-OutageClock-Event` and `X-OutageClock-Signature: t=<unix seconds>,v1=<hex>`.

`v1` is HMAC-SHA256, hex, of `timestamp + "." + raw body`, using the monitor signing secret.

Default body fields: `event`, `id`, `sent_at`, `monitor` (`id`, `hostname`, `port`, `url`), `subject`, `text`, `data`.

On Business, a saved custom payload template replaces that body when the template fills. If the template does not fill, the default body is sent.
