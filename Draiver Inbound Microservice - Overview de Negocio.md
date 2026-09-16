---
date: 2026-09-15
tags: [draiver, inbound-microservice, negocio, overview]
source: opencode-chat
status: draft
---

# Draiver Inbound Microservice - Overview de Negocio

## 🎯 ¿Qué problema resuelve?

Draiver coordina el transporte de vehículos (car hauling) entre distintos actores del ecosistema: haulers, brokers, plataformas de despacho, fabricantes/OEMs, etc. Cada uno de estos actores externos tiene su **propio sistema y su propia API**, con formatos de datos distintos.

El **inbound microservice** es el punto de entrada que recibe toda la información que llega desde esos sistemas externos (actualizaciones de estado de un viaje, confirmaciones de recogida/entrega, documentos de inspección, daños reportados, etc.) y la traduce al formato interno de Draiver para que el resto de la plataforma pueda operar sobre esa información de manera uniforme.

En una frase: **es el "traductor" entre 8+ sistemas externos y el core de Draiver.**

## 🧩 Clientes/integraciones soportadas

| Cliente | Qué es | Mecanismo de entrada |
|---|---|---|
| **AutoFleet** | Plataforma de gestión de flotas | Webhook/API |
| **ShipCars** | Marketplace de car shipping | Webhook (incluye variante `shipcars-webhook`) |
| **SuperDispatch** | Plataforma de despacho para car haulers | Webhook/API |
| **IMS** | Sistema de gestión de inspecciones | Webhook/API |
| **RunBuggy** | Marketplace de transporte de vehículos | Webhook/API |
| **TruckMovers** | Broker/operador de camiones | Webhook/API |
| **OpenLane** | Plataforma de subastas/logística de autos usados | Webhook/API |
| **Rivian** | Fabricante de vehículos (OEM) | **SFTP con documentos EDI** (X12: 214, 120, 204) |
| **Morgan** | Cliente adicional integrado | Webhook/API |

Cada integración es independiente: tiene sus propios DTOs, su propia autenticación, y su propio "recipe" (flujo de procesamiento).

## 🔄 ¿Cómo fluye la información?

1. Un sistema externo envía datos (webhook HTTP, o archivo por SFTP en el caso de Rivian con EDI).
2. El servicio identifica de qué cliente viene el dato y ejecuta la "receta" (recipe) correspondiente a ese cliente.
3. La receta transforma el dato externo al modelo interno de Draiver, hace validaciones/enriquecimiento adicional (por ejemplo, pedir más info a la API del cliente), y persiste el resultado.
4. El evento se publica de forma asíncrona (cola SQS FIFO → stream Kinesis) para que otros sistemas de Draiver puedan reaccionar.
5. Opcionalmente, se envía una confirmación/actualización de vuelta al sistema externo (acknowledgment).

Esto permite que, aunque cada cliente tenga un formato y flujo distinto, todos terminen alimentando el mismo pipeline interno de datos de Draiver.

## 🏗️ Patrón de "Recetas" (por qué importa para el negocio)

Cada cliente tiene un **recipe** (receta) definida como una configuración JSON con pasos secuenciales (por ejemplo: convertir datos → buscar información adicional → actualizar sistema interno → notificar al cliente).

Esto es clave desde una perspectiva de negocio porque:
- **Agregar un nuevo cliente/integración no requiere reescribir todo el sistema**, sino definir una nueva receta con los pasos reutilizables que ya existen (o crear pasos nuevos si el cliente tiene necesidades particulares).
- Permite **trazabilidad**: cada ejecución de receta tiene un ID único, lo que facilita diagnosticar problemas con un cliente puntual.
- Si una integración falla, el sistema **notifica automáticamente por Slack**, dando visibilidad rápida al equipo.

## 📦 Tipos de información que procesa

- Confirmaciones y actualizaciones de estado de viajes (recogida, tránsito, entrega)
- Inspecciones de vehículos, incluyendo fotos y reportes de daños
- Documentos EDI (para Rivian): transacciones estándar de la industria automotriz/logística (X12 214 = actualización de estado de envío, 120 = solicitud de transporte de vehículos, 204 = tender de carga)
- Cotizaciones (quotes) y órdenes desde marketplaces de despacho

## 💼 Por qué es crítico para Draiver

- Es el **único punto de entrada** de datos de todos los partners externos: si falla, Draiver pierde visibilidad en tiempo real de lo que pasa con los vehículos en tránsito.
- Maneja **múltiples formatos y protocolos distintos** (REST webhooks vs. SFTP/EDI) bajo una misma arquitectura, lo que reduce el costo de mantener 8+ integraciones diferentes.
- Cualquier degradación (timeouts, pérdida de mensajes, fallos silenciosos) tiene impacto directo en la relación con clientes/partners, porque implica retrasos o falta de confirmación de eventos reales del negocio (ej: un vehículo entregado que no se refleja en el sistema).

## 🔗 Ver también

- Nota técnica: `2026-09-15 - Draiver Inbound Microservice - Arquitectura Técnica.md`
- `2026-09-01 - Plan Refactorización SftpService - Garantizar Entrega de Documentos EDI.md` (caso concreto de mejora de resiliencia para la integración con Rivian)
