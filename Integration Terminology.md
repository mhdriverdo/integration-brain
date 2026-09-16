---
title: Integration Terminology
type: glossary
domain: integrations
tags:
  - integrations
  - terminology
  - glossary
  - api
  - activity
  - move
  - trip
  - organization
related:
  - "[[Integration]]"
  - "[[Webhook]]"
  - "[[Inbound]]"
  - "[[Procurement]]"
  - "[[CORE API]]"
  - "[[Activity Payload]]"
---

# Integration Terminology

This note defines the terminology used throughout the DRAIVER integration ecosystem.

The purpose is to keep integration documentation consistent. Some terms may have slightly different names depending on the customer or API, but the concepts below should be treated as the canonical terminology.

---

## Move

A **Move** is a single vehicle relocation within a trip/plan.

It represents one discrete transportation action.

Depending on the context, a Move can also be referred to as:

- **Action**
- **Activity**
- **Order**

For integration purposes, these terms can refer to the same underlying business concept unless a specific integration defines a distinction.

### Example

```text
Move
├── Pick Up: Kansas City International Airport
├── Drop Off: NW Chipman Rd
└── Vehicle: VIN 19UUA56631A001163
```

Related: [[Activity Payload]]

---

## Plan / Trip

A **Plan** or **Trip** is a set of actions for a specific time period or route assigned to a driver, including all related movements.

A Plan can contain multiple Moves.

```text
Plan / Trip
│
├── Move 1
├── Move 2
├── Move 3
└── ...
```

Therefore:

> A **Move** represents one transportation action, while a **Plan/Trip** groups multiple actions into a larger operational unit.

---

## Receipt

A **Receipt** is a document generated after the completion of a Plan/Trip.

It contains the expenses associated with the trip and is used for:

- Cost tracking
- Billing
- Expense management

Conceptually:

```text
Plan / Trip
    ↓
Completion
    ↓
Receipt
    ↓
Expenses / Billing
```

---

## Pick Up

**Pick Up** is the location where the vehicle is collected.

In an Activity/Move payload, this is normally represented by the `pickup` object.

Example:

```json
{
  "pickup": {
    "mapPointType": "LOCATION",
    "mapPointId": "a1PVE000000eZyL2AU",
    "mapPointLabel": "Kansas City International Airport"
  }
}
```

The pickup can contain:

- Location ID
- Address
- GPS coordinates
- Time window
- Contact information
- Location notes
- Organization information

---

## Drop off

**Drop off** is the location where the vehicle is delivered.

In an Activity/Move payload, this is normally represented by the `dropoff` object.

Example:

```json
{
  "dropoff": {
    "mapPointType": "LOCATION",
    "mapPointId": "a1PVE000000edXJ2AY",
    "mapPointLabel": "NW Chipman Rd, KCMO, MO, USA"
  }
}
```

The dropoff can contain:

- Location ID
- Address
- GPS coordinates
- Time window
- Destination contact information
- Location notes
- Organization information

---

## Vehicle

A **Vehicle** is the vehicle associated with the Move / Activity / Order being managed through the API.

A vehicle can be identified by its **VIN**.

Example:

```json
{
  "vehicle": {
    "vin": "19UUA56631A001163",
    "operable": true
  }
}
```

The vehicle can also contain operational information such as:

- Whether it is operable.
- Pickup instructions.
- Dropoff instructions.

---

## Organization

An **Organization** is the organization or account that owns or manages the order, vehicle, or related activities.

Organizations are commonly identified by an `organizationId`.

Example:

```json
{
  "organizationId": "EewFloT0mkGY4QK0BeQ"
}
```

An organization can appear at different levels of an integration payload, including:

- The Activity / Move.
- Pickup location.
- Dropoff location.
- Webhook configuration.
- Webhook subscription.

This is important when working with multi-tenant integrations because the same integration infrastructure can process events belonging to different organizations.

---

# Activity Payload

An **Activity Payload** represents the information associated with an Activity / Move / Order.

The payload below is representative of the structure used by the API.

At a high level:

```text
Activity Payload
│
├── action
│   ├── organizationId
│   ├── startDateTimeUTC
│   ├── promiseDateTimeUTC
│   ├── fulfillmentStrategy
│   └── action
│       ├── actionType
│       ├── type
│       ├── requestor
│       ├── notes
│       ├── metadata
│       ├── pickup
│       ├── dropoff
│       ├── vehicle
│       ├── movementType
│       └── operator
│
└── hauler
    └── fareType
```

## Activity Payload Structure

### Action

The outer `action` contains the main information about the activity.

```json
{
  "organizationId": "...",
  "startDateTimeUTC": "...",
  "promiseDateTimeUTC": "...",
  "fulfillmentStrategy": "AI_OR_MANUAL",
  "action": {}
}
```

Important fields:

| Field | Description |
|---|---|
| `organizationId` | Organization associated with the activity |
| `startDateTimeUTC` | Activity start date/time in UTC |
| `promiseDateTimeUTC` | Promised completion date/time in UTC |
| `fulfillmentStrategy` | Strategy used to fulfill the action |
| `action` | Detailed Move / Activity information |

---

## Action Details

The nested `action` contains the operational definition of the Move.

```json
{
  "actionType": "MOVE_VEHICLE",
  "type": "MOVE_VEHICLES",
  "requestor": {},
  "notes": [],
  "metadata": {},
  "pickup": {},
  "dropoff": {},
  "vehicle": {},
  "movementType": "HAUL",
  "operator": {}
}
```

### Action Type

`actionType` describes the type of action being performed.

Example:

```text
MOVE_VEHICLE
```

### Type

`type` provides the more specific action classification.

Example:

```text
MOVE_VEHICLES
```

---

## Requestor

The `requestor` represents the party that requested the action.

Example:

```json
{
  "requestor": {
    "id": "003c000000omk6vAAA",
    "label": "Big Jay",
    "contact": {
      "id": "003c000000omk6vAAA",
      "firstName": "Big Jay"
    }
  }
}
```

---

## Notes

Notes provide additional information associated with the action.

Example:

```json
{
  "notes": [
    {
      "noteType": "GENERAL",
      "category": "GENERAL",
      "message": "We need this order complete ASAP",
      "noteMimeType": "TEXT_PLAIN"
    }
  ]
}
```

Pickup and dropoff can also contain their own instructions/notes.

---

## Pickup

The `pickup` object describes where the Vehicle is collected.

It can contain:

```text
pickup
├── mapPointType
├── mapPointId
├── mapPointLabel
├── windowUTC
├── position
├── location
│   ├── address
│   ├── gpsLocation
│   ├── timezone
│   └── contact information
└── organization
```

---

## Dropoff

The `dropoff` object describes where the Vehicle is delivered.

It follows a structure similar to `pickup` and may contain:

```text
dropoff
├── mapPointType
├── mapPointId
├── mapPointLabel
├── windowUTC
├── position
├── location
│   ├── address
│   ├── gpsLocation
│   ├── timezone
│   └── contact information
├── destinationPhone
└── organization
```

---

## Vehicle

The Vehicle section identifies the vehicle being moved.

Example:

```json
{
  "vehicle": {
    "vin": "19UUA56631A001163",
    "operable": true,
    "pickupNotes": [],
    "dropoffNotes": []
  }
}
```

---

## Movement Type

`movementType` describes how the vehicle movement is classified.

Example:

```text
HAUL
```

---

## Operator

The `operator` represents the entity responsible for performing the movement.

Example:

```json
{
  "operator": {
    "id": "operatorID",
    "name": "RunBuggy"
  }
}
```

---

## Hauler

The `hauler` section contains information associated with the transportation provider / hauler.

Example:

```json
{
  "hauler": {
    "fareType": "Gold"
  }
}
```

---

# Terminology Mapping

When documenting integrations, use this mapping consistently:

| General term | Common API / integration terms |
|---|---|
| Move | Action / Activity / Order |
| Plan | Trip / Plan |
| Pick Up | Pickup |
| Drop off | Dropoff |
| Vehicle | Vehicle |
| Organization | Organization / Account |
| Receipt | Receipt |
| Activity Payload | Action / Activity payload |

---

# Integration Documentation Convention

When documenting a customer integration, prefer the generic business terminology first and mention the API-specific terminology where useful.

For example:

> The integration receives a **Move (Action/Activity)** containing a **Vehicle**, **Pick Up**, and **Drop off**.

Rather than:

> The integration receives an `action` object containing `pickup` and `dropoff`.

This makes the documentation easier to understand across different integrations while still allowing developers to map the business concepts to the API payload.

---

# Related Notes

- [[Integration]]
- [[Webhook]]
- [[Inbound]]
- [[Procurement]]
- [[CORE API]]
- [[Activity Payload]]
- [[Move]]
- [[Plan]]
- [[Receipt]]
- [[Vehicle]]
- [[Organization]]

# Tags

#integrations #terminology #glossary #api #activity #move #trip #organization
