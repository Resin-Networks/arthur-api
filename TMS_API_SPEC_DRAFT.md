# Arthur Verification API — TMS Integration Spec (DRAFT)

**Status:** Draft for review — not yet implemented
**Last updated:** 2026-03-23

---

## Overview

Arthur provides driver identity verification for freight brokerages. This API allows a TMS to programmatically initiate verifications, receive results via webhook, and review reports.

### How it works

1. **TMS creates a verification** via `POST /api/v1/verify` with optional match criteria
2. Arthur returns a verification link — the TMS delivers this link to the driver (via their own messaging, dispatch app, etc.)
3. The driver opens the link on their phone and completes identity verification
4. Arthur processes the documents, runs checks against any match criteria provided, and fires a **webhook** to the TMS with the result
5. The TMS can also poll status or retrieve the report on demand

### Environments

| Environment | Base URL | Purpose |
|-------------|----------|---------|
| Sandbox | `https://sandbox.api.choosearthur.com` | Testing — no data saved, configurable outcomes |
| Production | `https://api.choosearthur.com` | Live verifications |

---

## Authentication

All requests require a signature header for authenticity verification:

```
X-Arthur-Signature: sha256=<hmac_hex_digest>
```

Compute `HMAC-SHA256` of the raw request body using the api key provided to us by TransportPro and compare to the signature header.

> **Open question:** TransportPro mentioned using JWT. We're proposing HMAC-SHA256 request signing (above) — which does your team prefer? Options:
> - **HMAC-SHA256 signing** — shared secret, signature on each request (spec as written)
> - **JWT bearer token** — TransportPro issues a JWT, Arthur validates against your JWKS endpoint
> - **Something else?**

---

## Endpoints

### 1. Create Verification

```
POST /api/v1/verify
```

Creates a verification session. Returns a link to send to the driver. The TMS is responsible for delivering the link to the driver.

#### Request Body

All `*_match_details` fields are optional. If provided, Arthur will match the driver's submitted documents against the provided values and include match results in the report.

```json
{
  "callback_url": "https://your-tms.com/webhooks/arthur",

  "reference_id": "your-internal-id-123",

  "location_match_details": {
    "lat": 41.8781,
    "lng": -87.6298,
    "acceptable_radius_miles": 1 // Defaults to 1 mile if not provided, floats are accepted
  },

  "driver_match_details": {
    "given_names": "JOHN",
    "family_name": "DOE",
    "date_of_birth": "1970-01-01",
    "document_id": "R111-1111-1111",
    "address": "123 Main St, Chicago, IL 60639",
    "expiration_date": "2027-01-01",
    "issue_date": "2022-01-01"
  },

  "carrier_match_details": {
    "organization": "MIDWEST FREIGHT SOLUTIONS LLC",
    "address": "4200 W. Diversey Ave., Chicago, IL 60639"
  },

  "truck_match_details": {
    "make": "Kenworth",
    "model_year": "1998",
    "vin": "1XKYD49X0XR000001",
    "plate_number": "AB 1234",
    "usdot_number": "1234567"
  },

  "sandbox_options": {
    "force_status": "verified"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `callback_url` | string | Yes | Webhook URL — Arthur will POST results here when verification completes |
| `reference_id` | string | No | Your internal identifier — returned in webhook for correlation |
| `location_match_details` | object | No | Expected driver location. If provided, driver's GPS is checked against this point |
| `location_match_details.lat` | number | Conditional | Latitude (-90 to 90) |
| `location_match_details.lng` | number | Conditional | Longitude (-180 to 180) |
| `location_match_details.acceptable_radius_miles` | number | No | Radius in miles. Default: `2` |
| `driver_match_details` | object | No | Expected CDL details. Any subset of fields can be provided |
| `driver_match_details.given_names` | string | No | First/middle name(s) as they appear on CDL |
| `driver_match_details.family_name` | string | No | Last name as it appears on CDL |
| `driver_match_details.date_of_birth` | string | No | Format: `YYYY-MM-DD` |
| `driver_match_details.document_id` | string | No | CDL number |
| `driver_match_details.address` | string | No | Address on CDL |
| `driver_match_details.expiration_date` | string | No | Format: `YYYY-MM-DD` |
| `driver_match_details.issue_date` | string | No | Format: `YYYY-MM-DD` |
| `carrier_match_details` | object | No | Expected carrier info from cab card |
| `carrier_match_details.organization` | string | No | Carrier/company name |
| `carrier_match_details.address` | string | No | Carrier address |
| `truck_match_details` | object | No | Expected truck info from cab card |
| `truck_match_details.make` | string | No | Vehicle make (e.g., "Kenworth") |
| `truck_match_details.model_year` | string | No | Vehicle model year |
| `truck_match_details.vin` | string | No | Vehicle identification number |
| `truck_match_details.plate_number` | string | No | License plate number |
| `truck_match_details.usdot_number` | string | No | USDOT number |
| `sandbox_options` | object | No | **Sandbox only.** Ignored in production. |
| `sandbox_options.force_status` | string | No | Force the verification outcome: `"verified"` or `"requires_review"`. Immediately completes the verification and triggers the webhook with boilerplate data. |

#### Response — `201 Created`

```json
{
  "verification_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "verification_url": "https://verify.choosearthur.com/v/a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "expires_at": "2026-03-25T12:00:00Z",
  "reference_id": "your-internal-id-123"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `verification_id` | string | Arthur's unique ID for this verification |
| `verification_url` | string | Link to send to the driver. Driver opens this on their phone to begin. |
| `expires_at` | string | ISO 8601 timestamp. Link expires 48 hours after creation. |
| `reference_id` | string | Echo of your reference ID, if provided |

#### Sandbox Behavior

- Webhook fires with boilerplate report data matching the forced status
- No real document processing or photo analysis occurs
- The `report_url` links to a sample report page
- The `report_pdf_url` returns a sample PDF
- Sandbox does not retain any submitted data beyond the session lifetime

---

### 2. Webhook — Verification Result

When verification completes, Arthur sends a `POST` to the `callback_url` you provided.

```
POST {your_callback_url}
```

#### Webhook Payload

```json
{
  "verification_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "reference_id": "your-internal-id-123",
  "status": "verified",
  "report_url": "https://verify.choosearthur.com/report/a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "report_pdf_url": "https://api.choosearthur.com/api/v1/verifications/a1b2c3d4-e5f6-7890-abcd-ef1234567890/report.pdf",
  "report_pdf_base64": "<base64-encoded PDF bytes>",
  "completed_at": "2026-03-23T14:30:00Z"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `verification_id` | string | Arthur's ID for this verification |
| `reference_id` | string \| null | Your internal ID, if provided at creation |
| `status` | string | `"verified"` or `"requires_review"` (see below) |
| `report_url` | string | Link to Arthur's hosted report page (includes photos for human review) |
| `report_pdf_url` | string | Direct link to download the PDF report (authenticated, no photos) |
| `report_pdf_base64` | string | Base64-encoded PDF report included directly in the webhook payload (no photos) |
| `completed_at` | string | ISO 8601 timestamp of when processing finished |

> **PDF delivery:** We can provide the report as a URL (`report_pdf_url`), as raw base64 data inline in the webhook (`report_pdf_base64`), or both. Let us know which you prefer — inline data means one fewer HTTP call but larger payloads.

#### Status Values

| Status | Meaning | Recommended TMS Action |
|--------|---------|----------------------|
| `verified` | All checks passed. Driver identity is confirmed. | Proceed with load assignment |
| `requires_review` | One or more checks flagged an issue. The driver has **not** been alerted. | Broker/ops should review the report before proceeding. Treat with caution. |

> **Note:** The driver never sees pass/fail results. They always see a generic "Verification Underway" screen. If a verification comes back `requires_review`, the driver does not know — the broker can decide how to handle it.

#### Webhook Verification

Webhooks include a provided api key for authenticity verification:

```
X-API-Key: our_api_key_here
```

TransportPro to provide api key to Arthur.


#### Retry Policy

If your endpoint returns a non-2xx status, Arthur will retry:
- 3 retries with exponential backoff (1 min, 5 min, 30 min)
- After all retries fail, the result is still available via the GET endpoint

---

### 3. Get Verification Status (Optional Polling)

```
GET /api/v1/verifications/{verification_id}
```

Returns current status. Can be used for polling if webhooks are not feasible, or as a fallback.

#### Response — `200 OK`

```json
{
  "verification_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "reference_id": "your-internal-id-123",
  "status": "processing",
  "created_at": "2026-03-23T12:00:00Z",
  "completed_at": null,
  "report_url": null,
  "report_pdf_url": null
}
```

| Status | Meaning |
|--------|---------|
| `pending` | Session created, driver has not started |
| `processing` | Documents submitted, checks are running |
| `verified` | Complete — all checks passed |
| `requires_review` | Complete — flagged for review |
| `expired` | Driver did not complete within 48 hours |

---

### 4. Get Report PDF

```
GET /api/v1/verifications/{verification_id}/report.pdf
```

Returns the verification report as a PDF. Requires API key authentication. The report includes check results and match details but **does not include photos** — photos are only viewable on the hosted report page (`report_url`).

Returns `404` if the verification is not yet complete.

---

## Error Responses

All errors follow this format:

```json
{
  "error": {
    "code": "rate_limited",
    "message": "Human-readable description of what went wrong"
  }
}
```

| HTTP Status | Code | Description |
|-------------|------|-------------|
| 400 | `invalid_request` | Missing or malformed fields |
| 401 | `unauthorized` | Missing or invalid API key |
| 404 | `not_found` | Verification ID does not exist |
| 422 | `validation_error` | Field validation failed (e.g., invalid lat/lng) |
| 429 | `rate_limited` | Too many requests |
| 500 | `internal_error` | Something went wrong on our end |

---

## Open Questions

1. **Authentication method** — HMAC-SHA256 request signing or JWT bearer tokens? We'd prefer to share a token issued by Transpro and use HMAC signing to obscure it in traffic.
2. **PDF delivery** — Do you want the report PDF inline in the webhook payload (base64), as a download URL, or both?
3. **Link expiration** — Default is 48 hours. Would a shorter expiration window work for you and if so, how short?
4. **Additional match fields** — We currently support the driver, carrier, and truck fields listed above. Are there any details you collect on your end that we could compare that you don't see here?
