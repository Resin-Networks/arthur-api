# Arthur Gate Pass API — Yard Integration Spec
**Status:** Draft for review
**Last updated:** 2026-09-29

---

## Overview

Arthur verifies a driver's identity before they arrive for pickup. Once the driver is authorized for the load, their phone shows an Arthur **gate pass**: a QR code tied to that driver's phone. This API lets a yard app scan the gate pass and confirm, before releasing the load, that the driver at the window is the verified driver. The yard app receives the pickup number and short-lived links to the driver's selfie and side-of-cab photos.

### How it works

1. A broker creates a verification for the load, and Arthur texts the driver a link.
1. The driver completes identity verification on their phone. Once they're authorized, the link shows their gate pass.
1. At the yard, the driver opens the link and shows the gate pass. The yard app scans the QR code.
1. The yard app sends the scanned contents to `POST /api/v1/yard/authorization`. A `200` means release the load. Any error means don't.
1. No gate pass, no load. A driver without one is sent back to get it.

The QR code refreshes every 30 seconds while it's on screen, so a screenshot or forwarded image stops working almost immediately. See [Expired passes](#expired-passes).

### Base URL

```
https://api.choosearthur.com
```

---

## Authentication

Requests to Arthur use API key authentication.

| Direction | Key | Issued by | Header |
|-----------|-----|-----------|--------|
| Yard app → Arthur | API key | Arthur | `X-Api-Key` |

### Key exchange

During onboarding, Arthur issues an API key to the yard app for requests to Arthur's API.

### Authenticating requests to Arthur

Include your API key in the `X-Api-Key` header on every request:

```
X-Api-Key: your_api_key_here
```

Call Arthur from your server. Don't ship the key inside the mobile app.

---

## Endpoints

### 1. Scan Gate Pass

```
POST /api/v1/yard/authorization
```

Looks up the verification behind a scanned gate pass.

#### Request Body

```json
{
  "qr_payload": "arthur:gp1:eyJ2aWQiOiJhMWIyYzNkNC1lNWY2LTc4OTAi..."
}
```

#### Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `qr_payload` | string | yes | The exact string decoded from the QR code. Send it unmodified — the format is Arthur's and may change without notice. |

#### Response — `200 OK`

```json
{
  "driver_name": "JOHN DOE",
  "authorization": {
    "pickup_number": "PU-88213",
    "reference": "ART-7F3K2Q"
  },
  "photos": [
    {
      "type": "selfie",
      "url": "https://storage.googleapis.com/arthur-images/...&X-Goog-Signature=...",
      "expires_at": "2026-09-29T15:55:03Z"
    },
    {
      "type": "cab_markings_photo",
      "url": "https://storage.googleapis.com/arthur-images/...&X-Goog-Signature=...",
      "expires_at": "2026-09-29T15:55:03Z"
    }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `driver_name` | string | The driver this verification is for, for display in your app. |
| `authorization.pickup_number` | string | The pickup number the shipper assigned this load. Match it against the pickup you're releasing. |
| `authorization.reference` | string | Arthur's reference for this release. Each scan issues a new one. Arthur logs the scan, so either side can look up when the release happened. |
| `photos` | array | Photos the driver took during verification, so staff can match the person and truck in front of them. |
| `photos[].type` | string | `selfie` or `cab_markings_photo` (who the driver should be, the markings that should be present on the side of the truck). |
| `photos[].url` | string | Signed link to the photo. |
| `photos[].expires_at` | string | ISO 8601 timestamp, 15 minutes after the scan. The link stops working after this. Display photos, never store these links or photos. |

#### Revoked passes

A broker can withdraw a driver's authorization after the gate pass was issued, e.g. when the load is cancelled or reassigned. The pass disappears from the driver's phone, and a scan of an older code returns `403 not_authorized`. Turn the driver away.

#### Expired passes

Each QR code is valid for 30 seconds after it appears on the driver's screen. A scan after that returns `410 pass_expired`. That's most likely a screenshot. Ask the driver to open their gate pass link on their own phone and scan the live code.

---

## Error Responses

All errors follow this format:

```json
{
  "detail": {
    "error": {
      "code": "pass_expired",
      "message": "Human-readable description of what went wrong"
    }
  }
}
```

**Treat every error as do not release.**

| HTTP Status | Code | Description |
|-------------|------|-------------|
| 400 | `invalid_request` | `qr_payload` missing, not an Arthur gate pass, or tampered with |
| 401 | `unauthorized` | Missing or invalid API key |
| 403 | `not_authorized` | The driver's authorization was withdrawn. See [Revoked passes](#revoked-passes) |
| 404 | `not_found` | The verification behind this pass no longer exists |
| 410 | `pass_expired` | The code is past its 30-second window. See [Expired passes](#expired-passes) |
| 429 | `rate_limited` | Too many requests - currently not implemented. Will be added later. |
| 500 | `internal_error` | Something went wrong on our end |
