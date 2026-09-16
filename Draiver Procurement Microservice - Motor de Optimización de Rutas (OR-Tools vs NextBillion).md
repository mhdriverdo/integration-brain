---
date: 2026-09-15
tags: [draiver, procurement-microservice, or-tools, optimizacion-rutas, arquitectura]
source: opencode-chat
status: draft
---

# Draiver Procurement Microservice - Motor de Optimización de Rutas (OR-Tools vs NextBillion)

## 🎯 Contexto

Dentro de la implementación **HYBRID** del procurement microservice, el endpoint `POST /actions/machina` ("Quick Plan") resuelve un problema clásico de **VRP (Vehicle Routing Problem) con Pickup & Delivery y ventanas de tiempo**: dado un set de movimientos de vehículos y N conductores disponibles, calcular la secuencia óptima de paradas.

Se seleccionan dos motores intercambiables según `request.getQuickPlanProvider()` en `HybridProcurementService.onGetMachinaQuickPlan()`:

```java
switch (request.getQuickPlanProvider()) {
    case NEXT_BILLION -> NextBillionQuickPlanProvider.getQuickPlan(...);
    case OR_TOOLS -> OrToolsQuickPlanProvider.getQuickPlan(...);
}
```

## 🅰️ NextBillion.ai (SaaS externo)

`impl/hybrid/service/QuickPlanProviders/NextBillion/NextBillionQuickPlanProvider.java` (~33 líneas):

1. Convierte el request al formato de NextBillion (`RouteOptimizationRequestConverter`).
2. Llama a `nextBillionClient.generatePlan()` (API externa vía `draiver-client-thirdparty-java`).
3. Hace **polling** (`getPlan(planId)`) cada 1s, hasta 100 reintentos, esperando que `response.getResult().getRoutes()` no esté vacío.

⚠️ **Estado incompleto**: el método actualmente retorna un `GetMachinaQuickPlanResponse()` vacío al final del polling. La conversión de vuelta (resultado NextBillion → respuesta del servicio) parece pendiente de terminar.

## 🅱️ Google OR-Tools (motor propio, in-process)

`impl/hybrid/service/QuickPlanProviders/OrTools/OrToolsQuickPlanProvider.java` (~1300 líneas) — el motor maduro y de referencia. No depende de red; corre embebido usando `com.google.ortools:ortools-java:9.15.6755` (requiere librerías nativas por plataforma — desempaquetadas especialmente en el build de Spring Boot para linux-x86-64/aarch64, darwin-x86-64/aarch64, win32-x86-64).

### Modelado del problema

- **Nodos** (`RouteNode`): un START por conductor (real o virtual en 0,0 si no hay ubicación de arranque), un par pickup/delivery por cada acción, y nodos END (uno por conductor).
- **Matrices de distancia y tiempo**: distancia Haversine, velocidad asumida 45 km/h (12.5 m/s), buffer 1.3x sobre la línea recta.
- **Dimensiones OR-Tools**:
  - `Time`: incluye 900s de tiempo de servicio por parada.
  - `Duration`: límite de horas por conductor.
  - `Capacity`: 1 unidad por vehículo (solo puede llevar un vehículo a la vez).
- **Restricciones pickup-delivery**: mismo vehículo, pickup siempre antes que delivery.
- **Restricciones de paralelización**: fuerza distribuir trabajo entre múltiples conductores cuando es posible, en vez de concentrar todo en uno.

### Estrategias de negocio (`QuickPlanStrategy`)

Documentadas en detalle en `src/main/resources/docs/STRATEGIES.md` (409 líneas, vale la pena leerla directamente si se va a tocar este código):

| Estrategia | Comportamiento |
|---|---|
| `SMART_CHASE` | Penaliza 10x las "empty miles" (millas vacías: viaje delivery→pickup sin carga), para minimizar uso de Uber/chase vans |
| `MINIMIZE_DISTANCE` (default) | Sin penalizaciones especiales, minimiza distancia total |
| `MINIMIZE_TIME` | `setGlobalSpanCostCoefficient(100)` — minimiza el makespan (hora en que termina el último conductor) |
| `MINIMIZE_COST` | Penalización moderada: 5x empty miles, 1.5x start/end |
| `MINIMIZE_DRIVERS` | Calcula el mínimo de conductores necesarios (`FLEXIBLE`=1 conductor; `STRICT`=greedy por deadline) |

### Deadline Strictness (`STRICT` / `FLEXIBLE`)

Controla si las fechas límite son duras o relajables. Incluye:
- **Detección automática de deadlines inalcanzables** (`isDeadlineFeasible`), en vez de fallar silenciosamente.
- **Reintentos en cascada** cuando el solver no encuentra solución:
  1. Reintenta con estrategia relajada (`SMART_CHASE` → `MINIMIZE_DISTANCE`).
  2. Reintenta en modo `FLEXIBLE`.
  3. Reintenta sin restricciones de paralelización.
- Códigos de error tipados (`RoutingErrorCode`): `NO_SOLUTION`, `NOT_STRICT`, `STRATEGY_ERROR`, `NOT_PARALLELIZED`, `NOT_ENOUGH_HOURS`.

### Driver Grouping / ride sharing (post-procesamiento)

`strategy/DriverGroupingAnalyzer`: detecta oportunidades donde 2+ conductores podrían compartir un mismo Uber/chase vehicle (dentro de 15 min y 500m entre sí), calculando el ahorro estimado (`DriverGroupingOpportunity`).

### Parámetros del solver

- Tiempo límite: 5s (modo normal) / 30s (modo relajado en reintentos).
- `FirstSolutionStrategy`: `PARALLEL_CHEAPEST_INSERTION` o `AUTOMATIC`.

## 🔄 Flujo end-to-end de un Quick Plan

1. `POST /actions/machina` con `GetMachinaQuickPlanRequest` (acciones con orgId, GPS origen/destino, ready/due epoch millis, start/end location, máx. conductores, máx. horas por conductor, strategy, deadlineStrictness, provider).
2. `ServiceProvider.detect()` fuerza HYBRID (path contiene `actions/machina`).
3. `HybridProcurementService` invoca OR-Tools (in-process) o NextBillion (externo) según el provider pedido.
4. OR-Tools arma el modelo VRP, resuelve, y produce una `RouteSolution` (rutas por conductor, timestamps estimados, distancia total, oportunidades de agrupación).
5. Se enriquece con datos reales de actividades vía `legacyUtility.getActivities()` (llamada al backend legacy/Salesforce).
6. Se convierte a `GetMachinaQuickPlanResponse`.

También existe `GetMachinaMultiplePlanResponse` (`/actions/machina/multiple`) para generar **varias alternativas de plan**, no solo la óptima — pensado para que un dispatcher humano elija.

## 🔗 Ver también

- Nota técnica: `Draiver Procurement Microservice - Arquitectura Técnica.md`
- Nota de negocio: `Draiver Procurement Microservice - Overview de Negocio.md`
- Doc fuente en el repo: `src/main/resources/docs/STRATEGIES.md`
