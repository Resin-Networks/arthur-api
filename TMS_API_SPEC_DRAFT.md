# Arthur Verification API — TMS Integration Spec
**Status:** Draft for review
**Last updated:** 2026-08-19

---

## Overview

Arthur provides driver identity verification for freight brokerages. This API allows a TMS to programmatically initiate verifications, cancel them, and check their status. All driver PII stays inside Arthur. The TMS receives only verification status, never the underlying data.

### How it works

1. TMS creates a verification via `POST /api/v1/verify` with optional match criteria. By default, drivers are texted a link to complete their verification.
1. The driver opens the link on their phone and completes identity verification.
1. Arthur processes the documents and runs checks against any match criteria provided.
1. The TMS tracks progress either by polling a single verification via `GET /api/v1/verifications/{id}`, or by polling the change feed via `GET /api/v1/verifications/changes` to pick up status movement across all of its verifications at once.
1. If the verification is no longer needed (e.g. load cancelled), the TMS can cancel it via `DELETE /api/v1/verifications/{id}`.

A load carrying two drivers (a team load) creates **one verification per driver**. See [Team loads](#team-loads).

### Base URL

```
https://api.choosearthur.com
```

To try the API during integration, create a verification against production using your own mobile number as `driver_match_details.phone`. You'll receive the driver text and can walk the flow end to end.

---

## Authentication

Requests to Arthur use API key authentication.

| Direction | Key | Issued by | Header |
|-----------|-----|-----------|--------|
| TMS → Arthur | API key | Arthur | `X-Api-Key` |

### Key exchange

During onboarding, Arthur issues an API key to the TMS for requests to Arthur's API.

### Authenticating requests to Arthur

Include your API key in the `X-Api-Key` header on every request:

```
X-Api-Key: your_api_key_here
```

Your key identifies your organization, and every endpoint is scoped to it. You can only read, poll, or cancel verifications your own key created; another organization's verification IDs return `404`.

---

## Endpoints

### 1. Create Verification

```
POST /api/v1/verify
```

Creates a verification session.

#### Request Body

```json
{
  "load_id": "your-internal-id-123",
  "tms_record_id": "a0X5f000001AbCdEFG",
  "assigned_broker_email": "dispatcher@yourbrokerage.com",
  "skip_if_exists": true,
  "location_match_details": {
    "lat": 41.8781,
    "lng": -87.6298,
    "city": "Chicago",
    "state": "IL",
    "address": "1400 S Wolf Rd, Des Plaines, IL 60018",
    "pickup_window_start_at": "2026-05-20T08:00:00-05:00",
    "appointment_at": "2026-05-20T08:30:00-05:00",
    "shipping_hours": "07:00-15:00",
    "appointment_required": true
  },
  "driver_match_details": {
    "first_name": "JOHN",
    "last_name": "DOE",
    "phone": "+13125551234"
  },
  "carrier_match_details": {
    "name": "Acme Trucking LLC",
    "mc_number": "123456",
    "usdot_number": "1234567"
  },
  "truck_match_details": {
    "equipment": "Straight Van",
    "vin": "1XKYD49X0XR000001",
    "truck_number": "4412"
  }
}
```

Every field except `load_id` and the two required driver fields is optional. Send whatever your TMS has — each additional field either tightens a match Arthur runs against the driver's documents or improves the timing of the driver's text.

#### Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `load_id` | string | yes | Your internal identifier for the load. Echoed back on every GET. |
| `tms_record_id` | string | no | Your record's primary key, when it differs from `load_id` (e.g. a Salesforce record id). Used to link the Arthur report back to the record in your system. |
| `assigned_broker_email` | string | no | The broker who owns the load. CC'd on the results email, and shown as the load's owner in the Arthur dashboard. |
| `skip_if_exists` | boolean | no | Defaults to `false`. When `true`, if a verification already exists for this `load_id` and driver phone, Arthur returns it instead of creating a second one and re-texting the driver. Recommended for any caller that may retry. |
| `location_match_details.lat` | number | no | Pickup latitude. Compared to the driver's GPS at submission. |
| `location_match_details.lng` | number | no | Pickup longitude. |
| `location_match_details.city` | string | no | Pickup city. Used when lat/lng are not available. |
| `location_match_details.state` | string | no | Pickup state (two-letter abbreviation). Used alongside `city`. |
| `location_match_details.address` | string | no | Free-form ship-from address. Arthur geocodes it to coordinates for the GPS comparison, so it's a good substitute when you don't have lat/lng. |
| `location_match_details.pickup_window_start_at` | string (ISO 8601) | no | Start of the pickup window. Used to time the text message sending window. |
| `location_match_details.appointment_at` | string (ISO 8601) | no | Scheduled appointment time at the pickup. Takes precedence over `shipping_hours` when timing the driver's text. |
| `location_match_details.shipping_hours` | string | no | The pickup's receiving window, e.g. `"07:00-15:00"`. Used to time the text when no appointment time is set. |
| `location_match_details.appointment_required` | boolean | no | Whether the pickup requires an appointment. Recorded on the load. |
| `driver_match_details.first_name` | string | yes | Driver first name. Used to make the verification text friendly for the driver. Compared to the parsed CDL. |
| `driver_match_details.last_name` | string | no | Driver last name. Used to make the verification text friendly for the driver. Compared to the parsed CDL. |
| `driver_match_details.phone` | string | yes | Driver mobile phone. Required because Arthur runs a VoIP fraud check on this number — only true mobile lines pass. SMS is also sent here. E.164 preferred; common US formats (`843-555-2426`) are accepted and normalized. |
| `carrier_match_details.name` | string | no | Carrier legal name. Compared to driver provided data. |
| `carrier_match_details.mc_number` | string | no | Carrier MC number. Compared to driver provided data. |
| `carrier_match_details.usdot_number` | string | no | Carrier USDOT number. Compared to driver provided data, and screened against Arthur's do-not-use list of carriers reported for fraud. |
| `truck_match_details.equipment` | string | no | Type of equipment used for the load. ex: "Straight Van", "Hot Shot". Compared to truck and used to validate license class. A value containing "team" also tells Arthur the load has two drivers — see [Team loads](#team-loads). More values can easily be mapped in our system. |
| `truck_match_details.vin` | string | no | Truck VIN. Compared to driver provided data. Arthur retains only the last four characters. |
| `truck_match_details.truck_number` | string | no | Fleet-assigned power-unit number painted on the cab. Compared against the number Arthur reads off the photo of the truck. |

#### Response — `200 OK`

```json
{
  "verification_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "load_id": "your-internal-id-123",
  "verification_url": "https://choosearthur.com/v/a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "expires_at": "2026-03-23T18:30:00Z",
  "status": "pending",
  "human_readable_status": "Pending",
  "verifications": [
    {
      "verification_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "verification_url": "https://choosearthur.com/v/a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "driver_name": "JOHN DOE",
      "expires_at": "2026-03-23T18:30:00Z",
      "status": "pending",
      "human_readable_status": "Pending"
    }
  ]
}
```

`verification_url` is the link the driver opens to complete verification.
`status` can be `verified` on initial creation if Arthur already recognizes this driver from a recent verification for your organization — in that case no text is sent and the load is good to release immediately.

| Field | Type | Description |
|-------|------|-------------|
| `verification_id` | string | Arthur's unique ID for this verification. Mirrors `verifications[0]`. |
| `load_id` | string | Echo of the `load_id` you supplied on the request. Always returned. |
| `verification_url` | string | Link to send to the driver. Driver opens this on their phone to begin. Mirrors `verifications[0]`. |
| `expires_at` | string | ISO 8601 timestamp. See [Expiry](#expiry). |
| `status` | string | Current status. Usually `pending`; may be `verified` if Arthur can resolve immediately. Mirrors `verifications[0]`. |
| `human_readable_status` | string | Display-ready label for `status`, safe to render directly in your UI. See the Statuses table for the mapping. |
| `verifications` | array | **Every verification this request created — one entry per driver.** Always present, and always has at least one entry. See below. |

#### `verifications[]`

| Field | Type | Description |
|-------|------|-------------|
| `verification_id` | string | Arthur's ID for this driver's verification. Use it for `GET` and `DELETE`. |
| `verification_url` | string | The link this driver opens. Each driver gets their own. |
| `driver_name` | string | The driver this verification is for. Use it to label the verification in your UI when a load has more than one. |
| `expires_at` | string | ISO 8601 timestamp. |
| `status` | string | Current status for this driver. |
| `human_readable_status` | string | Display-ready label for this driver's `status`. |

**Read the `verifications` array, not just the top-level fields.** A single-driver load returns one entry and the top-level fields mirror it, so integrations written against the top-level fields keep working. A team load returns two, and only the first is mirrored to the top level — an integration that ignores the array will silently track one driver and miss the other.

---

### Team loads

A load with two drivers creates **one verification per driver**. Each driver is texted their own link, completes their own identity check, and resolves to their own status. Both share the `load_id` you supplied.

Arthur treats a load as a team load when `truck_match_details.equipment` contains the word "team" (case-insensitive) — e.g. `"Dry Van Team 53'"`, `"Reefer 53' - Team"`, `"Van w/Team"`.

#### Supplying two drivers

TMS systems generally carry both team drivers in the same driver fields, separated by a slash, so that's what Arthur accepts:

```json
{
  "load_id": "your-internal-id-123",
  "driver_match_details": {
    "first_name": "Larry Jones / Joyce Jones",
    "phone": "843-809-8540 / 843-809-8541"
  },
  "truck_match_details": {
    "equipment": "Dry Van Team 53'"
  }
}
```

Names and phones are paired by position: the first name gets the first phone.

> **Put both full names in `first_name` and leave `last_name` empty.** Arthur joins `first_name` and `last_name` into a single string before splitting on `/`, so splitting the names across both fields (`first_name: "Larry / Joyce"`, `last_name: "Jones / Jones"`) produces three name parts against two phones. When the number of names and phones doesn't match, Arthur can't pair them safely and falls back to creating a **single** verification for the load rather than guessing.

#### Response — `200 OK`

```json
{
  "verification_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "load_id": "your-internal-id-123",
  "verification_url": "https://choosearthur.com/v/a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "expires_at": "2026-03-23T18:30:00Z",
  "status": "pending",
  "human_readable_status": "Pending",
  "verifications": [
    {
      "verification_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "verification_url": "https://choosearthur.com/v/a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "driver_name": "Larry Jones",
      "expires_at": "2026-03-23T18:30:00Z",
      "status": "pending",
      "human_readable_status": "Pending"
    },
    {
      "verification_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
      "verification_url": "https://choosearthur.com/v/b2c3d4e5-f6a7-8901-bcde-f12345678901",
      "driver_name": "Joyce Jones",
      "expires_at": "2026-03-23T18:30:00Z",
      "status": "pending",
      "human_readable_status": "Pending"
    }
  ]
}
```

#### Working with a team load

- **Store every `verification_id`.** All subsequent calls are per verification, not per load.
- **Track both.** Poll each `verification_id`, or use the change feed — both drivers appear as separate entries carrying the same `load_id`.
- **Release on both.** The load is only good to release when every verification on it is `verified`. Arthur does not roll the two up into a single load-level status.
- **Cancel both.** `DELETE` takes one `verification_id`. Cancelling one driver's verification does not cancel the other's — call it once per verification.

---

### 2. Get Verification Status

```
GET /api/v1/verifications/{verification_id}
```

Returns current status. Poll this endpoint to track verification progress.

#### Response — `200 OK`

```json
{
  "verification_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "load_id": "your-internal-id-123",
  "status": "verified",
  "human_readable_status": "Verified",
  "verification_url": "https://choosearthur.com/v/a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "expires_at": "2026-03-23T18:30:00Z",
  "created_at": "2026-03-23T14:00:00Z",
  "updated_at": "2026-03-23T14:30:00Z"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `verification_id` | string | Arthur's ID for this verification. |
| `load_id` | string | Echo of the `load_id` you supplied on the request. Always returned. On a team load, both verifications return the same `load_id`. |
| `status` | string | Current status. See the Statuses table below. |
| `human_readable_status` | string | Display-ready label for `status`, safe to render directly in your UI. |
| `verification_url` | string | Link the driver opens to complete verification. Same value returned at creation. |
| `expires_at` | string | ISO 8601 timestamp. See [Expiry](#expiry). |
| `created_at` | string | ISO 8601 timestamp of when the verification was created. |
| `updated_at` | string | ISO 8601 timestamp of when the status last changed. |

#### Statuses

**Implementations should accept any string in the `status` field**

| Status | `human_readable_status` | Meaning |
|---|---|---|
| `pending` | "Pending" | Verification created. Waiting for the driver to open the link and start the flow. |
| `in_progress` | "In Progress" | Driver is going through the verification process, or is retaking a photo Arthur asked for again. |
| `processing` | "Processing" | The driver has submitted and the verification is being run. |
| `in_review` | "In Review" | Checks completed and something needs a human look. Not terminal — an Arthur or brokerage reviewer resolves it to `verified` or `failed`. Do not release the load on this status. |
| `verified` | "Verified" | **Terminal.** Checks passed, load is good to be released. Also returned at creation for a driver Arthur already recognizes. |
| `failed` | "Failed" | **Terminal.** Identity could not be confirmed and the load should not be released to this driver. |
| `cancelled` | "Cancelled" | **Terminal.** Verification was cancelled — by the TMS via `DELETE`, or dismissed in the Arthur dashboard. |
| `expired` | "Expired" | **Terminal.** The driver never completed the flow. See [Expiry](#expiry). |

#### Expiry

`expires_at` is returned as 12 hours after the verification was created, and is the window in which the driver is expected to complete the flow.

The link itself is not hard-cut at that timestamp — a driver who is running late can still finish. A verification the driver never completes moves to `expired` 48 hours after the text was sent. Treat `expires_at` as the point to follow up, and `status: "expired"` as the point the verification is closed.

---

### 3. Get Verification Status Changes

```
GET /api/v1/verifications/changes?since={timestamp}&limit={limit}
```

Returns the verifications whose status changed after `since`, so a TMS can poll a single feed for status movement across all of its verifications instead of polling each one by id. Scoped to the calling TMS — you only ever see your own verifications.

#### Query Parameters

| Param | Type | Required | Description |
|---|---|---|---|
| `since` | string (ISO 8601) | yes | Exclusive lower bound — only changes strictly after this timestamp are returned. On your first call, pass the time you last synced (or any time in the past). |
| `limit` | integer | no | Maximum number of changes to return in one page, oldest change first. Defaults to 200. |

#### Response — `200 OK`

```json
{
  "changes": [
    {
      "verification_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "load_id": "your-internal-id-123",
      "status": "verified",
      "human_readable_status": "Verified",
      "verification_url": "https://choosearthur.com/v/a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "expires_at": "2026-03-23T18:30:00Z",
      "created_at": "2026-03-23T14:00:00Z",
      "updated_at": "2026-03-23T14:30:00Z"
    }
  ],
  "cursor": "2026-03-23T14:30:00Z"
}
```

Each entry in `changes` is the same object returned by **Get Verification Status**. A team load's two drivers appear as two separate entries sharing one `load_id`, and move independently.

| Field | Type | Description |
|-------|------|-------------|
| `changes` | array | Verifications that changed since `since`, oldest change first. Each entry has the same fields as the Get Verification Status response. |
| `cursor` | string \| null | ISO 8601 timestamp — the latest change in this page. Hand it back as the next request's `since` to page forward. `null` when the page is empty; keep your previous `since` and poll again later. |

#### Polling

Start with `since` set to your last sync time, then hand the returned `cursor` back as the next request's `since`. Each status change is returned once. When there's no movement the page is empty and `cursor` is `null`, so keep your prior `since` until the next poll.

---

### 4. Cancel Verification

```
DELETE /api/v1/verifications/{verification_id}
```

Cancels an in-flight verification. Use this when the underlying load is cancelled or the verification is otherwise no longer needed. The driver's link is invalidated, no further checks will run, and any text Arthur had scheduled for that driver won't be sent.

Cancelling a verification that is already in a terminal state (`verified`, `failed`, `cancelled`, or `expired`) is a no-op and returns the current state.

On a team load, this cancels one driver. Call it once per `verification_id` to cancel the whole load.

#### Response — `200 OK`

```json
{
  "verification_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "load_id": "your-internal-id-123",
  "status": "cancelled",
  "human_readable_status": "Cancelled",
  "verification_url": "https://choosearthur.com/v/a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "expires_at": "2026-03-23T18:30:00Z",
  "created_at": "2026-03-23T14:00:00Z",
  "updated_at": "2026-03-23T14:30:00Z"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `verification_id` | string | Arthur's ID for this verification. |
| `load_id` | string | Echo of the `load_id` you supplied on the request. Always returned. |
| `status` | string | Current status. Will be `cancelled` unless the verification was already in a terminal state. |
| `human_readable_status` | string | Display-ready label for `status`, safe to render directly in your UI. |
| `verification_url` | string | Link the driver opens to complete verification. Same value returned at creation. |
| `expires_at` | string | ISO 8601 timestamp. |
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
| 404 | `not_found` | Verification ID does not exist, or belongs to another organization |
| 422 | `validation_error` | Field validation failed (e.g., invalid lat/lng) |
| 429 | `rate_limited` | Too many requests - currently not implemented. Will be added later. |
| 500 | `internal_error` | Something went wrong on our end |
