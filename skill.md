---
name: tumban
description: Tumban is a creator-profile compliance scanning API. Use this skill when an agent needs to submit profile URLs for ToS / prohibited-content analysis, retrieve the resulting triage report, integrate webhooks for asynchronous results, or manage organization API keys and webhook secrets. Covers every v2 endpoint, the asynchronous scan lifecycle, rate-limit semantics, webhook signature verification, and the role-aware permission model.
license: Proprietary. Contact hello@tumban.com for terms.
compatibility: Tumban v2 public API. Requires HTTPS, JSON over `application/json`, and a bearer credential (`sk_…` API key or dashboard session token).
metadata:
  api-version: v2
  base-url: https://api.tumban.com
  docs: https://docs.tumban.com
---

# Tumban v2 — agent skill

Tumban analyses creator profile URLs and returns a triage report
(`recommendation`, `risk_score`, `confidence`, `reason_codes`,
`evidence_index`, `coverage`). Scans are asynchronous: submission
returns a `scan_id` immediately, and final results are delivered via
webhook to a `callback_url` or polled with `GET /api/v2/scans/{scan_id}`.

## Base URL and auth

```
Base URL: https://api.tumban.com
All v2 endpoints mounted under: /api/v2
```

Every request requires `Authorization: Bearer <token>`. Two token kinds:

- **API key** — `sk_<64-hex>` (67 characters total). Long-lived
  server-side credential. Returned exactly once at creation; Tumban
  stores only its SHA-256 hash.
- **Dashboard session token** — short-lived browser credential issued
  by the auth provider. Used by the Tumban dashboard.

`org_id` (returned in `Get org settings` and on signed webhooks) is an
**opaque, case-sensitive string** with the prefix `org_`. Do not regex
against hex or any fixed alphabet — the body is alphanumeric and may
include mixed case.

## Endpoint catalogue (canonical paths)

### Scans

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/api/v2/scan` | Submit a single profile URL |
| `POST` | `/api/v2/batch` | Submit up to N profile URLs in one request |
| `GET`  | `/api/v2/scans/{scan_id}` | Fetch a scan record (status + triage report) |
| `GET`  | `/api/v2/batches/{batch_id}` | Fetch aggregate batch progress |

### Organization

| Method | Path | Purpose |
|--------|------|---------|
| `GET`    | `/api/v2/org/settings` | Read org settings |
| `PATCH`  | `/api/v2/org/settings` | Update `default_callback_url` (admin-only, dashboard session) |
| `POST`   | `/api/v2/org/webhook-secret/rotate` | Rotate webhook signing secret (admin-only, dashboard session) |
| `GET`    | `/api/v2/org/scans` | List recent scans for the org |

### API key management

| Method | Path | Purpose |
|--------|------|---------|
| `POST`   | `/api/v2/org/api-keys` | Create a new API key (raw value shown once) |
| `GET`    | `/api/v2/org/api-keys` | List API keys (hashes + metadata) |
| `DELETE` | `/api/v2/org/api-keys/{key_id}` | Revoke an API key |

Usage / analytics endpoints are **not** part of the public API. The
dashboard renders usage data internally; there is no supported way to
read it programmatically.

## Endpoint details that summarisers commonly get wrong

These are the high-precision facts the autogenerator must not collapse.

### Revoke API key — `DELETE`, no `/revoke` suffix

```
DELETE /api/v2/org/api-keys/{key_id}
```

There is **no** `POST /api/v2/org/api-keys/{key_id}/revoke` variant.
Hitting that path returns `404`; hitting the correct path with `POST`
returns `405`. The canonical form is `DELETE` on the bare key resource.
Successful response is `204 No Content` with no body.

### Rotate webhook secret — `/org/` segment is required

```
POST /api/v2/org/webhook-secret/rotate
```

`POST /api/v2/webhook-secret/rotate` (without `/org/`) does not exist
and returns `404`. There are no path variants — only the `/org/`-prefixed
form is wired up.

## Authentication and role model

API keys can call almost every endpoint. The five exceptions below
require a dashboard session — API-key auth gets `403` immediately:

| Endpoint | Required auth |
|----------|---------------|
| `PATCH /api/v2/org/settings` | Dashboard session, role `admin` |
| `POST /api/v2/org/webhook-secret/rotate` | Dashboard session, role `admin` |
| `POST /api/v2/org/api-keys` | Dashboard session (any role) |
| `GET /api/v2/org/api-keys` | Dashboard session (any role) |
| `DELETE /api/v2/org/api-keys/{key_id}` | Dashboard session (role-aware — see below) |

Credential-management endpoints (the three `/api-keys` rows) reject
`sk_…` auth so a leaked key cannot mint, list, or revoke other keys.
The exact 403 detail strings differ between create/list and revoke —
see `https://docs.tumban.com/api/errors` for the canonical list.

### Revoke role rules

Revoke is **not admin-only**. Both admins and members may call it from
a dashboard session, but scope differs:

- **Admins** — may revoke any key in the organization.
- **Members** — may revoke **only keys they created themselves**. A
  member targeting another user's key gets `404` (same status as a
  missing key, to prevent `key_id` enumeration).
- **API keys (`sk_…`)** — always rejected with `403`.

## Webhook secret storage

The webhook signing secret is stored in **plaintext** on the server
(it has to be — Tumban computes the HMAC on every outbound webhook).
This contrasts with API keys, which are kept only as SHA-256 hashes.

Because the secret is shown to you exactly once at rotation time and
never reappears in `GET /api/v2/org/settings`, **there is no recovery
path** — capture it from the rotation response immediately. If you
lose it, rotate again and update every verifier in lockstep before
sending any more outbound traffic that would expect the old signature.

## Scan lifecycle

```
POST /api/v2/scan
  → { scan_id, status: "processing", submitted_at, estimated_completion }

(async)
  → scan runs server-side (timeout: 450 s)
  → on terminal status, Tumban POSTs to callback_url (when configured)
  → result also queryable via GET /api/v2/scans/{scan_id}
```

`status` values:

- `processing` — in flight.
- `completed` — terminal; triage report available. May reflect a
  pipeline where some steps were skipped (slow page, login wall,
  transient model error). The `coverage` object records what actually
  ran (e.g. `social_links_checked`, `blocked_by_login`). Read
  `coverage` to detect partial pipelines; do not gate partial-handling
  logic on the `status` field.
- `failed` — terminal; `error` field set.

### `is_banned`

`GET /api/v2/scans/{scan_id}` also returns a top-level `is_banned`
field (`bool | null`). `null` until the ban-checker has evaluated the
profile; then `true` if the upstream platform has banned the creator
(404 / 410 / redirect on the profile URL) or `false` if the profile is
still live. Useful for skipping enforcement on already-banned creators.

## Confidence semantics

`confidence` ∈ `{high, medium, low}`:

- **`high`** — strong, corroborated evidence. Act on it directly.
- **`medium`** — solid signal, less corroboration. Useful, but a
  manual reviewer may want to spot-check on edge cases.
- **`low`** — a thin or borderline lead. Treat as review-worthy, not
  as a verdict.

`confidence` is **never `low` when `recommendation` is `no_flags`** —
clean profiles always come back with `high` confidence in the
`no_flags` decision. This combination cannot occur.

How Tumban arrives at each level is **not part of the public contract**
and may change without notice. Build against `confidence`,
`risk_score`, `recommendation`, `reason_codes`, and `evidence_index`;
ignore any other field on the response.

## Score → recommendation bands

| Range | Recommendation |
|-------|----------------|
| 0–10 | `no_flags` |
| 11–40 | `review_low` |
| 41–60 | `review_medium` |
| 61–100 | `review_high` |

`risk_score` reflects confidence that a policy violation is present
(not uncertainty); missing or unreachable data does not push the score
up — partial coverage surfaces in the `coverage` object, never as
inflated risk.

The four enum values above are **frozen API contract**. The score
thresholds may be tuned over time as detection improves; the enum
strings will not change. How the score is produced internally is not
part of the contract.

## Rate limits

When an org has a `daily_scan_limit` configured, exceeding it returns
`429` with a **structured** detail body (not a string):

```json
{
  "detail": {
    "error": "daily_scan_limit_exceeded",
    "limit": 1000,
    "used": 1000
  }
}
```

- Counter resets at `00:00 UTC`.
- Batch submissions have a **partial-acceptance** path: when remaining
  capacity is less than the requested batch size, the batch is accepted
  with the leading N profiles and the response sets
  `daily_limit_truncated: true` plus `profiles_skipped: <int>`. Only a
  zero-capacity submission returns `429`.
- Tumban does **not** return `X-RateLimit-*` headers and does **not**
  honour `Idempotency-Key`. Use the returned `scan_id` (or per-profile
  scan ids in a batch) as the natural idempotency key in your handler.

## Webhook delivery and verification

Tumban POSTs JSON to your `callback_url` when a scan reaches a terminal
status. Up to 3 attempts; backoff `1s` then `2s`. `Retry-After` is
honoured when numeric (decimal seconds allowed, e.g. `2.5`); HTTP-date
form is not parsed.

### Signed headers (when org has a webhook secret)

| Header | Meaning |
|--------|---------|
| `X-Tumban-Signature` | `sha256=<hex>` over the raw body bytes (V1). |
| `X-Tumban-Signature-V2` | `sha256=<hex>` over `"{timestamp}.{org_id}." + body` (raw bytes concatenation). **Recommended.** |
| `X-Tumban-Timestamp` | Unix seconds when the payload was signed. |
| `X-Tumban-Org-Id` | The `org_id` this webhook is for. Treat as opaque. Under rare error paths Tumban may send an empty value (`""`) — V2 verifiers reject these because they will not match `EXPECTED_ORG_ID`. |

A correct V2 verifier performs three checks:

1. Tenant binding — `X-Tumban-Org-Id` matches the receiver's expected
   `org_id`.
2. Replay protection — `X-Tumban-Timestamp` is within ~5 minutes of now.
3. Constant-time signature compare — never use `==` on the hex digest.

## Common workflows

### Submit a single scan and read the result via webhook

1. `POST /api/v2/scan` with `profile_url`, `callback_url`,
   optional `metadata` (any JSON, echoed back).
2. Receive webhook at `callback_url`. Verify the V2 signature first;
   then process `recommendation`, `risk_score`, `evidence_index`, and
   `coverage`.
3. Acknowledge with any `2xx` status. Non-2xx triggers retry.

### Submit a batch and watch aggregate progress

1. `POST /api/v2/batch` with `profile_urls`. Per-profile webhooks fire
   independently as each scan completes.
2. Poll `GET /api/v2/batches/{batch_id}` for aggregate counts
   (`completed`, `failed`, `in_progress`).
3. If `daily_limit_truncated: true` is set on the submission response,
   the trailing `profiles_skipped` URLs were not queued — resubmit
   them after the next `00:00 UTC`.

### Set up webhook signature verification

1. `POST /api/v2/org/webhook-secret/rotate` from a dashboard admin
   session. Capture `webhook_secret` from the response (shown once).
2. Store the secret in your secret manager alongside the `org_id` your
   receiver expects.
3. Implement V2 verification — see `https://docs.tumban.com/webhooks/signatures`
   for reference verifiers in Python / Node / Ruby.
4. Roll out the verifier **before** the secret is actively used by
   senders. Tumban switches signing to the new secret immediately on
   rotation; old secrets become inactive at the same instant.

### Rotate an API key without downtime

1. `POST /api/v2/org/api-keys` to create a new key. Capture the raw
   `sk_…` value (shown once).
2. Deploy the new key. Both keys are valid simultaneously.
3. Once traffic has cut over (watch `last_used_at` on the old key via
   `GET /api/v2/org/api-keys`), call
   `DELETE /api/v2/org/api-keys/{old_key_id}` from a dashboard session
   to revoke. There is no auto-expiry.

## Gotchas worth surfacing to a user

- **Partial pipelines surface in `coverage`, not in `status`.** A
  scan with step failures still reports `status: "completed"` —
  read the `coverage` object to see what ran.
- **Webhook secret is plaintext server-side.** Treat any leak as a
  full compromise of webhook authenticity; rotate immediately.
- **Revoke is `DELETE`, not `POST` + `/revoke`.** And the path has no
  `/revoke` suffix — see Endpoint details above.
- **Webhook rotate path includes `/org/`.** `/api/v2/webhook-secret/rotate`
  (no `/org/`) does not exist.
- **No `X-RateLimit-*` headers, no `Idempotency-Key`.** Plan around
  the structured 429 body and `scan_id` as natural idempotency key.
- **`org_id` is opaque.** Do not regex against hex or any specific
  alphabet. Compare for exact equality.
- **The `confidence` field is never `low` when `recommendation` is
  `no_flags`.** This combination cannot occur.

## See also

- Full reference docs: `https://docs.tumban.com`
- Status values: `https://docs.tumban.com/reference/status`
- Reason codes: `https://docs.tumban.com/reference/reason-codes`
- Webhook signature verifiers: `https://docs.tumban.com/webhooks/signatures`
- Errors and rate-limit detail: `https://docs.tumban.com/api/errors`
