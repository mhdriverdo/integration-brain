---
date: 2026-09-15
tags: [draiver, procurement-microservice, arquitectura, java, spring-boot, aws]
source: opencode-chat
status: draft
---

# Draiver Procurement Microservice - Arquitectura Técnica

## 📋 Datos generales

- **Repo:** `draiver-microservice-procurement-java`
- **Artifact:** `com.draiver:draiver-microservice-procurement-java:0.0.62-SNAPSHOT`
- **Parent Maven:** `draiver-build-parent_microservice-maven:0.0.24`
- **Entry point:** `com.draiver.procurement.Application` (extiende `MicroserviceApplication`)
- **Puerto:** 9000 (local) / 5000 (Beanstalk)
- **Health check:** `/system/health`
- **Deploy:** AWS Elastic Beanstalk, blue-green en prod
- **Nombre app Beanstalk:** `microservice-procurement`

## 🏗️ Arquitectura general: 3 implementaciones seleccionadas en runtime

El servicio implementa un **Strategy Pattern por request**, no por ambiente/config. `domain/ServiceProvider.java` decide dinámicamente qué implementación usar según:

1. Si el path contiene `actions/machina` o `hauler` → fuerza **HYBRID**
2. Si viene el header `x-provider` → usa ese valor
3. Default → **LEGACY**

```
HTTP Request
     │
     ▼
ServiceProvider.detect(request)
     │
     ├── LEGACY  → LegacyProcurementService   (delega a Salesforce/backend legacy)
     ├── HYBRID  → HybridProcurementService   (legacy + OR-Tools/NextBillion + Mongo)
     └── (V1 no se selecciona por request; es un consumidor de eventos Kinesis aparte)
```

Esto se implementa con un **Factory genérico**: `ServiceFactoryBase<T>` mantiene un `Map<ServiceProvider, T>`, y cada dominio (`ProcurementServiceFactory`, `HaulerServiceFactory`, `FuelPaymentServiceFactory`, `AISenseServiceFactory`) registra sus implementaciones concretas por provider.

`ProcurementServiceBase` (811 líneas) es la clase plantilla (Template Method) que envuelve cada operación pública con logging, auditoría asíncrona y manejo uniforme de excepciones, delegando la lógica real a métodos abstractos que implementan Legacy y Hybrid.

## 📂 Estructura de paquetes

```
com.draiver.procurement
├── Application.java
├── controller/                     — REST controllers
├── service/                        — interfaces base, factories
│   ├── hauler/, runbuggy/, repository/, aisense/
├── domain/                         — modelos propios (SF*, Kinesis, ServiceProvider)
│   ├── persist/, runbuggy/, aisense/
├── impl/
│   ├── legacy/                     — implementación LEGACY
│   ├── hybrid/                     — implementación HYBRID
│   │   └── service/QuickPlanProviders/{NextBillion, OrTools/{model,strategy}}
│   ├── v1/                         — consumidores Kinesis / eventos Salesforce
│   │   └── service/streamevent/{processors/{actionsheet,activity,location,receipt,trip,vehicle}}
│   ├── repository/                 — MongoDB (metadata + runbuggy locations)
│   └── requestedactivity/          — consulta JPA de snapshots de actividades
├── config/                         — Kinesis, Mongo, seguridad, CORS
├── aggregation/                    — ActionBatcher (batch release)
├── async/                          — KinesisProducer (webhooks salientes)
├── audit/                          — eventos de auditoría
└── client/machina/                 — cliente/dominio "Machina"
```

## 🌐 Controllers REST

| Controller | Rutas base | Responsabilidad |
|---|---|---|
| `ProcurementController` | `/actions`, `/trips` | CRUD de Actions, BOL, receipt, tracking URL, `/actions/machina` (Quick Plan), chainable moves |
| `V2ActionsController` | `/actions/v2` | API v2 de Actions (solo soportado por Legacy) |
| `HaulerController` | `/hauler` | Crear órdenes con proveedores externos de hauling |
| `FuelPaymentController` | `/payments/fuel/{organizationId}` | Pagos de combustible |
| `RequestedActivityController` | — | Lectura de snapshots locales de actividades Salesforce |
| `AISenseController` | `/aisense` | Sugerencias de acciones vía IA |
| `BatchSchedulerController` | `/scheduler` | Start/stop/status del batch release |

## 🗄️ Persistencia

- **MySQL vía JPA** (`com.mysql:mysql-connector-j:8.3.0`): entidades compartidas en `com.draiver.jpa.draivercore` (tenants, requested activities, webhook subscriptions).
- **MongoDB** (`impl/repository/MongoDBRepository.java`): implementación mínima ad-hoc (no Spring Data), solo 2 colecciones: `metadata` (actividades) y `runbuggy_locations`.
- **H2 in-memory** en tests (modo MySQL, `MODE=MYSQL`).

## 🔌 Dependencias internas Draiver clave

| Librería | Propósito |
|---|---|
| `draiver-api-procurement-java` (0.0.51-SNAPSHOT) | Contratos del dominio (requests/responses, `HttpRoutes`, `QuickPlanStrategy`) |
| `draiver-client-draiver_legacy-java` | Cliente al backend legacy/Salesforce — pieza clave de LEGACY |
| `draiver-client-thirdparty-java` | Clientes NextBillion y OpenAI |
| `draiver-api-webhook-java` | Contratos de webhooks salientes |
| `draiver-library-jpa_draiver_core-java` / `draiver-library-draiver_core-java` | Entidades y modelos compartidos |
| `draiver-utility-microservice-java` | Base `MicroserviceApplication`/`MicroserviceController`, AOP `@LogTime`/`@AuditTime` |

## ⚙️ Configuración

- `bootstrap.yml`: nombre de app resuelto vía Maven (`@consul.application.name@`), multipart hasta 500MB.
- `application.properties`: puerto 9000, batching de auditoría.
- Perfiles por ambiente: `dev,nonprodconfig` / `qa,nonprodconfig` / `staging,prodconfig` / `prod-blue,prod,prodconfig` / `prod-green,prod,prodconfig`.

## 🚀 Deployment (Beanstalk)

| Ambiente | ASG min/max | Notas |
|---|---|---|
| Dev | 1/1 | Spot instances |
| Staging | 0/4 | VPC de prod |
| Prod Blue | 2/4 | `PROD_ENV=BLUE` |
| Prod Green | 2/4 | `PROD_ENV=GREEN` |

- Blue/Green: mismo código, distinto hostname/perfil/variable `PROD_ENV`, alternado vía Route53/ALB (deploy sin downtime).
- `JAVA_TOOL_OPTIONS`: `-Xms2g -Xmx2g -XX:MetaspaceSize=64m -XX:MaxMetaspaceSize=256m`.
- `SERVER_PORT=5000` en Beanstalk. Health check `/system/health` cada 30s.
- Spot instances habilitadas (`t3.medium, t2.medium`).

## ⚠️ Pitfalls / notas conocidas

- El `README.md` del repo está vacío — no había documentación previa de alto nivel.
- `draiver-api-procurement-java` es un **SNAPSHOT** activo → el contrato de API puede cambiar.
- `NextBillionQuickPlanProvider.getQuickPlan()` retorna actualmente una respuesta vacía tras el polling — la conversión de vuelta parece incompleta (ver nota de optimización de rutas).
- MongoDB es deliberadamente "mínimo" (comentario explícito en el código) hasta que se justifique Spring Data MongoDB.

## 🔗 Ver también

- Nota de negocio: `Draiver Procurement Microservice - Overview de Negocio.md`
- Nota técnica: `Draiver Procurement Microservice - Motor de Optimización de Rutas (OR-Tools vs NextBillion).md`
- Nota técnica: `Draiver Procurement Microservice - Integración con Salesforce vía Kinesis.md`
