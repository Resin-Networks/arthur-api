# Arthur Verification API — TMS Integration Spec
**Status:** Draft for review
**Last updated:** 2026-05-05

---

## Overview

Arthur provides driver identity verification for freight brokerages. This API allows a TMS to programmatically initiate verifications and receive results via webhook or polling. All driver PII stays inside Arthur. The TMS receives only verification status, never the underlying data.

### How it works

1. TMS creates a verification via `POST /api/v1/verify` with optional match criteria. By default, drivers are texted a link to complete their verification.
1. The driver opens the link on their phone and completes identity verification.
1. Arthur processes the documents, runs checks against any match criteria provided, and fires a **webhook** to the TMS as the verification moves through processing stages.
1. The TMS can also poll status on demand.

### Environments

| Environment | Base URL | Purpose |
|-------------|----------|---------|
| Sandbox | `https://sandbox.api.choosearthur.com` | Testing — no data saved, configurable outcomes |
| Production | `https://api.choosearthur.com` | Live verifications |

---

## Authentication

Both directions (TMS → Arthur and Arthur → TMS) use API key authentication.

| Direction | Key | Issued by | Header |
|-----------|-----|-----------|--------|
| TMS → Arthur | API key | Arthur | `X-Api-Key` |
| Arthur → TMS (webhooks) | Webhook secret | TMS | `X-Webhook-Secret` |

### Key exchange

During onboarding, Arthur issues an API key to the TMS for requests to Arthur's API. The webhook secret is supplied by the TMS per-request via the `webhook_secret` field on the create-verification body — there is no onboarding-time exchange.

### Authenticating requests to Arthur

Include your API key in the `X-Api-Key` header on every request:

```
X-Api-Key: your_api_key_here
```

### Verifying webhooks from Arthur

Arthur returns the `webhook_secret` you supplied on the create-verification request as the `X-Webhook-Secret` header on every webhook delivery for that verification. Compare with constant-time comparison and reject non-matches with a non-2xx response.

---

## Endpoints

### 1. Create Verification

```
POST /api/v1/verify
```

Creates a verification session. By default, Arthur sends an SMS to the driver with the verification link. Set `suppress_driver_sms: true` to deliver the link yourself.

#### Request Body

```json
{
  "callback_url": "https://your-tms.com/webhooks/arthur",
  "webhook_secret": "whsec_xxx",
  "load_id": "your-internal-id-123",
  "suppress_driver_sms": false,
  "location_match_details": {
    "lat": 41.8781,
    "lng": -87.6298,
    "acceptable_radius_miles": 1
  },
  "driver_match_details": {
    "first_name": "JOHN",
    "last_name": "DOE",
    "phone": "+13125551234"
  },
  "truck_match_details": {
    "vin": "1XKYD49X0XR000001",
    "usdot_number": "1234567"
  },
  "sandbox_options": {
    "force_status": "verified"
  }
}
```

#### Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `callback_url` | string (URL) | yes | Where Arthur POSTs webhooks for this verification. |
| `webhook_secret` | string | no | If provided, Arthur returns this as the `X-Webhook-Secret` header on every webhook so you can verify origin. |
| `load_id` | string | yes | Your internal identifier for the load. Echoed back on every webhook and GET. |
| `suppress_driver_sms` | boolean | no, default `false` | If `true`, Arthur will not text the driver. Use when your TMS delivers the link via its own channel. |
| `location_match_details.lat` | number | no | Pickup latitude. Driver's GPS at submission must fall within `acceptable_radius_miles`. |
| `location_match_details.lng` | number | no | Pickup longitude. |
| `location_match_details.acceptable_radius_miles` | number | no | Allowed distance between pickup and the driver's reported location. Defaults to 1 mile. |
| `driver_match_details.first_name` | string | yes | Driver first name. Used to make the verification text friendly for the driver. Compared to the parsed CDL. |
| `driver_match_details.last_name` | string | no | Driver last name. Used to make the verification text friendly for the driver. Compared to the parsed CDL. |
| `driver_match_details.phone` | string (E.164) | yes | Driver mobile phone. Required because Arthur runs a VoIP fraud check on this number. SMS is also sent here unless `suppress_driver_sms` is `true`. |
| `truck_match_details.vin` | string | no | Truck VIN. Compared to the side of truck and parsed cab card. |
| `truck_match_details.usdot_number` | string | no | Carrier USDOT number. Compared to the side of truck and parsed cab card. |
| `sandbox_options.force_status` | enum | no | Sandbox testing only. One of `pending`, `processing`, `verified`, `failed`. Resolves the verification deterministically without requiring driver photos. |

#### Response — `200 OK`

```json
{
  "verification_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "load_id": "your-internal-id-123",
  "verification_url": "https://choosearthur.com/v/a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "expires_at": "2026-03-23T18:30:00Z",
  "status": "pending",
  "human_readable_status": "Pending"
}
```

`verification_url` is the link the driver opens to complete verification. When `suppress_driver_sms` is `true`, deliver this URL to the driver through your own channel.
`status` can be `verified` or `failed` on initial creation if Arthur's systems can identify them.

| Field | Type | Description |
|-------|------|-------------|
| `verification_id` | string | Arthur's unique ID for this verification. |
| `load_id` | string | Echo of the `load_id` you supplied on the request. Always returned. |
| `verification_url` | string | Link to send to the driver. Driver opens this on their phone to begin. |
| `expires_at` | string | ISO 8601 timestamp. Link expires 12 hours after creation. |
| `status` | string | Current status. Usually `pending`; may be `verified` or `failed` if Arthur can resolve immediately. |
| `human_readable_status` | string | Display-ready label for `status`, safe to render directly in your UI. See the Statuses table for the mapping. |

---

### 2. Webhook — Status Updates

Arthur POSTs to your `callback_url` whenever the verification's status changes.

```
POST {your_callback_url}
```

#### Webhook Payload

```json
{
  "verification_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "load_id": "your-internal-id-123",
  "status": "verified",
  "human_readable_status": "Verified",
  "updated_at": "2026-03-23T14:30:00Z"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `verification_id` | string | Arthur's ID for this verification. |
| `load_id` | string | Echo of the `load_id` you supplied on the request. |
| `status` | string | Current status. See the Statuses table below. |
| `human_readable_status` | string | Display-ready label for `status`, safe to render directly in your UI. |
| `updated_at` | string | ISO 8601 timestamp of when this status was set. |

#### Delivery semantics

- **Delivered on every status change.** A single verification typically produces several webhooks as it moves through statuses, e.g. `pending` → `processing` → `verified`. Treat each call as the latest known state for that `verification_id`.
- **Status may repeat across deliveries** (e.g. on retry). If your handler is non-idempotent, dedupe on `(verification_id, status, updated_at)`.
- **Retries.** If your endpoint returns non-2xx or times out, Arthur retries with backoff (1 min, 5 min, 30 min). After three failed attempts the webhook is dropped; use `GET /verifications/{id}` to reconcile.
- **Acknowledge with 2xx.** Any 2xx response is treated as delivered. Response body is ignored.

#### Statuses

**Implementations should accept any string in the `status` field** and only switch on values they recognize, falling through unknown values as "still in progress."

| Status | `human_readable_status` | Meaning |
|---|---|---|
| `pending` | "Pending" | Verification created. Waiting for the driver to open the link and start the flow. |
| `in_progress` | "In Progress" | Driver is going through the verification process. |
| `processing` | "Processing" | The verification is being run. |
| `in_review` | "In Review" | Driver's verification is in review. |
| `verified` | "Verified" | **Terminal.** All checks passed, load is good to be released. |
| `failed` | "Failed" | **Terminal.** Identity could not be confirmed and the load should not be released to this driver. |

---

### 3. Get Verification Status

```
GET /api/v1/verifications/{verification_id}
```

Returns current status. Use this for polling if webhooks are not feasible, or as a fallback.

#### Response — `200 OK`

```json
{
  "verification_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "load_id": "your-internal-id-123",
  "status": "verified",
  "human_readable_status": "Verified",
  "created_at": "2026-03-23T14:00:00Z",
  "updated_at": "2026-03-23T14:30:00Z"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `verification_id` | string | Arthur's ID for this verification. |
| `load_id` | string | Echo of the `load_id` you supplied on the request. Always returned. |
| `status` | string | Current status. See the Statuses table above. |
| `human_readable_status` | string | Display-ready label for `status`, safe to render directly in your UI. |
| `created_at` | string | ISO 8601 timestamp of when the verification was created. |
| `updated_at` | string | ISO 8601 timestamp of when the status last changed. |

---

## Error Responses

All errors follow this format:

```json
{
  "detail": {
    "error": {
      "code": "validation_error",
      "message": "Human-readable description of what went wrong"
    }
  }
}
```

| HTTP Status | Code | Description |
|-------------|------|-------------|
| 400 | `invalid_request` | Missing or malformed fields |
| 401 | `unauthorized` | Missing or invalid API key |
| 404 | `not_found` | Verification ID does not exist |
| 422 | `validation_error` | Field validation failed (e.g., invalid lat/lng) |
| 429 | `rate_limited` | Too many requests - currently not implemented. Will be added later. |
| 500 | `internal_error` | Something went wrong on our end |
