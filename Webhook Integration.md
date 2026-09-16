---
title: Webhook Integration
type: integration-component
domain: integrations
tags:
  - integrations
  - webhook
  - event-driven
  - aws
  - kinesis
  - java
  - spring-boot
  - draiver
related:
  - "[[Integration]]"
  - "[[Inbound]]"
  - "[[Procurement]]"
  - "[[CORE API]]"
  - "[[AWS Kinesis]]"
---

# Webhook Integration

## Overview

The **Webhook** microservice is one of the main building blocks used by the [[Integration]] team to implement customer-specific, event-driven integrations.

It is a Java / Spring Boot microservice responsible for:

- Defining which webhook events and fields are available.
- Allowing integrators to register callbacks for those events.
- Receiving events through **AWS Kinesis Data Streams**.
- Matching incoming events with configured webhooks.
- Executing the corresponding HTTP callbacks against external systems.

In the typical integration architecture, Webhook works together with [[Inbound]], [[Procurement]], and the [[CORE API]].

> **Key idea:** Webhook does not generally contain the business logic of a customer integration. Instead, it provides the event-driven mechanism that allows an external system to be notified when something happens inside DRAIVER.

---

## Repository

The service is hosted in GitHub:

- [Draiver Microservice Webhook Java](https://github.com/DriverDo/draiver-microservice-webhook-java)

---

## Architecture Role

A simplified integration flow looks like:

```text
                    DRAIVER
                       │
          ┌────────────┴────────────┐
          │                         │
      Procurement                 CORE API
          │                         │
          └────────────┬────────────┘
                       │
                    Event
                       │
                       ▼
                AWS Kinesis
                       │
                       ▼
             Webhook Microservice
                       │
              Match configuration
                       │
                       ▼
                Webhook Callback
                       │
                       ▼
                Customer System
```

The exact source of the event depends on the integration and the configured `source`.

For example:

```text
Procurement event
      ↓
AWS Kinesis
      ↓
Webhook
      ↓
Configured callback
      ↓
Customer endpoint
```

---

# Core Concepts

## WebhookConfig

`WebhookConfig` defines **what can be subscribed to**.

It is an internal configuration managed by DRAIVER.

A configuration contains:

| Field | Description |
|---|---|
| `id` | Unique identifier |
| `entity` | Entity associated with the webhook |
| `type` | `entity` or `organization` |
| `fields` | Fields/events available for subscription |
| `source` | Microservice where the event originates |
| `organizationId` | Required when the type is `organization` |
| `createdAt` | Creation timestamp |
| `updatedAt` | Last update timestamp |

### Important distinction

`WebhookConfig` is **not the customer callback itself**.

It defines the available webhook capabilities.

Conceptually:

```text
WebhookConfig
    ↓
"What events can be subscribed to?"
```

while:

```text
Webhook
    ↓
"Where should this event be sent?"
```

---

## Webhook

A `Webhook` represents a subscription/configuration created by an external integrator.

It connects:

1. An entity.
2. A field/event.
3. A source.
4. An organization.
5. One or more callbacks.

Example:

```json
{
  "entity": "action",
  "type": "organization",
  "field": "status",
  "source": "draiver-procurment-microservice",
  "organizationId": "EewSQK4_QQGY4QK0AD5BeQ",
  "webhookCallbacks": []
}
```

---

## WebhookCallback

A `WebhookCallback` defines the actual HTTP request that should be executed when the configured event occurs.

It contains:

| Field | Description |
|---|---|
| `id` | Callback identifier |
| `eventId` | Identifier of the event |
| `eventType` | Type of event |
| `headers` | HTTP headers to send |
| `method` | HTTP method |
| `url` | Customer endpoint |
| `body` | Payload/body for the callback |
| `createdAt` | Creation timestamp |
| `updatedAt` | Last update timestamp |

Example:

```json
{
  "eventId": "123",
  "eventType": "ACTION_CREATED",
  "headers": "x-caller-id:12232425",
  "method": "post",
  "url": "https://customer.example.com/webhook",
  "body": "{}"
}
```

---

# Configuration vs Integration API

The service exposes two conceptual APIs.

## Internal Configuration API

Used by DRAIVER to define available webhook configurations.

Main endpoints:

```text
POST   /v1/configuration
PATCH  /v1/configuration/{id}
GET    /v1/configuration/{id}
GET    /v1/configuration/organization/{id}
DELETE /v1/configuration/{id}
```

This API controls the **catalog of available webhook events**.

Example:

```json
{
  "entity": "action",
  "type": "organization",
  "fields": [
    "ACTION_UNPLANNED",
    "ACTION_CREATED"
  ],
  "source": "draiver-procurment-microservice",
  "organizationsId": [
    "Ee7sZ5Ro3beEGgKzDNZ5Lw"
  ]
}
```

---

## Public Integration API

Used by external integrators to create and manage their webhook subscriptions.

Main endpoints:

```text
POST   /v1/webhooks
PATCH  /v1/webhooks/{id}
GET    /v1/webhooks/{id}
GET    /v1/webhooks/organization/{id}
DELETE /v1/webhooks/{id}
POST   /v1/webhooks/callback/{id}/execute
```

The most important operation for an integration is:

```text
POST /v1/webhooks
```

because it creates the webhook subscription and specifies the callback endpoint.

---

# Event Processing

Webhook is event-driven.

When an event is produced by another DRAIVER service, it is communicated to the Webhook microservice through **AWS Kinesis Data Streams**.

The general process is:

```text
1. Business event occurs
        ↓
2. Event is published
        ↓
3. Event reaches AWS Kinesis
        ↓
4. Webhook consumes the event
        ↓
5. Webhook identifies matching configuration
        ↓
6. Matching callback is selected
        ↓
7. HTTP request is generated
        ↓
8. Customer endpoint receives the callback
```

This means the producer of the event and the customer-facing callback are decoupled.

---

# Example End-to-End Flow

Suppose an action is created in DRAIVER.

```text
Procurement
    │
    │ ACTION_CREATED
    ▼
AWS Kinesis
    │
    ▼
Webhook
    │
    │ Find matching subscription
    ▼
WebhookCallback
    │
    │ POST
    ▼
Customer
```

The customer may have previously registered:

```json
{
  "entity": "action",
  "type": "organization",
  "field": "status",
  "source": "draiver-procurment-microservice",
  "organizationId": "EewSQK4_QQGY4QK0AD5BeQ",
  "webhookCallbacks": [
    {
      "eventId": "123",
      "eventType": "ACTION_CREATED",
      "method": "post",
      "url": "https://customer.example.com/webhook",
      "body": "{}"
    }
  ]
}
```

When the corresponding event arrives, Webhook executes the configured HTTP callback.

---

# Why Webhook Matters for Integrations

Webhook is particularly useful when an integration needs to **push changes from DRAIVER to a customer system**.

A common integration can therefore be thought of as two directions:

```text
CUSTOMER → DRAIVER
     │
     ▼
  [[Inbound]]
     │
     ▼
DRAIVER
     │
     ▼
  [[Webhook]]
     │
     ▼
CUSTOMER
```

This is an important architectural distinction:

- **Inbound** is primarily concerned with receiving information from external systems.
- **Webhook** is primarily concerned with notifying external systems about events occurring inside DRAIVER.
- **Procurement / CORE API** provide the business capabilities and data that can generate or consume these events.

---

# Integration-Specific Considerations

Although the Webhook microservice provides a common mechanism, each customer integration can use it differently.

The integration team needs to understand:

- Which DRAIVER service produces the event.
- Which `source` is associated with the event.
- Which entity and field are exposed.
- Which `eventType` identifies the business event.
- Which organization the webhook belongs to.
- What HTTP method the customer expects.
- Which headers are required.
- What payload/body format the customer expects.
- Whether the customer expects raw or transformed data.
- How failures and retries are handled.

Therefore, **Webhook provides the infrastructure for the integration, but the integration-specific contract still needs to be understood and documented.**

---

# Common Integration Pattern

For many customer integrations, the architecture follows this pattern:

```text
                 ┌──────────────┐
                 │ Customer     │
                 └──────┬───────┘
                        │
                  inbound data
                        │
                        ▼
                 ┌──────────────┐
                 │   Inbound    │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │ Procurement  │
                 └──────┬───────┘
                        │
                     events
                        │
                        ▼
                 ┌──────────────┐
                 │ AWS Kinesis  │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │   Webhook    │
                 └──────┬───────┘
                        │
                  HTTP callback
                        │
                        ▼
                 ┌──────────────┐
                 │ Customer     │
                 └──────────────┘
```

Not every integration follows this exact flow, but this is the **common conceptual model** for event-driven customer integrations.

---

# Operational / Debugging Mental Model

When a webhook notification is not received by a customer, investigate the flow from left to right:

```text
Business event
     ↓
Event generated?
     ↓
Event published?
     ↓
Kinesis
     ↓
Webhook consuming?
     ↓
Webhook configuration exists?
     ↓
Event matches configuration?
     ↓
Callback exists?
     ↓
Callback executed?
     ↓
Customer endpoint reachable?
     ↓
Customer accepted request?
```

This decomposition is useful because a failure in any stage can look like a generic "webhook is not working" problem from the customer's perspective.

---

# Related Notes

- [[Integration]] — Integration team and overall integration architecture.
- [[Inbound]] — Receiving information/events from external systems.
- [[Procurement]] — Core business workflow that can produce integration events.
- [[CORE API]] — Main API used by integrations.
- [[AWS Kinesis]] — Event streaming infrastructure used by Webhook.
- [[Openlane]] — Customer-specific integration.
- [[Rivian]] — Customer-specific integration.
- [[Shipcar]] — Customer-specific integration.
- [[SuperDispatch]] — Integration provider serving multiple customers.
- [[Autofleet]] — Customer-specific integration.
- [[Park My Fleet]] — Customer-specific integration.

---

# Knowledge Gaps / Things to Document Later

This note describes the Webhook service based on the current available documentation. The following areas should be documented separately as they become known:

- Exact Kinesis stream/topic configuration.
- Event schema received by Webhook.
- How event matching is implemented.
- Retry policy.
- Timeout configuration.
- Failure handling / DLQ behavior, if applicable.
- Idempotency behavior.
- Ordering guarantees.
- Authentication requirements for callbacks.
- Raw vs transformed event payloads.
- Observability: logs, metrics and dashboards.
- How to troubleshoot a callback that was not sent.
- How Webhook interacts with specific integrations such as [[Openlane]], [[Rivian]], or [[SuperDispatch]].
- Deployment and AWS infrastructure.
- Staging vs production differences.

---

# Tags

#integrations #webhook #event-driven #aws #kinesis #java #spring-boot #draiver
