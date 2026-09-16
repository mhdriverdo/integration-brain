---
date: 2026-09-15
tags: [draiver, procurement-microservice, kinesis, salesforce, arquitectura, eventos]
source: opencode-chat
status: draft
---

# Draiver Procurement Microservice - Integración con Salesforce vía Kinesis

## 🎯 Contexto

Salesforce sigue siendo el **sistema de registro histórico** de Draiver para Trips, Activities, ActionSheets, Vehicles y Recibos. La implementación **V1** (`impl/v1/`) del procurement microservice no expone una API propia — es un **consumidor de eventos de Kinesis** que escucha cambios de Salesforce (CDC / Platform Events) y los propaga hacia adentro (persistencia local) y hacia afuera (webhooks a partners).

```
Salesforce (Trips/Activities/ActionSheets/Vehicles/CustomDestinations/Receipts)
   │  CDC / Platform Events
   ▼
Kinesis stream (entrante)
   │
   ▼
KCL Scheduler (KinesisConfig) → RecordProcessor
   │
   ▼
SalesforceTopicHandlers (routing por topic)
   │
   ▼
UseCases específicos (persistencia local JPA + lógica de negocio)
   │
   ▼
EventDispatcher → KinesisProducer → Kinesis stream (saliente, webhooks)
   │
   ▼
Sistemas externos suscriptos (vía draiver-api-webhook-java)
```

## 📥 Consumo (KCL — Kinesis Client Library 2.5.4)

- **`config/kinesis/StreamConfigurations.java`**: define 2 streams entrantes — uno de "activities" y otro de "receipts" — sobre un ARN base configurable.
- **`config/kinesis/KinesisConfig.java`**: arma un `Scheduler` de KCL con beans AWS (`KinesisAsyncClient`, `DynamoDbAsyncClient` para lease/checkpointing, `CloudWatchAsyncClient` para métricas). Nombre de app KCL: `{spring.application.name}-{aws.env}`, worker id fijo `"procurement-consumer"`.
- **`config/kinesis/KinesisStreamListener.java`**: al arrancar Spring lanza el `Scheduler` en un hilo daemon; shutdown limpio en `@PreDestroy`.
- **`RecordProcessor`**: por cada record, decodifica UTF-8, deserializa a `KinesisRecord` (contiene `salesforceTopic` + `data: SFData` polimórfico), y despacha según el topic.

## 🗂️ Routing por topic de Salesforce

`config/SalesforceTopicHandlers.java` mapea 6 topics a sus handlers:

| Topic Salesforce | Handler |
|---|---|
| `/topic/ActionSheetUpdates` | `ActionSheetTopicHandler` |
| `/topic/TripUpdates` | `TripUpdatesTopicHandler` |
| `/topic/ActivityUpdates` | `ActivityUpdatesTopicHandler` |
| `/topic/ActivityCostUpdate` | `ReceiptsTopicHandler` |
| `/topic/CustomDestinationUpdates` | `CustomDestinationUpdatesTopicHandler` |
| `/topic/VehicleUpdates` | `VehicleUpdatesTopicHandler` |

Cada handler tiene use cases especializados (`impl/v1/service/streamevent/processors/{actionsheet,activity,location,receipt,trip,vehicle}`), por ejemplo:
- `PickupCompletedUseCase`: dispara webhook `IN_PROGRESS` cuando se completa el pickup de un vehículo.
- `DryRunUseCase`, `VinMismatchUseCase`: validaciones/casos especiales sobre action sheets.
- `ActivityPersistenceUseCase`, `CustomDestinationPersistenceUseCase`, `RequestedActivityVehiclePersistenceUseCase`: persisten "snapshots" locales vía JPA para consulta rápida sin ir a Salesforce.

## 📤 Emisión de webhooks (EventDispatcher)

`impl/v1/service/streamevent/processors/EventDispatcher.java` es el componente central de salida:

1. Resuelve el `Tenant` (organización) por `accountId` de Salesforce (`SharedEntityUtility`), soportando tenants jerárquicos (root tenant).
2. Construye un `WebhookEvent` (id, orgId, referenceId, eventType, entity, field, payload, source, timestamp).
3. Lo envía a **`KinesisProducer`** (`async/KinesisProducer.java`), un productor saliente hacia otro stream de Kinesis, notificando a integraciones externas suscriptas (contrato `draiver-api-webhook-java`).
4. Emite un evento de auditoría (`WebhookAuditEvent`).
5. Lógica especial para **Penske** (`PenskeConfig`): si el orgId corresponde a la cuenta admin de Penske, marca el `source` del webhook como `"penske"` en vez de `"procurement"`.

### Filtrado por suscripción

`WebhookSubscriptionFilterService` chequea si hay suscripciones activas para la organización **antes** de emitir el evento — evita despachar webhooks a nadie cuando no hay nadie escuchando (`WebhookSubscriptionIndex`/`WebhookSubscriptionIndexEntry`).

## 🗄️ Modelos de dominio relevantes (`domain/`)

Modelos que espejan objetos de Salesforce, deserializados de Kinesis (todos con `@JsonProperty` mapeando campos `__c`):

- `SFTrip`, `SFActivity`, `SFActionSheet`, `SFVehicle`, `SFCustomDestination`, `SFEvent`
- `SFData`: envoltorio polimórfico (`sobject` + `event`)
- `KinesisRecord`: wrapper de entrada (`salesforceTopic` + `data: SFData`), con deserializador custom `KinesisObjectDeserializer`

## ⚠️ Por qué importa

- Es el mecanismo que mantiene sincronizados los sistemas propios de Draiver con Salesforce **sin acoplar a los clientes externos directamente a Salesforce** — todo pasa por el contrato interno de webhooks.
- El filtrado por suscripción y el manejo especial de Penske son puntos sensibles: cualquier cambio ahí puede silenciar notificaciones a partners reales sin errores visibles.
- Al ser consumo vía KCL con checkpointing en DynamoDB, un reinicio del servicio no reprocesa desde cero — retoma desde el último checkpoint por shard.

## 🔗 Ver también

- Nota técnica: `Draiver Procurement Microservice - Arquitectura Técnica.md`
- Nota de negocio: `Draiver Procurement Microservice - Overview de Negocio.md`
- Servicio relacionado (otra punta de la integración con partners externos): `Draiver Inbound Microservice - Arquitectura Técnica.md`
