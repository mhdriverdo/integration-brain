---
title: CORE API - Quotes
type: api-component
domain: core-api
tags:
  - core-api
  - quotes
  - quoting
  - estimation
  - move
  - vehicle-delivery
related:
  - "[[CORE API]]"
  - "[[CORE API - Authentication]]"
  - "[[CORE API - Create Order]]"
  - "[[Move]]"
  - "[[Vehicle]]"
  - "[[Pick Up]]"
  - "[[Drop off]]"
---

# CORE API - Quotes

## Overview

The **Quoting API** is used to create, retrieve, and manage quotes for different delivery types.

According to the provided documentation, the API currently supports operations in:

- United States
- Canada

Quotes can be requested for either:

- A single estimation.
- Multiple estimations.

The quote provides an estimation for a delivery, including cost, distance, duration, and availability to assign a driver.

---

# EstimatorType

`estimatorType` specifies the type of estimation being requested.

The documented values are:

```text
PACKAGE_DELIVERY
PERSON_DELIVERY
ITINERARY_VEHICLE_DELIVERY
ONEWAY_VEHICLE_DELIVERY
TRIP_VEHICLE_DELIVERY
ONE_FOR_ONE_VEHICLE_DELIVER
```

For vehicle integrations, the documented vehicle-related options are:

```text
ITINERARY_VEHICLE_DELIVERY
ONEWAY_VEHICLE_DELIVERY
TRIP_VEHICLE_DELIVERY
ONE_FOR_ONE_VEHICLE_DELIVER
```

The documentation does not further define the behavioral difference between these estimator types.

---

# Quote Request

The provided documentation shows the following endpoint:

```text
GET /quote
```

Example:

```bash
curl --location --request GET 'http://core.prod.appservice.draiver.net/quote'   --header 'Content-Type: application/json'   --header 'Authorization: Bearer <apiToken>'   --data '{
    "estimatorType": "ONEWAY_VEHICLE_DELIVERY",
    "actions": [
      {
        "actionType": "MOVE_VEHICLES",
        "pickup": {
          "point": {
            "mapPointType": "GPS",
            "latitude": 41.69074190597469,
            "longitude": -83.44269023903364
          },
          "windowUTC": {
            "start": "2023-10-24T18:41:29.051Z",
            "end": "2023-10-24T22:41:29.051Z"
          }
        },
        "dropoff": {
          "point": {
            "mapPointType": "ADDRESS",
            "street1": "185 GA-16",
            "street2": null,
            "city": "Newnan",
            "state": "GA",
            "postalCode": "30263"
          },
          "windowUTC": {
            "start": "2023-10-26T18:41:29.052Z",
            "end": "2023-10-26T18:41:29.052Z"
          }
        },
        "vehicle": {
          "year": 2014,
          "make": "CHEVROLET",
          "model": "Cruze",
          "vin": "1G1PA5SG0E7174385"
        }
      }
    ]
  }'
```

---

# Request Structure

At a high level, a Quote request contains:

```text
Quote
│
├── estimatorType
│
└── actions
    │
    └── Action / Move
        ├── actionType
        ├── pickup
        │   ├── point
        │   └── windowUTC
        ├── dropoff
        │   ├── point
        │   └── windowUTC
        └── vehicle
```

The Action structure is closely related to the structure used when creating an Action through [[CORE API - Create Order]].

---

# Actions

The `actions` field is an array.

This allows the quote request to contain one or more Actions.

Example:

```json
{
  "actions": [
    {
      "actionType": "MOVE_VEHICLES",
      "pickup": {},
      "dropoff": {},
      "vehicle": {}
    }
  ]
}
```

The supplied documentation does not specify the maximum number of Actions that can be included in a quote request.

Related: [[Move]]

---

# Pickup

The quote request contains a Pick Up location and time window.

Example:

```json
{
  "pickup": {
    "point": {
      "mapPointType": "GPS",
      "latitude": 41.69074190597469,
      "longitude": -83.44269023903364
    },
    "windowUTC": {
      "start": "2023-10-24T18:41:29.051Z",
      "end": "2023-10-24T22:41:29.051Z"
    }
  }
}
```

The location can use GPS coordinates.

Related: [[Pick Up]]

---

# Dropoff

The quote request contains a Drop off location and time window.

Example:

```json
{
  "dropoff": {
    "point": {
      "mapPointType": "ADDRESS",
      "street1": "185 GA-16",
      "street2": null,
      "city": "Newnan",
      "state": "GA",
      "postalCode": "30263"
    },
    "windowUTC": {
      "start": "2023-10-26T18:41:29.052Z",
      "end": "2023-10-26T18:41:29.052Z"
    }
  }
}
```

The example demonstrates that the Pick Up and Drop off locations do not have to use the same location representation: one can use GPS while the other uses an address.

Related: [[Drop off]]

---

# Vehicle

The quote request includes Vehicle information.

Example:

```json
{
  "vehicle": {
    "year": 2014,
    "make": "CHEVROLET",
    "model": "Cruze",
    "vin": "1G1PA5SG0E7174385"
  }
}
```

Related: [[Vehicle]]

---

# Quote Response

The response contains an identifier and estimated operational information.

Representative response:

```json
{
  "id": "4qg25eYTRWGQQm5Z7qoFYw",
  "duration": {
    "value": 9387.5,
    "unitType": "SECOND"
  },
  "totalCost": {
    "value": 233.9,
    "unitType": "US_DOLLAR",
    "isoCurrencyCode": "USD"
  },
  "distance": {
    "value": 53788,
    "unitType": "METER"
  },
  "availability": {
    "value": 6,
    "unitType": "DAY"
  },
  "expirationUTC": "2022-09-01T20:00-03:00"
}
```

---

# Quote Response Fields

| Field | Description |
|---|---|
| `id` | Identifier of the quote |
| `duration` | Estimated delivery duration |
| `totalCost` | Estimated total cost |
| `distance` | Estimated distance |
| `availability` | Availability to assign a driver |
| `expirationUTC` | Quote expiration timestamp |

---

## Duration

The response expresses duration using a value and unit.

Example:

```json
{
  "duration": {
    "value": 9387.5,
    "unitType": "SECOND"
  }
}
```

The documented example uses seconds.

---

## Total Cost

The response includes the estimated total cost.

Example:

```json
{
  "totalCost": {
    "value": 233.9,
    "unitType": "US_DOLLAR",
    "isoCurrencyCode": "USD"
  }
}
```

The documentation states that the total cost can be represented in currencies including:

- Dollars
- Pesos
- Reales

The example uses:

```text
USD
```

with:

```text
US_DOLLAR
```

---

## Distance

The response includes estimated distance.

Example:

```json
{
  "distance": {
    "value": 53788,
    "unitType": "METER"
  }
}
```

The documentation states that distance can be represented in:

- Miles
- Kilometers

The supplied example uses meters.

---

## Availability

The `availability` field represents availability to assign a driver.

Example:

```json
{
  "availability": {
    "value": 6,
    "unitType": "DAY"
  }
}
```

The source description refers to availability in hours, while the provided response example uses:

```text
unitType = DAY
```

This discrepancy should be verified against the actual API contract.

---

## Expiration

`expirationUTC` indicates when the quote expires.

Example:

```json
{
  "expirationUTC": "2022-09-01T20:00-03:00"
}
```

The supplied documentation does not define:

- The default quote lifetime.
- What happens after expiration.
- Whether an expired quote can be refreshed.
- Whether the quote ID remains queryable after expiration.

---

# Quote vs Order

A **Quote** is an estimation; an **Action / Move** is an actual request to perform work.

Conceptually:

```text
Quote
  ↓
Estimated cost / distance / duration / availability
  ↓
Decision
  ↓
Create Action / Move
  ↓
Planning
  ↓
Plan / Trip
```

A quote should therefore not be interpreted as an already-created Move.

Related: [[CORE API - Create Order]]

---

# Integration Use Case

A customer integration can use Quotes when it needs an estimate before creating a Move.

Example flow:

```text
Customer Order
      ↓
Quote Request
      ↓
CORE API /quote
      ↓
Quote
      │
      ├── Cost
      ├── Distance
      ├── Duration
      └── Availability
      ↓
Customer decision
      ↓
Create Move
      ↓
POST /v2/actions
```

This can be useful when the external system needs to present pricing or delivery estimates before committing to the actual Move.

---

# Integration Mental Model

When troubleshooting a quote request:

```text
Authentication valid?
       ↓
Correct estimatorType?
       ↓
Actions present?
       ↓
Action type valid?
       ↓
Pickup valid?
       ↓
Dropoff valid?
       ↓
Vehicle information valid?
       ↓
Time windows valid?
       ↓
Quote generated?
       ↓
Inspect cost / distance / duration / availability
```

When a quote is unexpectedly different from a created Move, compare:

```text
Quote Request
     │
     ├── Pickup
     ├── Dropoff
     ├── Time Windows
     ├── Vehicle
     └── EstimatorType
            │
            ▼
      Created Action
```

Differences in these inputs may affect the resulting estimate.

The supplied documentation does not define the exact pricing or estimation algorithm, so the reason for a particular quote value should not be inferred from this document alone.

---

# Documentation Gaps / Things to Verify

The current documentation does not fully define:

- Exact HTTP method for `/quote` beyond the supplied example.
- Whether `/quote` supports both single and multiple estimations through the same endpoint.
- Exact endpoint(s) for retrieving quotes.
- Exact endpoint(s) for managing quotes.
- Maximum number of Actions per request.
- Detailed behavior of each `EstimatorType`.
- Exact pricing calculation.
- Whether taxes, fees, tolls or other costs are included in `totalCost`.
- Exact currency/unit options.
- Exact meaning and unit behavior of `availability`.
- Default quote expiration period.
- Behavior of expired quotes.
- Whether quotes can be converted directly into Actions.
- Whether `quoteId` is required or optional when subsequently creating an Action.
- Error responses and validation rules.

---

# Related Notes

- [[CORE API]]
- [[CORE API - Authentication]]
- [[CORE API - Create Order]]
- [[Move]]
- [[Vehicle]]
- [[Pick Up]]
- [[Drop off]]
- [[Organization]]
- [[Plan]]
- [[Procurement]]
- [[Provisioning]]
- [[Fulfillment]]
- [[Webhook]]

# Tags

#core-api #quotes #quoting #estimation #move #vehicle-delivery
