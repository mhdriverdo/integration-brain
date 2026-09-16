---
date: 2026-09-01
tags: [draiver, sftp, refactoring, edi, rivian, resilience4j]
source: opencode-chat
status: approved
---

# Plan de Refactorización SftpService - Garantizar Entrega de Documentos EDI

## 🎯 Objetivos

1. **Garantizar cero pérdida de documentos EDI** con Dead Letter Queue
2. **Reducir timeouts** de 300s → 30s para no bloquear procesamiento de SQS
3. **Implementar retry robusto** con Resilience4j (1 retry = 2 intentos totales)
4. **Session pooling eficiente** con 5 sesiones para cliente Rivian
5. **Soportar entregas parciales** (continuar aunque falle 1 de 4 archivos)
6. **Monitoreo proactivo** con DLQ monitor y alertas

## 📊 Configuración Final Aprobada

| Parámetro | Valor Actual | Valor Nuevo | Justificación |
|-----------|--------------|-------------|---------------|
| **SFTP Operation Timeout** | 300s | **30s** | Evitar bloqueo de threads |
| **SFTP Connect Timeout** | 60s | **15s** | Fallo rápido en conexión |
| **SQS Visibility Timeout** | 300s | **300s** (mantener) | Permite 4 uploads con retries (248s max) |
| **SQS Max Receive Count** | ∞ | **3** | Después → DLQ |
| **Resilience4j Max Attempts** | N/A | **1** (2 intentos totales) | Con SQS = 1×3 = 3 intentos por upload |
| **Session Pool Size** | 1 (cache) | **5** | Para 10 concurrent messages |
| **Health Check Interval** | N/A | **30s** | Mantener sesiones vivas |

## ⏱️ Tiempos de Procesamiento

### Caso Normal (SFTP funciona)
- 4 uploads × ~2s = **8 segundos**
- ✅ Procesamiento exitoso

### SFTP Lento
- 4 uploads × 10s = **40 segundos**
- ✅ Dentro de visibility timeout (300s)

### SFTP Caído (peor caso)
- Por upload: 30s timeout → 2s backoff → 30s timeout = 62s
- 4 uploads × 62s = **248 segundos**
- ✅ Dentro de visibility timeout (300s)
- Después de 3 intentos SQS → DLQ

### Entrega Parcial (2/4 uploads fallan)
- 2 uploads × 62s (fallan) = 124s
- 2 uploads × 2s (éxito) = 4s
- Total: **128 segundos**
- ✅ No relanza excepción → mensaje procesado exitosamente

## 🏗️ Arquitectura

```
┌─────────────────────────────────────────────────┐
│     SQS FIFO Queue (inbound-testing.fifo)       │
│  • Visibility timeout: 300s                     │
│  • Max receive count: 3 → DLQ                   │
└──────────────────┬──────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────┐
│         SqsConsumerService                      │
│  • ON_SUCCESS acknowledgement                   │
│  • Relanza excepciones para retry               │
└──────────────────┬──────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────┐
│      RivianInspectionProcessor                  │
│  • 4 uploads independientes (E1,X3,AF,X928)     │
│  • Falla solo si 4/4 fallan                     │
└──────────────────┬──────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────┐
│           SftpService                           │
│  ┌───────────────────────────────────┐          │
│  │ @Retry (Resilience4j)             │          │
│  │ • Max: 1 retry (2 intentos)       │          │
│  │ • Wait: 2s fijo                   │          │
│  └───────────────────────────────────┘          │
│  ┌───────────────────────────────────┐          │
│  │ SftpSessionPool                   │          │
│  │ • 5 sesiones max                  │          │
│  │ • Health check cada 30s           │          │
│  └───────────────────────────────────┘          │
│  • Operation timeout: 30s                       │
│  • Connect timeout: 15s                         │
└──────────────────┬──────────────────────────────┘
                   │ (3 fallos)
                   ▼
┌─────────────────────────────────────────────────┐
│    DLQ + Monitor                                │
│  • Retention: 14 días                           │
│  • Monitor cada 5 min                           │
│  • Alerta Slack si count > 0                    │
└─────────────────────────────────────────────────┘
```

## 📁 Archivos a Crear (7 nuevos)

1. **`SftpSessionPool.java`** - Pool custom de sesiones SFTP con health checks
2. **`Resilience4jConfig.java`** - Configuración de RetryRegistry
3. **`DlqMonitor.java`** - Monitor de DLQ con alertas Slack
4. **`SftpSessionPoolTest.java`** - Tests del pool
5. **`DlqMonitorTest.java`** - Tests del monitor
6. **`docs/SFTP_REFACTORING.md`** - Documentación técnica
7. **`docs/DLQ_RUNBOOK.md`** - Runbook para mensajes en DLQ

## 📝 Archivos a Modificar (6 existentes)

1. **`pom.xml`** - Agregar dependencias Resilience4j
2. **`application.properties`** - Configuración completa (timeouts, pool, retry, DLQ)
3. **`SqsConfig.java`** - Mantener visibility timeout en 300s
4. **`SftpService.java`** - Refactorización completa:
   - Eliminar: `sessionCache`, `executeWithRetry()`, `cleanupDeadSessions()`
   - Agregar: `SftpSessionPool`, `@Retry`, `@PreDestroy shutdown`
   - Actualizar: timeouts, manejo de sesiones
5. **`RivianInspectionProcessor.java`** - Entregas parciales:
   - Try-catch individual por cada upload (4 total)
   - Lanzar excepción solo si 4/4 fallan
   - Logging detallado de fallos parciales
6. **`SqsConsumerService.java`** - Relanzar excepciones en catch block

## 🧪 Tests a Actualizar/Crear (5)

1. **`SftpServiceTest.java`** - Actualizar con mocks de SftpSessionPool
2. **`RivianInspectionProcessorTest.java`** - Tests de entregas parciales
3. **`SqsConsumerServiceTest.java`** - Test de relanzamiento de excepciones
4. **`SftpSessionPoolTest.java`** - Nuevo, tests completos del pool
5. **`DlqMonitorTest.java`** - Nuevo, tests del monitor

## ⚙️ Configuración AWS Requerida

### 1. Crear DLQ en SQS Console

Para cada ambiente (dev, qa, staging, prod):

```
Nombre: nonprod-dev-microservice-inbound-testing-dlq.fifo
Type: FIFO
Content-Based Deduplication: Enabled
Message Retention: 14 days
```

### 2. Configurar Redrive Policy

En la cola principal, agregar redrive policy:

```json
{
  "maxReceiveCount": "3",
  "deadLetterTargetArn": "arn:aws:sqs:us-east-1:ACCOUNT_ID:nonprod-dev-microservice-inbound-testing-dlq.fifo"
}
```

### 3. Ambientes

- Dev: `nonprod-dev-microservice-inbound-testing-dlq.fifo`
- QA: `nonprod-qa-microservice-inbound-testing-dlq.fifo`
- Staging: `prod-staging-microservice-inbound-testing-dlq.fifo`
- Prod: `prod-microservice-inbound-testing-dlq.fifo`

## 📋 Checklist de Implementación

### Fase 1: Setup (1 día)
- [ ] Agregar dependencias Resilience4j a pom.xml
- [ ] Crear DLQs en AWS para todos los ambientes
- [ ] Configurar redrive policies en SQS queues
- [ ] Agregar configuración en application.properties
- [ ] Crear Resilience4jConfig.java

### Fase 2: Core Refactoring (2-3 días)
- [ ] Implementar SftpSessionPool.java
- [ ] Refactorizar SftpService.java
  - [ ] Eliminar código legacy
  - [ ] Agregar @Retry annotations
  - [ ] Integrar SftpSessionPool
  - [ ] Actualizar timeouts
  - [ ] Agregar @PreDestroy shutdown
- [ ] Modificar SqsConsumerService para relanzar excepciones
- [ ] Actualizar SqsConfig (mantener visibility timeout 300s)

### Fase 3: Entregas Parciales (1 día)
- [ ] Modificar RivianInspectionProcessor
  - [ ] Sección pickup con try-catch individual
  - [ ] Sección dropoff con try-catch individual
  - [ ] Lógica de fallo total vs parcial
  - [ ] Logging apropiado

### Fase 4: Monitoreo (1 día)
- [ ] Implementar DlqMonitor.java
- [ ] Configurar alertas Slack
- [ ] Testear detección de mensajes en DLQ

### Fase 5: Testing (2-3 días)
- [ ] Crear SftpSessionPoolTest.java
- [ ] Actualizar SftpServiceTest.java
- [ ] Actualizar RivianInspectionProcessorTest.java
- [ ] Crear DlqMonitorTest.java
- [ ] Ejecutar tests unitarios (cobertura >80%)
- [ ] Tests de integración en dev
- [ ] Tests de integración en qa

### Fase 6: Deployment (1 día)
- [ ] Deploy a dev → validación
- [ ] Deploy a qa → validación completa
- [ ] Deploy a staging → smoke tests
- [ ] Deploy a prod (blue-green)
- [ ] Monitorear logs y métricas por 24h

**Tiempo Total Estimado: 8-10 días**

## 🚨 Vulnerabilidades Identificadas (Resueltas)

### 1. Timeout en session.connect() sin manejo robusto
**Antes:** 60s timeout, no persistencia  
**Después:** 15s timeout, retry con Resilience4j, DLQ si falla

### 2. ExecutorService sin shutdown graceful
**Antes:** Pool sin cerrar, pérdida de datos en redeploys  
**Después:** @PreDestroy con shutdown de 30s

### 3. Timeout de operaciones SFTP demasiado largo
**Antes:** 300s (5 minutos)  
**Después:** 30s (10x reducción)

### 4. Session cache sin validación proactiva
**Antes:** Solo verificación reactiva isConnected()  
**Después:** Health checks cada 30s con keep-alive

### 5. Múltiples uploads sin atomicidad
**Antes:** Si 1 falla, todos se pierden  
**Después:** Entregas parciales, solo falla si 4/4 fallan

### 6. No hay Dead Letter Queue
**Antes:** Mensajes se pierden después de fallos  
**Después:** DLQ con monitor y alertas

### 7. Exponential backoff inadecuado
**Antes:** 2s, 4s, 6s lineal  
**Después:** Resilience4j con 2s fijo (1 solo retry)

### 8. Channel pooling inexistente
**Antes:** Nuevo channel por operación  
**Después:** Session pool con 5 sesiones reutilizables

### 9. cleanupDeadSessions con race condition
**Antes:** ConcurrentModificationException potencial  
**Después:** Eliminado, reemplazado por health checks del pool

### 10. Falta monitoreo y alertas
**Antes:** Solo logs  
**Después:** DlqMonitor con Slack alerts

## ⚠️ Riesgos y Mitigaciones

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|--------------|---------|------------|
| Timeout 30s muy corto | Media | Alto | Monitorear en dev/qa, ajustar si necesario |
| Session pool bottleneck | Baja | Medio | Pool size=5 suficiente, puede aumentar |
| Entregas parciales generan inconsistencia | Media | Alto | Logging detallado + retry manual desde DLQ |
| DLQ se llena en prod | Baja | Alto | Monitor activo + alertas + runbook |
| Visibility timeout insuficiente | Baja | Crítico | 300s confirmado suficiente para peor caso |

## 📚 Documentación a Crear

### 1. SFTP_REFACTORING.md
- Arquitectura completa
- Explicación de configuraciones
- Troubleshooting guide
- Qué logs buscar
- Métricas a monitorear

### 2. DLQ_RUNBOOK.md
- Qué hacer cuando DLQ alerta
- Cómo inspeccionar mensajes
- Cómo reprocesammientomanual
- Prevención de futuros fallos

## ✅ Verificación Post-Deploy

- [ ] Timeout SFTP reducido a 30s (verificar logs)
- [ ] Resilience4j configurado (1 retry visible en logs)
- [ ] SftpSessionPool creando max 5 sesiones
- [ ] ExecutorService shutdown correcto en restart
- [ ] SqsConsumerService relanza excepciones
- [ ] RivianInspectionProcessor soporta entregas parciales
- [ ] DLQ configurada en todos los ambientes
- [ ] DlqMonitor detectando mensajes
- [ ] Tests pasan (cobertura >80%)
- [ ] Monitoreo por 24h sin issues

## 🔢 Total de Intentos por Mensaje

```
Mensaje SQS recibido
  │
  ├─ Intento SQS #1
  │   ├─ Resilience4j intento 1 (30s timeout)
  │   ├─ Backoff 2s
  │   └─ Resilience4j intento 2 (30s timeout)
  │   └─ Total: ~62s por upload × 4 = 248s
  │   └─ Excepción → NO ACK
  │
  ├─ [Espera 300s visibility timeout]
  │
  ├─ Intento SQS #2 (mismo proceso)
  │   └─ Excepción → NO ACK
  │
  ├─ [Espera 300s]
  │
  ├─ Intento SQS #3 (mismo proceso)
  │   └─ Excepción → NO ACK
  │
  └─ Max receive count (3) alcanzado
      └─ Mensaje → DLQ
```

**Total intentos SFTP:** Hasta 24 (3 SQS × 2 Resilience4j × 4 uploads)  
**Pero:** Si un upload tiene éxito, ese no se reintenta

## 🎯 Decisiones Clave Tomadas

1. **Visibility timeout = 300s** (mantener original)
   - Razón: Garantiza que 4 uploads con retries caben sin duplicación

2. **Resilience4j = 1 retry** (2 intentos totales)
   - Razón: Balance entre reintentos rápidos y no exceder visibility timeout

3. **Session pool = 5 sesiones**
   - Razón: Suficiente para 10 concurrent messages, no todos son Rivian

4. **Entregas parciales aceptadas**
   - Razón: Priorizar entregar algo vs. nada, con logging claro

5. **DLQ después de 3 intentos SQS**
   - Razón: Balance entre dar oportunidad de recuperación y no bloquear queue

6. **No usar circuit breaker**
   - Razón: No hay fallback, queremos que falle para que SQS reintente

## 📞 Contacto

- **Implementador:** Por definir
- **Revisores:** Equipo backend
- **Aprobado por:** Martin Huber
- **Fecha aprobación:** 2026-09-01
