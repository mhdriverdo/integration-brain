---
date: 2026-09-15
tags: [draiver, procurement-microservice, negocio, overview]
source: opencode-chat
status: draft
---

# Draiver Procurement Microservice - Overview de Negocio

## 🎯 ¿Qué problema resuelve?

Draiver coordina el transporte de vehículos (car hauling / driveaway) entre distintos actores: drivers, haulers de terceros, brokers, clientes finales. El **procurement microservice** es el servicio que gestiona el ciclo de vida de los **movimientos de vehículos** ("Actions"): crearlos, consultarlos, actualizarlos, documentarlos (BOL, receipts, tracking) y — su función más sofisticada — **optimizar cómo se asignan y secuencian esos movimientos entre los conductores disponibles**.

En una frase: **es el cerebro operativo que decide qué conductor hace qué movimiento, en qué orden, y produce la documentación asociada.**

No confundir con "procurement" en el sentido de compras corporativas — acá el término refiere a la **adquisición/asignación de trabajo de transporte**.

## 🧩 El concepto central: la "Action"

Una `Action` (`MoveVehicleAction`, `MoveItemAction`) es la unidad mínima de trabajo: mover un vehículo/artículo desde un punto de pickup a un punto de dropoff, con:
- Ventanas de tiempo (ready time / due time)
- Organización dueña (multi-tenant)
- Estado (pending, in_progress, completed, etc.)

Sobre esa unidad se construyen todas las funcionalidades del servicio: documentación (BOL), pagos (fuel), tracking, y el ruteo óptimo.

## 🔀 Tres "sabores" de implementación (por qué importa para el negocio)

El servicio convive con **tres formas distintas de resolver la misma operación**, elegidas automáticamente según la ruta del request:

- **LEGACY**: delega en el sistema histórico de Draiver (backend legacy + Salesforce como sistema de registro). Es la fuente de la verdad para todo lo que ya existía antes de las nuevas capacidades de ruteo.
- **HYBRID**: combina LEGACY (para crear/leer/actualizar acciones básicas) con las **nuevas capacidades de optimización de rutas** (Quick Plan) y persistencia de metadata adicional. Es la puerta de entrada a todo lo nuevo.
- **V1**: no es una API sino un **procesador de eventos**: escucha cambios que ocurren en Salesforce (trips, activities, vehículos, recibos) y los traduce en actualizaciones locales + notificaciones (webhooks) hacia sistemas externos suscriptos.

Esto refleja una **migración en curso**: el negocio sigue operando sobre Salesforce como sistema histórico, mientras se construyen encima capacidades nuevas (optimización algorítmica de rutas) sin tener que reescribir todo de una.

## 🗺️ La funcionalidad estrella: "Quick Plan" (optimización de rutas)

Dado un conjunto de movimientos pendientes (pickups/deliveries) con ventanas de tiempo y un número de conductores disponibles, el sistema calcula automáticamente **la secuencia óptima de paradas por conductor**, minimizando según el objetivo de negocio elegido:

- **Minimizar distancia/costo total** recorrido por la flota.
- **Minimizar tiempo total** (que el último conductor termine lo antes posible).
- **Minimizar "millas vacías"** (chase/reposicionamiento sin carga — mover un conductor de un dropoff al próximo pickup sin vehículo a bordo). Esto es clave porque cada milla vacía es costo puro sin ingreso asociado.
- **Minimizar cantidad de conductores** necesarios para cubrir todos los movimientos.

Además, el sistema puede:
- Detectar automáticamente si una fecha límite de entrega es **inalcanzable** y avisarlo, en vez de fallar silenciosamente.
- Sugerir cuándo **dos conductores podrían compartir un mismo traslado en auto/Uber** (chase van) para ahorrar costo (agrupamiento/ride sharing).
- Generar **múltiples alternativas de plan** (no solo una única "mejor" respuesta), para que un dispatcher humano pueda elegir.

Esto se calcula con dos motores intercambiables:
- **Google OR-Tools**: motor propio que corre dentro del mismo servicio (sin depender de un proveedor externo), maduro y con varias estrategias de negocio configurables.
- **NextBillion.ai**: servicio externo (SaaS) de optimización de rutas, usado como alternativa. *(Nota: al momento de este análisis la integración de vuelta de NextBillion parece incompleta — devuelve una respuesta vacía tras calcular el plan.)*

## 📄 Otras capacidades de negocio

- **BOL (Bill of Lading)**: documento legal de transporte generado por acción.
- **Receipts y pagos de combustible**: registro de gastos asociados a los viajes.
- **Tracking URLs**: links de seguimiento en tiempo real para el cliente final.
- **Chainable moves**: sugerencias (asistidas por IA/OpenAI) de movimientos que un conductor cercano podría "encadenar" manualmente a su ruta actual.
- **Hauler orders**: integración con transportistas de terceros (partners de "car hauling") para tercerizar movimientos.
- **AI Sense**: sugerencia de acciones asistida por IA.
- **RunBuggy auto-match**: cuando una acción llega desde ciertas cuentas, el sistema reasigna automáticamente el trabajo a la sub-organización más cercana geográficamente, sin intervención manual.
- **Batch release**: liberación programada de lotes de acciones (scheduler configurable, on/off).

## 💼 Por qué es crítico para Draiver

- Es el punto donde se **decide operativamente cómo se mueve la flota**: una mala asignación de rutas se traduce directamente en más millas vacías, más horas pagadas a conductores y peor cumplimiento de deadlines con clientes.
- Sostiene simultáneamente **el negocio actual (vía Salesforce/legacy)** y **la evolución hacia optimización algorítmica**, sin que el resto de la plataforma note el cambio (gracias al patrón LEGACY/HYBRID/V1).
- Es el nexo entre **Salesforce (sistema de registro histórico)** y **los partners externos que necesitan enterarse de cambios en tiempo real** (vía webhooks), sosteniendo integraciones críticas (ej. Penske).

## 🔗 Ver también

- Nota técnica: `Draiver Procurement Microservice - Arquitectura Técnica.md`
- Nota técnica: `Draiver Procurement Microservice - Motor de Optimización de Rutas (OR-Tools vs NextBillion).md`
- Nota técnica: `Draiver Procurement Microservice - Integración con Salesforce vía Kinesis.md`
- Servicio relacionado: `Draiver Inbound Microservice - Overview de Negocio.md` (punto de entrada de datos de partners externos, distinto rol al de procurement)
