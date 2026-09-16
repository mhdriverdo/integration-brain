---
tags: [java, refactoring, kinesis, webhook, performance, draiver]
date: 2026-08-26
project: draiver-microservice-webhook-java
---

# Refactorización: Procesamiento Paralelo en StreamRecordProcessor

## Contexto
El método `processRecords` en `StreamRecordProcessor` procesaba los registros de Kinesis de forma secuencial usando `forEach`, lo que podía causar cuellos de botella en el throughput cuando se recibían múltiples registros.

## Cambios Implementados

### 1. Procesamiento Paralelo con CompletableFuture
- **Antes**: Procesamiento secuencial con `forEach`
- **Después**: Procesamiento paralelo usando `CompletableFuture.runAsync()`

### 2. Thread Pool Dedicado
```java
private static final int THREAD_POOL_SIZE = 10;
private final ExecutorService executorService;
```
- Se crea un `ExecutorService` con un pool fijo de **10 threads**
- Permite un control explícito sobre la concurrencia
- Tamaño optimizado para el caso de uso específico del webhook consumer

### 3. Método Extraído: processRecord()
Se extrajo la lógica de procesamiento individual en un método privado para:
- Mejorar la legibilidad
- Facilitar el testing
- Permitir mejor manejo de errores por registro

### 4. Gestión del Ciclo de Vida
- El checkpoint solo se ejecuta después de que **todos** los registros se hayan procesado exitosamente
- Se agregó `shutdownExecutor()` para liberar recursos cuando el shard termina
- El executor se apaga en el método `shutdownRequested()`

## Beneficios

### Performance
- **Throughput mejorado**: Múltiples registros se procesan simultáneamente (hasta 10 concurrentes)
- **Mejor utilización de CPU**: Pool controlado de 10 threads para procesamiento paralelo
- **Reducción de latencia**: Los registros no esperan en cola secuencial

### Resiliencia
- Si un registro falla, el error se propaga correctamente a través del `CompletableFuture`
- El checkpoint solo ocurre si todos los registros se procesan exitosamente
- Logging mejorado con información del `partitionKey`

### Mantenibilidad
- Código más limpio y modular
- Responsabilidades separadas (procesamiento individual vs batch)
- Gestión explícita del ciclo de vida del thread pool

## Consideraciones Importantes

### 1. Orden de Procesamiento
⚠️ **IMPORTANTE**: El procesamiento paralelo no garantiza el orden de los registros. Si el orden es crítico para tu caso de uso, considera:
- Usar procesamiento secuencial
- Implementar una estrategia de ordenamiento por `sequenceNumber`
- Agrupar por `partitionKey` antes de procesar

### 2. Backpressure
El uso de `.join()` espera a que todos los futures completen, lo que puede causar bloqueo si:
- El `StreamConsumerService` es muy lento
- Hay muchos registros en el batch
- Considerar timeouts si es necesario

### 3. Manejo de Recursos
- El `ExecutorService` se crea una vez por instancia del processor
- Se cierra correctamente en el shutdown
- Cada shard tiene su propio thread pool de 10 threads

### 4. Dimensionamiento del Thread Pool
- **10 threads** fue elegido como un balance entre throughput y uso de recursos
- Si los registros son I/O-bound (llamadas HTTP, DB), este número es apropiado
- Si son CPU-bound, considerar ajustar según los cores disponibles

## Ubicación del Código
`src/main/java/com/draiver/webhook/consumer/StreamRecordProcessor.java:29,54-83`

## Próximos Pasos Potenciales
- [ ] Agregar métricas de performance (tiempo de procesamiento paralelo)
- [ ] Implementar circuit breaker si el servicio downstream falla frecuentemente
- [ ] Considerar configurar el tamaño del thread pool externamente (via properties)
- [ ] Agregar tests unitarios específicos para el procesamiento paralelo
- [ ] Monitorear el uso real del thread pool para ajustar el tamaño si es necesario
