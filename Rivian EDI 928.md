---
title: Rivian EDI 928 - Automotive Inspection Detail
type: edi-specification
version: 1.2
company: Rivian
date: 2021-06-16
tags:
  - edi
  - rivian
  - logistics
  - 928
  - automotive
---

# 🚚 Rivian EDI 928: Automotive Inspection Detail (v1.2)

> [!info] **Propósito del Estándar**
> El conjunto de transacciones **EDI 928** (X12/V4010) establece el formato y contenido de datos para el **Informe de Inspección Automotriz**. Es utilizado por Rivian, transportistas (*carriers*) y agencias de inspección para registrar la condición física de los vehículos (VIN), la ubicación de inspección (Origen/Destino) y el detalle exhaustivo de daños sufridos durante el tránsito.

---

## 📌 Estructura Principal de Segmentación

La transacción se organiza en los siguientes segmentos clave:


| Posición | Segmento | Nombre                                      |           Requerimiento           | Uso / Descripción                                                                                                                                     |
| :------: | :------: | :------------------------------------------ | :-------------------------------: | :---------------------------------------------------------------------------------------------------------------------------------------------------- |
| **010**  |   `ST`   | Transaction Set Header                      |     Obligatorio (`Must use`)      | Inicio de transacción y número de control.                                                                                                            |
| **020**  |  `BIX`   | Beginning Segment for Automotive Inspection |     Obligatorio (`Must use`)      | SCAC del transportista, fecha, tipo de ubicación (Origen `10` / Destino `20`) y Rivian Shipment ID (`BIX11`).                                         |
| **030**  |   `TI`   | Transport Information                       |         Opcional (`Used`)         | SCAC del equipo, número de tráiler/equipo (`TI04`) y fecha de envío.                                                                                  |
| **040**  |   `VC`   | Motor Vehicle Control                       |     Obligatorio (`Must use`)      | **Loop VC (Repite hasta 21 veces)**. Identifica el **VIN** (`VC01`), tipo de vehículo (`VC03`), *Bay Location* (`VC06`) e indicador de daño (`VC08`). |
| **050**  |   `ID`   | Inspection Detail                           | Opcional (`Must use` si hay daño) | **Hasta 99 repeticiones por VC**. Detalla área de daño (`ID01`), tipo (`ID02`) y severidad (`ID03`).                                                  |
| **060**  |   `SE`   | Transaction Set Trailer                     |     Obligatorio (`Must use`)      | Cierre de transacción con conteo exacto de segmentos.                                                                                                 |

---

## 🔍 Detalle de Segmenteación y Campos Clave

### 1. `BIX` - Inicio de Inspección
- **`BIX01`**: Propósito de la transacción (`01` Cancelación, `02` Agregar, `04` Cambio).
- **`BIX02`**: SCAC del transportista.
- **`BIX03`**: Fecha de inspección (`CCYYMMDD`).
- **`BIX04`**: Tipo de ubicación de inspección:
  - `10` = Origen (*Origin*)
  - `20` = Destino (*Destination*)
- **`BIX10`**: Calificador de código (`92` = Asignado por el comprador / Rivian).
- **`BIX11`**: **Rivian Shipment ID**.

### 2. `VC` - Control de Vehículo
- **`VC01`**: Número de Identificación del Vehículo (**VIN** - 17 caracteres).
- **`VC03`**: Código de tipo de vehículo (`1` = Automóvil, `V` = Van grande).
- **`VC06`**: Ubicación en patio (*Bay Location*, ej. `R66`).
- **`VC08`**: Indicador de Excepción de Daño (`Y` = Se reportan daños o inspección completada).

### 3. `ID` - Detalle de Daños (Obligatorio si se reportan daños)
> [!warning] **Regla de Formato Crítica**
> Los códigos `ID01` e `ID02` exigen estrictamente una longitud de **2 caracteres** (`Min/Max 2/2`). Si el código es menor a 10, debe formatearse con cero a la izquierda (ej. `01`, `02`, `07`).

- **`ID01`** (Damage Area Code - 2 dígitos): Ubicación del daño (ej. `01` Antena, `03` Bumper frontal, `20` Parabrisas, `27` Capó, `72` Neumático delantero izquierdo).
- **`ID02`** (Damage Type Code - 2 dígitos): Tipo de daño (ej. `01` Doblado, `02` Roto, `04` Abollado con pintura rota, `12` Rayado, `14` Abollado sin daño de pintura).
- **`ID03`** (Damage Severity Code - 1 dígito):
  - `1` = Hasta 1 pulgada
  - `2` = De 1 a 3 pulgadas
  - `3` = De 3 a 6 pulgadas
  - `4` = De 6 a 12 pulgadas
  - `5` = Mayor a 12 pulgadas
  - `6` = Faltante / Daño mayor

---

## 📝 Ejemplos Prácticos

### Ejemplo 1: Inspección completada sin reporte de daño
```text
ST*928*0001~
BIX*01*SCAC*20210614*10******92*21008517~
TI*SCAC***11111*20210609~
VC*8F342390088900828**1***Bay Lo**Y~
SE*5*0001~
```

### Ejemplo 2: Inspección con detalle de daños
```text
ST*928*0001~
BIX*01*SCAC*20210614*20******92*21008517~
TI*SCAC***11111*20210609~
VC*8F342390088900828**1***Bay Lo**Y~
ID*11*01*1~
ID*27*11*6~
ID*21*06*4~
ID*18*13*3~
ID*19*13*5~
SE*10*0001~
```

---

## 💡 Notas de Implementación y Reglas de Sintaxis
- **Conteo SE**: `SE01` debe reflejar el número exacto de segmentos desde `ST` hasta `SE` inclusive.
- **Sintaxis de Delimitadores**: Evitar separadores vacíos sobrantes al final de cada segmento antes de la tilde (`~`).
