---
date: 2026-09-15
tags: [draiver, inbound-microservice, arquitectura, java, spring-boot, aws]
source: opencode-chat
status: draft
---

# Draiver Inbound Microservice - Arquitectura Técnica

## 📋 Datos generales

- **Repo:** `draiver-microservice-inbound-java`
- **Stack:** Java 17 (Amazon Corretto), Spring Boot, Maven
- **Parent Maven:** `draiver-build-parent_microservice-maven:0.0.24`
- **Entry point:** `com.draiver.inbound.Application`
- **Puerto:** 5000 (beanstalk envs) / 8080 (local)
- **Health check:** `/system/health` (no usa el default de actuator)
- **Deploy:** AWS Elastic Beanstalk, Amazon Linux 2, blue-green en prod

## 🏗️ Arquitectura general

Es un **hub de integración multi-cliente** que conecta 8+ APIs de clientes externos con la plataforma Draiver, usando un **patrón de orquestación basado en recetas (recipes)**.

```
Cliente externo (webhook/API/SFTP)
        │
        ▼
   Recipe execution (ChefService)
        │
        ▼
   SQS FIFO queue
        │
        ▼
   Kinesis stream
        │
        ▼
   MySQL + actualizaciones al cliente externo
```

## 📂 Estructura de paquetes clave

```
controller/                              → REST endpoints (/inbound/webhook, /v1/recipes/*)
service/                                  → ChefService, SQS, Kinesis, SFTP, webhook processors
service/engine/steps/inbound/impl/        → Steps reutilizables (FetchData, ConvertToDraiver, etc.)
domain/client/{client}/                   → DTOs específicos por cliente
domain/configuration/                     → Recipe, ClientRecipe (modelos de dominio)
config/recipes/                           → RecipeConfigurations, RecipeLoader, InboundRecipeStepNames
config/param/store/auth/                  → Configs de AWS Parameter Store con @RefreshScope
```

Clientes con dominio propio bajo `domain/client/`: `autofleet`, `ims`, `morgan`, `runbuggy`, `shipcars`, `superdispatch`, `truckmovers`. Rivian se maneja distinto (SFTP/EDI, ver `domain/edi/` y `service/sftp/`).

## 🍳 Patrón de Recetas (Recipe Pattern) — el corazón del sistema

**Ubicación de recetas:** `src/main/resources/static/recipes/{client}/inbound-{client}.json`

Ejemplo de estructura:
```json
{
  "steps": [
    ["ConvertToDraiver"],
    ["FetchData"],
    ["UpdateDraiverActions"],
    ["UpdateOutboundActions"]
  ]
}
```

**Modelo de ejecución:**
- `ChefService.cook(recipe, name)` ejecuta recetas de forma asíncrona con un **thread pool de 20 hilos**.
- Cada ejecución tiene su propio `RequestContext` (ConcurrentHashMap) aislado + un `executionId` (UUID) único para trazabilidad.
- Los steps de una receta se ejecutan secuencialmente, modificando el contexto compartido.
- Los steps son **Spring beans** registrados en `InboundRecipeStepNames.AVAILABLE_STEPS`.
- Los fallos disparan **notificaciones a Slack**.

**Convención de nombres de steps comunes:**
- `{CLIENT}_FETCH_DATA` — llama a la API del cliente
- `{CLIENT}_CONVERT_TO_DRAIVER` — transforma al formato interno
- `{CLIENT}_UPDATE_DRAIVER_ACTIONS` — persiste en la DB de Draiver
- `{CLIENT}_UPDATE_OUTBOUND_ACTIONS` — envía ack al cliente
- `{CLIENT}_AUTORIZATION` — autenticación con la API del cliente

**Endpoints de recetas:**
```
GET  /v1/recipes/client/{clientId}
POST /v1/recipes/inbound/execute/client/{id}
POST /v1/recipes/outbound/execute/client/{id}
POST /inbound/webhook?recipeId={clientId}&recipeType=inbound&payload={data}
```

**Client IDs hardcodeados en `RecipeConfigurations`:**

| Cliente | UUID |
|---|---|
| AutoFleet | `a0a8419e-5996-4c8a-a0bb-fa5256709eb0` |
| ShipCars | `a1b2c3d4-e5f6-7890-abcd-ef1234567890` (también `shipcars-webhook`) |
| SuperDispatch | `313cb6e0-7a3c-4bb7-afe8-caf0a34c5fe3` |
| IMS | `ead9278b-436e-45ad-ae71-189c7947d9c1` |
| TruckMovers | `11ec0596-84f4-9a41-98e1-02b4003e4179` |
| OpenLane | `f061e04b-3f3f-435b-b9de-c2bd0254bbb7` |
| Rivian | `585c68f6-f03d-4a2b-9e2a-158eed8b4789` |
| Morgan | `91662826-566f-43e0-802d-32f64dbf2cc9` |

**Cómo agregar un nuevo cliente:**
1. Crear el JSON del recipe en `src/main/resources/static/recipes/{client}/`
2. Implementar los Steps necesarios en `service/engine/steps/inbound/impl/`
3. Registrar los nombres de steps en `InboundRecipeStepNames.AVAILABLE_STEPS`
4. Agregar el cliente en `RecipeConfigurations.INBOUND_CLIENT_RECIPES`
5. Crear los DTOs en `domain/client/{client}/`
6. Agregar config de auth en `config/param/store/auth/{Client}Config.java` con `@RefreshScope`

## ☁️ Integraciones externas

- **AWS SQS FIFO** para procesamiento asíncrono (`spring.cloud.aws.sqs`), requiere sufijo `.fifo` y `MessageGroupId`
- **AWS Kinesis streams** para distribución de eventos
- **SFTP** para transacciones EDI de Rivian (X12: 214, 120, 204) — único cliente que no usa webhook/SQS como entrada
- **MySQL vía JPA** (entidades `com.draiver.jpa.draivercore`)
- **Spring Retry** habilitado (`@EnableRetry`)
- **AWS Parameter Store** para credenciales/tokens, con beans `@RefreshScope`

## ⚙️ Configuración

**Orden de resolución de properties:**
1. `bootstrap.yml` — setea `consul.application.name` desde Maven (`@consul.application.name@` se resuelve en build)
2. `application.properties` — colas SQS, retry, batch de auditoría
3. `clients.yml` — configs de SFTP/S3/SQS por cliente (importado vía `spring.config.import=clients.yml`)
4. AWS Parameter Store — tokens/credenciales (vía configs `@RefreshScope`)

**Properties críticas:**
- `microservice.procurement.url` → default `http://localhost:9000`
- `aws.sqs.queue.inbound-queue` → cola FIFO
- `draiver.audit.use-batch=true`, batch-size 200

**Profiles por ambiente:** `dev,nonprodconfig` / `qa,nonprodconfig` / `staging,prodconfig` / `prod,prodconfig`

## 🧪 Testing

- **Solo unit tests** — nada de `@SpringBootTest`, `@WebMvcTest` ni integración real
- `@ExtendWith(MockitoExtension.class)` + `@Mock`
- Tests en `src/test/java/com/draiver/inbound/` reflejan el paquete de `main`
- Sin prerequisitos externos (todo mockeado)

## 🚀 Deployment

- AWS Elastic Beanstalk, blue-green en prod
- URLs: `inbound.{env}.microservice.draiver.net` (dev, qa, staging, prod-blue, prod-green)
- JVM args dev: `-Xms768m -Xmx1536m -XX:MetaspaceSize=64m -XX:MaxMetaspaceSize=256m`
- ALB compartido con host-based routing (priority 1)
- Instance profile: `aws-elasticbeanstalk-ec2-role`

## ⚠️ Pitfalls conocidos

- **Consul:** el placeholder `@consul.application.name@` se resuelve en build time desde el `pom.xml`
- **`@RefreshScope`:** los beans recargan desde Parameter Store, pero si el store no dispara el refresh puede requerir reinicio
- **SQS FIFO:** requiere sufijo `.fifo` y `MessageGroupId` sí o sí
- **Complejidad multi-cliente:** 8+ integraciones, cada una con su propio flujo de auth/datos — siempre revisar `domain/client/{client}/` antes de tocar algo genérico
- **SFTP vs SQS:** Rivian es la excepción — usa SFTP para EDI, el resto usa webhooks/SQS
- **Uploads grandes:** `max-file-size: 500MB` configurado para inspecciones con fotos

## 🔗 Ver también

- Nota de negocio: `2026-09-15 - Draiver Inbound Microservice - Overview de Negocio.md`
- `2026-08-26 - Refactorización Procesamiento Paralelo StreamRecordProcessor.md`
- `2026-09-01 - Plan Refactorización SftpService - Garantizar Entrega de Documentos EDI.md`
