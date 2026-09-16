---
title: CORE API - Create Order
type: api-component
domain: core-api
tags:
  - core-api
  - actions
  - moves
  - orders
  - vehicles
  - pickup
  - dropoff
  - bulk
related:
  - "[[CORE API]]"
  - "[[CORE API - Authentication]]"
  - "[[Move]]"
  - "[[Vehicle]]"
  - "[[Plan]]"
  - "[[Organization]]"
  - "[[Webhook]]"
---

# CORE API - Create Order

## Overview

In the DRAIVER API, an **Action** represents a request for work to be performed.

For vehicle logistics, the main use case is moving a vehicle from a **Pick Up** location to a **Drop off** location.

An Action is a request to move. It is not yet the execution itself.

```text
Action / Move
      ↓
Planning
      ↓
Plan / Trip
      ↓
Itinerary
      ↓
Driver / Actor
      ↓
Execution
```

A [[Plan]] / Trip can contain one or more driver Itineraries, and each Itinerary can contain one or more vehicle Moves.

For example:

```text
Plan / Trip
│
├── Driver Itinerary 1
│   ├── Move 1
│   └── Move 2
│
├── Driver Itinerary 2
│   ├── Move 3
│   ├── Move 4
│   └── Move 5
│
└── ...
```

After a Move is created, it progresses through a lifecycle of statuses until it is completed. Status can be monitored either by querying the API or by configuring [[Webhook]] notifications.

---

# Create an Action / Move

## Endpoint

```text
POST /v2/actions
```

Authentication is performed using the Bearer token described in [[CORE API - Authentication]].

Example:

```bash
curl --location 'http://core.prod.appservice.draiver.net/v2/actions'   --header 'Authorization: Bearer <apiToken>'   --header 'Content-Type: application/json'
```

---

# Request Body

The main fields are:

| Field | Required | Description |
|---|---:|---|
| `referenceId` | No | External reference such as order number, unit number or external GUID |
| `actionType` | Yes | Type of action. `MOVE_VEHICLES` is currently the valid value documented |
| `assigned` | No | Text containing driver assignment information |
| `pickup` | Yes | Pick Up location, time window, tasks and notes |
| `dropoff` | Yes | Drop off location, time window, tasks and notes |
| `vehicle` | Yes | Vehicle information |
| `reason` | No | Reason category and label |
| `notes` | No | General notes associated with the Action |

---

# Action Type

The documented valid `actionType` is:

```text
MOVE_VEHICLES
```

This represents a Vehicle Move.

Related: [[Move]]

---

# Reference ID

`referenceId` is an optional external identifier.

It can represent, for example:

- Customer order number.
- Unit number.
- External GUID.
- Another identifier maintained by the integrating system.

Example:

```json
{
  "referenceId": "Your Reference ID"
}
```

This is useful for correlating a DRAIVER Action with the corresponding record in the customer's system.

---

# Pickup

The `pickup` object defines where the Vehicle is collected.

It can contain:

- Location.
- Time window.
- Tasks.
- Notes.

Example:

```json
{
  "pickup": {
    "point": {
      "mapPointType": "ADDRESS",
      "street1": "123 Main St",
      "city": "Anytown",
      "state": "State",
      "postalCode": "12345"
    },
    "windowUTC": {
      "start": "2023-01-01T09:00:00Z",
      "end": "2023-01-01T12:00:00Z"
    },
    "tasks": [
      {
        "taskType": "TO_DO",
        "note": "Check vehicle condition",
        "noteMimeType": "TEXT_PLAIN"
      }
    ]
  }
}
```

Related: [[Pick Up]]

---

# Dropoff

The `dropoff` object defines where the Vehicle is delivered.

It can contain:

- Location.
- Time window.
- Tasks.
- Notes.

Example:

```json
{
  "dropoff": {
    "point": {
      "mapPointType": "ADDRESS",
      "street1": "456 Elm St",
      "city": "Othertown",
      "state": "State",
      "postalCode": "67890"
    },
    "windowUTC": {
      "start": "2023-01-01T14:00:00Z",
      "end": "2023-01-01T16:00:00Z"
    },
    "tasks": [
      {
        "taskType": "TO_DO",
        "note": "Leave keys in drop box",
        "noteMimeType": "TEXT_PLAIN"
      }
    ]
  }
}
```

Related: [[Drop off]]

---

# Locations

A location can be provided using either:

1. A street address.
2. GPS coordinates.

The platform internally resolves the location.

```text
Address
   ↓
Geocoding
   ↓
Location + GPS

GPS
   ↓
Reverse geocoding
   ↓
Location + Address
```

---

## Address-Based Location

Example:

```json
{
  "point": {
    "mapPointType": "ADDRESS",
    "street1": "123 Main St",
    "city": "Anytown",
    "state": "State",
    "postalCode": "12345"
  }
}
```

When an address is supplied, the resulting GPS coordinates may differ slightly from the exact coordinates associated with the original address.

The documentation states that the platform attempts to resolve the location to the nearest available road or route using Google Maps data.

---

## GPS-Based Location

For a GPS-based location, use:

```text
mapPointType = GPS
```

and provide latitude and longitude.

Example:

```json
{
  "label": "replace me",
  "point": {
    "mapPointType": "GPS",
    "latitude": 45.334047,
    "longitude": -75.351948
  }
}
```

When GPS coordinates are supplied, the platform performs reverse geocoding to determine the corresponding address and location details.

---

# Requested Date / Time

The pickup and dropoff objects contain a `windowUTC`.

Example:

```json
{
  "windowUTC": {
    "start": "2023-01-01T09:00:00Z",
    "end": "2023-01-01T12:00:00Z"
  }
}
```

The documentation states that the Requested Date can be empty or `null`. In that case, the Action date becomes the current UTC instance time.

> The source documentation describes this as "Instance now UTC (ARG + 3)". This should be treated as source terminology and verified against the actual environment if timezone behavior matters to an integration.

---

# Vehicle

The `vehicle` object contains information about the Vehicle associated with the Move.

Possible information includes:

- `make`
- `model`
- `year`
- `vin`

Example:

```json
{
  "vehicle": {
    "make": "CHEVROLET",
    "model": "Cruze",
    "year": 2014,
    "vin": "1G1PA5SG0E7174385"
  }
}
```

Related: [[Vehicle]]

---

# VIN Validation

The VIN is the primary Vehicle identifier according to the documentation.

The documented validation rules are:

- VIN must be a 17-character alphanumeric code.
- A VIN shorter than 17 characters is not treated as a valid VIN.
- A 17-character value containing only numbers is not considered valid.
- A 17-character value containing only letters is not considered valid.

---

## Incomplete VIN

If the VIN is shorter than 17 characters:

```text
Incomplete VIN
     ↓
Action / Move is still created
     ↓
Value becomes part of Pick Up instructions
     ↓
VIN is not treated as a valid VIN
```

---

## Invalid 17-Character VIN

If the value has 17 characters but consists only of numbers or only of letters:

```text
17-character invalid VIN
        ↓
Action / Move is NOT created
```

---

## No VIN

If no VIN is provided:

```json
{
  "vehicle": {}
}
```

The Action can still be created and the Vehicle is marked as **Unknown**.

---

# Vehicle Without VIN

If the integration provides only year, make and model:

```json
{
  "vehicle": {
    "year": "2014",
    "make": "CHEVROLET",
    "model": "Cruze"
  }
}
```

The documentation states:

- If year, make and model are complete, the Action / Move is created.
- The Vehicle is marked as Unknown.
- The supplied information becomes part of the Pick Up notes/instructions.

If year, make and model are incomplete:

- The Action / Move is still created.
- The Vehicle is marked as Unknown.
- The supplied information becomes part of the Pick Up notes/instructions.

If no Vehicle information is supplied:

```json
{
  "vehicle": {}
}
```

the Vehicle is Unknown.

---

# Vehicle Requirements

Vehicle requirements can be sent as boolean values.

The documentation states that:

- `true` means the requirement is needed.
- Requirements are not mandatory.
- One or more requirements can be selected.

The exact list of available requirement fields is not included in the supplied documentation.

---

# Reason

The `reason` object describes the reason for the Action.

It contains:

```json
{
  "reason": {
    "category": "RETAIL",
    "label": "Your Label"
  }
}
```

---

# Notes

There are two general contexts for notes:

1. **Dispatcher / General Notes** — information applicable to the overall Action.
2. **Vehicle Notes** — instructions associated with pickup or dropoff.

Related: [[Integration Terminology]]

---

## General / Dispatcher Notes

General notes apply to the overall Action.

Example:

```json
{
  "noteType": "GENERAL",
  "category": "GENERAL",
  "message": "We need this order complete ASAP",
  "noteMimeType": "TEXT_PLAIN"
}
```

Typical uses include:

- Explaining the reason for a Move.
- Providing general context.
- Sharing high-level instructions.

---

## Pickup Notes

Pickup notes contain instructions specific to collecting the Vehicle.

Example:

```json
{
  "noteType": "GENERAL",
  "category": "PICKUP_INSTRUCTIONS",
  "message": "talk to mike at the warehouse, he has the keys",
  "noteMimeType": "TEXT_PLAIN"
}
```

---

## Dropoff Notes

Dropoff notes contain instructions specific to delivering the Vehicle.

Example:

```json
{
  "noteType": "GENERAL",
  "category": "DROPOFF_INSTRUCTIONS",
  "message": "leave keys at counter",
  "noteMimeType": "TEXT_PLAIN"
}
```

---

# Complete Create Action Example

```json
{
  "referenceId": "Your Reference ID",
  "actionType": "MOVE_VEHICLES",
  "pickup": {
    "point": {
      "mapPointType": "ADDRESS",
      "street1": "123 Main St",
      "city": "Anytown",
      "state": "State",
      "postalCode": "12345"
    },
    "windowUTC": {
      "start": "2023-01-01T09:00:00Z",
      "end": "2023-01-01T12:00:00Z"
    },
    "tasks": [
      {
        "taskType": "TO_DO",
        "note": "Check vehicle condition",
        "noteMimeType": "TEXT_PLAIN"
      }
    ]
  },
  "dropoff": {
    "point": {
      "mapPointType": "ADDRESS",
      "street1": "456 Elm St",
      "city": "Othertown",
      "state": "State",
      "postalCode": "67890"
    },
    "windowUTC": {
      "start": "2023-01-01T14:00:00Z",
      "end": "2023-01-01T16:00:00Z"
    },
    "tasks": [
      {
        "taskType": "TO_DO",
        "note": "Leave keys in drop box",
        "noteMimeType": "TEXT_PLAIN"
      }
    ]
  },
  "vehicle": {
    "make": "Make",
    "model": "Model",
    "year": "Year",
    "vin": "VIN"
  },
  "reason": {
    "category": "RETAIL",
    "label": "Your Label"
  },
  "notes": [
    {
      "noteType": "GENERAL",
      "message": "General notes about the action"
    }
  ]
}
```

---

# Create Action Response

The create endpoint returns an Action representation.

Representative response:

```json
{
  "id": "a1ODv000005LrF7MAK",
  "referenceId": "My Reference",
  "actionType": "MOVE_VEHICLES",
  "externalReportingId": "Optional non-unique string",
  "quoteId": "Optional provided id from Create Quote",
  "status": "UNPLANNED",
  "planSummary": {
    "actions": [],
    "classifications": [],
    "tags": []
  },
  "requestor": {
    "label": "Big Jay"
  },
  "createdAt": "2023-09-11T14:19:13Z",
  "ready": "2023-09-11T14:19:12Z",
  "deadline": "2023-09-13T14:19:12Z",
  "updatedOn": "2023-09-11T14:19:13Z"
}
```

The response also contains the resolved pickup, dropoff, vehicle, reason and notes information.

---

# Initial Status

A newly created Action can have:

```text
UNPLANNED
```

This is consistent with the concept that creating an Action is a **request to move**, not yet the execution of that Move.

```text
Create Action
      ↓
UNPLANNED
      ↓
Planning
      ↓
Plan / Trip
      ↓
Execution
```

---

# Tracking an Action

After creation, the Move progresses through statuses until completion.

The documentation identifies two ways to track these changes:

### Polling

Use the appropriate GET endpoint to retrieve the current Action state.

### Webhooks

Configure [[Webhook]] subscriptions to receive status/event notifications.

Conceptually:

```text
Action
  │
  ├── GET → Current state
  │
  └── Webhook → Event notification
```

---

# Bulk Creation

CORE API also supports creating multiple Actions in one request for a specific child [[Organization]].

## Endpoint

```text
POST /v2/actions/bulk
```

The request contains:

- `organizationId`
- `actions`

Example:

```json
{
  "organizationId": "child-organization-id",
  "actions": [
    {
      "...": "action1"
    },
    {
      "...": "action2"
    }
  ]
}
```

Each Action follows the same general structure as an individual Action.

---

# Bulk Response

The bulk endpoint returns separate `success` and `errors` collections.

Example:

```json
{
  "success": [
    {
      "actionId": "a1OVE000000TFN32AO",
      "referenceId": "My Reference 2",
      "status": "UNPLANNED",
      "createdAt": "2024-09-03T14:17:23Z",
      "requestor": "Mascot Brazil",
      "index": 1
    }
  ],
  "errors": [
    {
      "errorMessage": "pickup.windowUTC: start can not be null pickup.windowUTC.start: must not be null",
      "index": 0
    }
  ]
}
```

The `index` allows the integrator to identify which input Action succeeded or failed.

This is particularly important when processing large batches because a validation error in one Action does not necessarily mean that all Actions failed.

---

# Bulk Integration Mental Model

```text
Input Actions
      ↓
POST /v2/actions/bulk
      ↓
┌───────────────┐
│   Processing  │
└───────┬───────┘
        │
   ┌────┴────┐
   ↓         ↓
Success    Errors
   │         │
   ↓         ↓
Action ID   Index
```

An integration should therefore inspect both collections rather than assuming that an HTTP-level success means every Action was successfully created.

---

# Integration Design Considerations

When building an integration that creates Moves through CORE API, the main mapping to establish is:

```text
External Order
      │
      ├── referenceId
      │
      ├── Vehicle
      │      └── VIN
      │
      ├── Pick Up
      │      ├── Location
      │      ├── Time Window
      │      └── Tasks / Notes
      │
      ├── Drop off
      │      ├── Location
      │      ├── Time Window
      │      └── Tasks / Notes
      │
      ├── Reason
      │
      └── General Notes
      │
      ▼
CORE API
      │
      ▼
Action / Move
      │
      ▼
Planning
```

The integration-specific mapping should document which external fields are mapped into each of these sections.

---

# Troubleshooting Checklist

When an Action is not created as expected:

```text
1. Is authentication valid?
        ↓
2. Is the correct Organization being used?
        ↓
3. Is actionType = MOVE_VEHICLES?
        ↓
4. Is pickup present?
        ↓
5. Is dropoff present?
        ↓
6. Are pickup/dropoff time windows valid?
        ↓
7. Is vehicle information valid?
        ↓
8. If VIN is provided, is it a valid 17-character alphanumeric VIN?
        ↓
9. Are required fields correctly formatted?
        ↓
10. For bulk requests, which input index failed?
```

For a created Action that does not progress:

```text
Action created?
      ↓
Status = UNPLANNED?
      ↓
Sent to planning?
      ↓
Plan generated?
      ↓
Itinerary created?
      ↓
Actor assigned?
      ↓
Fulfillment started?
```

Related: [[Procurement]], [[Provisioning]], [[Fulfillment]].

---

# Documentation Gaps / Things to Verify

The supplied documentation does not fully define:

- Complete Action status lifecycle.
- Exact GET endpoints for Action status retrieval.
- Complete list of Vehicle Requirements.
- Complete list of `actionType` values beyond `MOVE_VEHICLES`.
- Complete Task schema.
- Exact validation rules for every Vehicle field.
- Exact behavior of `assigned`.
- Complete `reason.category` and `reason.label` values.
- Exact relationship between `externalReportingId` and `referenceId`.
- Complete Plan/Action relationship in the response.
- Exact timezone handling when Requested Date is null.
- Full error catalog.
- Rate limits for individual and bulk creation.
- Idempotency behavior when the same Action is submitted more than once.

These should be added as integration knowledge is discovered.

---

# Related Notes

- [[CORE API]]
- [[CORE API - Authentication]]
- [[Move]]
- [[Vehicle]]
- [[Organization]]
- [[Plan]]
- [[Procurement]]
- [[Provisioning]]
- [[Fulfillment]]
- [[Webhook]]
- [[Integration Terminology]]

# Tags

#core-api #actions #moves #orders #vehicles #pickup #dropoff #bulk
