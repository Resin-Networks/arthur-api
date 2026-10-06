# Arthur Loads API — TMS Integration Spec
**Status:** Draft for review
**Last updated:** 2026-10-02

---

## Overview

Arthur provides truck and driver identity verification for freight brokerages. This API lets a TMS declare a **load** and Arthur raises whatever work that load calls for, starting with an identity verification per driver. All driver PII stays inside Arthur. The TMS receives only status, never the underlying data.

Declaring a load is safe to repeat: a record-triggered flow can fire on every save without minting a duplicate load or re-texting a driver.

### How it works

1. TMS declares the load via `POST /api/v1/loads`. Arthur creates a verification for each declared driver and, by default, texts each of them a link.
1. Each driver opens their link on their phone and completes identity verification.
1. Arthur processes the documents and runs checks against whatever the load declared.
1. The TMS re-declares the load via `PUT /api/v1/loads/{load_id}` whenever the record changes — a corrected phone, a driver swap, a firmed-up appointment time.
1. The TMS tracks progress by polling `GET /api/v1/loads/{load_id}`, or by polling the change feed to pick up movement across every load at once.
1. If the load is cancelled, the TMS cancels it via `DELETE /api/v1/loads/{load_id}`, which closes every verification on it in one call.

A load carrying two drivers creates **one verification per driver**. See [Team loads](#team-loads).

### Base URL

```
https://api.choosearthur.com
```

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

---

## Endpoints

| | Endpoint | What it does |
|---|---|---|
| **POST** | `/api/v1/loads` | [Create a load](#create-a-load) — declares it and texts its drivers |
| **PUT** | `/api/v1/loads/{load_id}` | [Update a load](#update-a-load) — declares any changes |
| **GET** | `/api/v1/loads/{load_id}` | [Get a load](#get-a-load) — current state of every verification on it |
| **GET** | `/api/v1/loads/changes` | [Get status changes](#get-status-changes) — one feed across all your loads |
| **DELETE** | `/api/v1/loads/{load_id}` | [Cancel a load](#cancel-a-load) — closes every verification on it |

`POST` and `PUT` take the same body, and every endpoint returns the same load object — the change feed returns a page of them: [the load body](#the-load-body) and [the load response](#the-load-response).

---

## Endpoint details

### Create a load

```
POST /api/v1/loads
```

Declares a load and creates a verification per driver.

Idempotent on `load_number`. A caller that lost its `load_id`, or that retries, gets back the load it already has rather than a second one and a second text.

**Body:** [the load body](#the-load-body) · **Returns:** [the load response](#the-load-response)

---

### Update a load

```
PUT /api/v1/loads/{load_id}
```

Re-declares the load's current state. Whatever has changed is applied; whatever didn't is left alone.

This is the same upsert `POST` runs, so a record-triggered flow can fire on every save. An empty or omitted field means "not declared" and never overwrites what the load already holds.

Drivers are added, not swapped, unless you set `drivers_complete`. This can be useful if you intend to have verifications fire automatically when dispatch info changes. You can just always send the current dispatch information without worrying about cancelling prior verifications. More info: [The roster](#the-roster).

**Body:** [the load body](#the-load-body) · **Returns:** [the load response](#the-load-response)

---

### Get a load

```
GET /api/v1/loads/{load_id}
```

Returns the load's current state and where each of its verifications stands. Poll this to track progress.

**Returns:** [the load response](#the-load-response)

---

### Get status changes

```
GET /api/v1/loads/changes?since={timestamp}&limit={limit}
```

Returns the loads whose verifications moved after `since`, so a TMS can poll a single feed for movement across all of its loads instead of polling each one by id. Scoped to the calling organization — you only ever see your own.

#### Query parameters

| Param | Type | Required | Description |
|---|---|---|---|
| `since` | string (ISO 8601) | yes | Exclusive lower bound — only changes strictly after this timestamp are returned. On your first call, pass the time you last synced (or any time in the past). |
| `limit` | integer | no | Maximum number of status changes to read in one page, oldest change first. Defaults to 200. Several changes can belong to one load, so a page may return fewer loads than this. |

#### Response — `200 OK`

```json
{
  "changes": [
    {
      "load_id": "3f0c1d8e-9b21-4a77-b0e6-2a1f5c7d9e42",
      "load_number": "your-internal-id-123",
      "verifications": [
        {
          "verification_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
          "verification_url": "https://choosearthur.com/v/a1b2c3d4-e5f6-7890-abcd-ef1234567890",
          "driver_name": "JOHN DOE",
          "driver_phone": "+13125551234",
          "expires_at": "2026-05-20T06:30:00+00:00",
          "created_at": "2026-05-19T18:30:00+00:00",
          "updated_at": "2026-05-19T19:02:00+00:00",
          "first_texted_at": "2026-05-19T18:31:00+00:00",
          "status": "verified",
          "human_readable_status": "Verified"
        }
      ]
    }
  ],
  "cursor": "2026-05-19T19:02:00+00:00"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `changes` | array | The loads that moved since `since`, oldest change first. Each entry is [the load response](#the-load-response), carrying the load's full current state — not just the verification that moved. |
| `cursor` | string \| null | ISO 8601 timestamp — the latest change in this page. Hand it back as the next request's `since` to page forward. `null` when the page is empty; keep your previous `since` and poll again later. |

A team load appears once, with both drivers in its `verifications` array. A load whose two drivers both moved is one entry, not two.

#### Polling

Start with `since` set to your last sync time, then hand the returned `cursor` back as the next request's `since`. Each status change is returned once. When there's no movement the page is empty and `cursor` is `null`, so keep your prior `since` until the next poll.

---

### Cancel a load

```
DELETE /api/v1/loads/{load_id}
```

Cancels the load by closing every verification on it. A team load's two drivers included. Use this when the load is cancelled or the verifications are otherwise no longer needed. Each driver's link is invalidated, no further checks will run, and any text Arthur had scheduled won't be sent.

**Returns:** [the load response](#the-load-response), with `verifications` empty — the array lists live verifications, and cancelling closed them all.

---

## The load body

Sent to `POST /api/v1/loads` and `PUT /api/v1/loads/{load_id}`.

```json
{
  "load_number": "your-internal-id-123",
  "tms_record_id": "a0X5f000001AbCdEFG",
  "assigned_broker_email": "dispatcher@yourbrokerage.com",
  "equipment_type": "Straight Van",
  "load_weight_lbs": 8400,
  "gross_weight_lbs": 0,
  "carrier": {
    "name": "Acme Trucking LLC",
    "mc_number": "123456",
    "usdot_number": "1234567"
  },
  "truck": {
    "vin": "1XKYD49X0XR000001",
    "truck_number": "4412",
    "trailer_number": "T-88"
  },
  "drivers": [
    { "first_name": "JOHN", "last_name": "DOE", "phone": "+13125551234" }
  ],
  "drivers_complete": false,
  "stops": [
    {
      "kind": "pickup",
      "facility_name": "Des Plaines DC",
      "address": "1400 S Wolf Rd, Des Plaines, IL 60018",
      "city": "Des Plaines",
      "state": "IL",
      "lat": 41.8781,
      "lng": -87.6298,
      "date": "2026-05-20",
      "appointment_at": "08:30",
      "receiving_hours": "07:00-15:00",
      "appointment_required": true,
      "reference_numbers": ["PO-99812"]
    },
    {
      "kind": "delivery",
      "facility_name": "Charleston Cross-dock",
      "city": "Charleston",
      "state": "SC",
      "date": "2026-05-21"
    }
  ],
  "customer": {
    "name": "Widgets Inc",
    "address": "500 Industrial Pkwy, Charleston, SC",
    "phone": "+18435550100"
  }
}
```

Only `load_number` and the `drivers` array are required, and a driver only needs a name and a phone. Send whatever your TMS has. Each additional field tightens a check Arthur runs or improves the timing of the driver's text.

| Field | Type | Required | Description |
|---|---|---|---|
| `load_number` | string | yes | Your internal identifier for the load. Identity: the load is keyed on (your org, this number), so re-sending it is an update, not a new load. 1–100 characters. |
| `tms_record_id` | string | no | Your record's primary key, when it differs from `load_number` (e.g. a Salesforce record id). Used to link the Arthur report back to the record in your system. |
| `assigned_broker_email` | string | no | The broker who owns the load. CC'd on the results email, and shown as the load's owner in the Arthur dashboard. |
| `equipment_type` | string | no | What the shipment calls for, e.g. `"Straight Van"`, `"Hot Shot"`, `"Reefer 53'"`. Used to decide which license classes the load accepts — loads on non-CDL equipment accept Class D licenses too. See [Equipment and license class](#equipment-and-license-class). |
| `load_weight_lbs` | integer | no | Freight weight. `0` means undeclared. Only speaks for non-CDL equipment — see [Equipment and license class](#equipment-and-license-class). |
| `gross_weight_lbs` | integer | no | Gross vehicle weight. `0` means undeclared. Outranks `equipment_type` in both directions. |
| `carrier.name` | string | no | Carrier legal name. Compared to driver-provided data. |
| `carrier.mc_number` | string | no | Carrier MC number. Compared to driver-provided data. |
| `carrier.usdot_number` | string | no | Carrier USDOT number. Compared to driver-provided data, and screened against Arthur's do-not-use list of carriers reported for fraud. |
| `truck.vin` | string | no | Truck VIN. Compared to driver-provided data. Arthur retains only the last four characters. |
| `truck.truck_number` | string | no | Fleet-assigned power-unit number painted on the cab. Compared against the number Arthur reads off the photo of the truck. |
| `truck.trailer_number` | string | no | Trailer number. Recorded on the load. |
| `drivers` | array | yes | 1–4 entries. See [`drivers[]`](#drivers) below. |
| `drivers_complete` | boolean | no | Defaults to `false`. See [The roster](#the-roster). |
| `stops` | array | no | Pickups and deliveries in route order. See [`stops[]`](#stops) below. |
| `customer.name` | string | no | The party the load is hauled for. |
| `customer.address` | string | no | Customer address. |
| `customer.phone` | string | no | Customer phone. |
| `requested_steps` | array | no | Steps to run on this load beyond the standing plan your organization is configured with. Values: `"verification"`, `"proof_of_delivery"`. Verification runs on every load, so this is rarely needed. |

### `drivers[]`

A driver entry names one driver, or hands Arthur the slash-separated fields a TMS keeps a team in. Send one shape or the other:

| Field | Type | Description |
|---|---|---|
| `name` | string | Full name. Use this **or** `first_name`/`last_name`. |
| `first_name` | string | First name. Collapsed with `last_name` into `name`. |
| `last_name` | string | Last name. |
| `phone` | string | Driver mobile phone. Required because Arthur runs a VoIP fraud check on this number — only true mobile lines pass. SMS is also sent here. E.164 preferred; common US formats (`843-555-2426`) are accepted and normalized. |
| `names_raw` | string | Slash-separated names, e.g. `"Larry Jones / Joyce Jones"`. Sent **together** with `phones_raw`. See [Team loads](#team-loads). |
| `phones_raw` | string | Slash-separated phones, e.g. `"843-809-8540 / 843-809-8541"`. |

An entry needs a name and a phone, or both raw fields — sending `names_raw` without `phones_raw` is a `422`. The two shapes can share one load: one structured entry plus one crammed entry is fine, and the crammed one expands in place.

The **phone is what a verification is keyed on.** A declared phone that matches a verification already on the load updates it; a phone Arthur hasn't seen on this load raises a new one.

### `stops[]`

| Field | Type | Description |
|---|---|---|
| `kind` | string | `"pickup"` or `"delivery"`. The driver is verified against the first `pickup`; with no stop labelled, the first stop is used. |
| `facility_name` | string | Facility name. |
| `address` | string | Free-form address. Arthur geocodes it for the GPS comparison, so it's a good substitute when you don't have lat/lng. |
| `city` | string | City. |
| `state` | string | Two-letter state abbreviation. Also sets the time zone a wall-clock time reads in. |
| `lat` | number | Latitude. Compared to the driver's GPS at submission. |
| `lng` | number | Longitude. |
| `contact_phone` | string | Facility contact. |
| `window_start` | string (ISO 8601) | The window's start as an instant. Outranks `date`, `appointment_at` and `receiving_hours` — send it when your TMS already has a resolved timestamp. |
| `window_end` | string (ISO 8601) | The window's end. |
| `date` | string | The day the pieces below anchor to, e.g. `"2026-05-20"`. |
| `appointment_at` | string | `"09:30"` or a full timestamp. Outranks `receiving_hours`. |
| `receiving_hours` | string | The facility's window, e.g. `"07:00-15:00"`. Used when no appointment time is set. |
| `appointment_set` | string | `"Y"`, `"N"`, or empty. |
| `appointment_required` | boolean | Facility policy. |
| `pieces` | string | Verbatim, e.g. `"16 PALLETS"`. |
| `weight` | string | Verbatim. |
| `reference_numbers` | array of strings | Order / PO / pickup / BOL numbers. |
| `special_instructions` | string | Free text. |

**On timing.** A stop keeps its window as an instant, but a TMS usually holds a date, an appointment time and receiving hours separately. Send those three and Arthur assembles the window, reading a wall-clock time in the stop's own zone (from `state`, or from `address` when no state is declared). The pickup window is what times the driver's text.

---

## The load response

Returned by create, update, get and cancel.

```json
{
  "load_id": "3f0c1d8e-9b21-4a77-b0e6-2a1f5c7d9e42",
  "load_number": "your-internal-id-123",
  "verifications": [
    {
      "verification_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "verification_url": "https://choosearthur.com/v/a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "driver_name": "JOHN DOE",
      "driver_phone": "+13125551234",
      "expires_at": "2026-05-20T06:30:00+00:00",
      "created_at": "2026-05-19T18:30:00+00:00",
      "updated_at": "2026-05-19T18:30:00+00:00",
      "first_texted_at": "",
      "status": "pending",
      "human_readable_status": "Pending"
    }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `load_id` | string | Arthur's id for the load. **Store this** — it addresses the load on every subsequent call. |
| `load_number` | string | Echo of the `load_number` you supplied. |
| `verifications` | array | Every live verification on the load — one per driver. A cancelled verification is not listed. |

### `verifications[]`

| Field | Type | Description |
|-------|------|-------------|
| `verification_id` | string | Arthur's id for this driver's verification. |
| `verification_url` | string | The link this driver opens to complete verification. Each driver gets their own. |
| `driver_name` | string | The driver this verification is for. |
| `driver_phone` | string | Normalized to E.164, so you can see what Arthur keyed on. |
| `expires_at` | string | ISO 8601. See [Expiry](#expiry). |
| `created_at` | string | ISO 8601 timestamp of when the verification was created. |
| `updated_at` | string | ISO 8601 timestamp of when the status last changed. |
| `first_texted_at` | string | ISO 8601 timestamp of the first text to this driver, or `""` if none has gone out yet. |
| `status` | string | Current status. See the [Statuses](#statuses) table. |
| `human_readable_status` | string | Display-ready label for `status`, safe to render directly in your UI. |

`status` can be `verified` on creation if Arthur already recognizes this driver from a recent verification for your organization — in that case no text is sent and the load is good to release immediately.

**Verification ids are stable across updates.** A `PUT` that changes nothing about a driver returns the same `verification_id`. Read `first_texted_at` if you want to track how long we have been waiting for the driver to respond. 

---

## The roster

`drivers` is **additive by default.** A call naming one driver is adding them to the load. If you want verifications to be manually requested this allows multiple verifications to be appended to the same load. Driver name changes won't result in a new verification or new text to the driver but correct the name for whatever is associated with the phone number. 

Set **`drivers_complete: true`** to say outright that `drivers` is the load's whole roster. Then any live verification for a driver left off it is cancelled. A verification that already reached a terminal state is never cancelled this way. `drivers_complete` with an empty `drivers` array won't cancel outstanding verifications and is treated as a mistake. 

Use `drivers_complete` when your TMS record is authoritative about who is on the load, usually if you are firing automatically off of dispatch information changes. 

---

## Team loads

A load with two drivers creates **one verification per driver**. Each driver is texted their own link, completes their own identity check, and resolves to their own status. All of them are listed in the load's `verifications` array.

### Two structured drivers

The cleanest shape if possible with your setup. 

```json
{
  "load_number": "your-internal-id-123",
  "drivers": [
    { "name": "Larry Jones", "phone": "843-809-8540" },
    { "name": "Joyce Jones", "phone": "843-809-8541" }
  ]
}
```

### One crammed field

We can also handle drivers sent in the same field. Send them as `names_raw`/`phones_raw` and Arthur will split them:

```json
{
  "load_number": "your-internal-id-123",
  "drivers": [
    {
      "names_raw": "Larry Jones / Joyce Jones",
      "phones_raw": "843-809-8540 / 843-809-8541"
    }
  ]
}
```

Names and phones are paired by position: the first name gets the first phone. When the counts don't match, Arthur can't pair them safely and declares the raw fields as a **single** driver rather than guessing.

### Working with a team load

- **Track both.** Both verifications are on the load's `verifications` array, whether you poll the load or read the change feed.
- **Release on both.** The load is good to release only when every verification on it is `verified`. Arthur does not roll them up into a single load-level status.
- **Cancel once.** `DELETE /api/v1/loads/{load_id}` closes every verification on the load, both drivers included.

---

## Statuses

**Implementations should accept any string in the `status` field.**

| Status | `human_readable_status` | Meaning |
|---|---|---|
| `pending` | "Pending" | Verification created. Waiting for the driver to open the link and start the flow. |
| `in_progress` | "In Progress" | Driver is going through the verification process, or is retaking a photo Arthur asked for again. |
| `processing` | "Processing" | The driver has submitted and the verification is being run. |
| `in_review` | "In Review" | Checks completed and something needs a human look. Not terminal — an Arthur or brokerage reviewer resolves it to `verified` or `failed`. Do not release the load on this status. |
| `verified` | "Verified" | **Terminal.** Checks passed, load is good to be released. Also returned at creation for a driver Arthur already recognizes. |
| `failed` | "Failed" | **Terminal.** Identity could not be confirmed and the load should not be released to this driver. |
| `cancelled` | "Cancelled" | **Terminal.** Verification was cancelled — via `DELETE`, by `drivers_complete`, or dismissed in the Arthur dashboard. A cancelled verification drops off the load's `verifications` array. |
| `expired` | "Expired" | **Terminal.** The driver never completed the flow. See [Expiry](#expiry). |

### Expiry

`expires_at` is returned as 12 hours after the verification was created, and is the window in which the driver is expected to complete the flow.

The link itself is not hard-cut at that timestamp — a driver who is running late can still finish. A verification the driver never completes moves to `expired` 48 hours after the text was sent. Treat `expires_at` as the point to follow up, and `status: "expired"` as the point the verification is closed.

---

## Equipment and license class

`equipment_type` and the two weights decide which license classes the load will accept. The default is CDL-only.

- A declared `gross_weight_lbs` outranks everything: under 26,001 lbs the load accepts any license class, at or above it a CDL is required.
- With no gross declared, a **non-CDL equipment** value — `"Straight Van"`, `"Straight Truck 26'"`, `"Sprinter/Cargo Van"`, `"Hot Shot"`, `"Partial Van"`, `"Flatbed Hotshot"` and their variants — accepts any license class, unless `load_weight_lbs` is 11,000 lbs or more, which puts it back on a CDL.

---

## Errors

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
| 404 | `not_found` | Load ID does not exist, or belongs to another organization |
| 422 | `validation_error` | Field validation failed — a driver missing a name or phone, `names_raw` without `phones_raw`, a negative weight, a body renaming the load, or a load with no load number |
| 429 | `rate_limited` | Too many requests — currently not implemented. Will be added later. |
| 500 | `internal_error` | Something went wrong on our end |
