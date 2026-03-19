---
title: "Grain"
description: "El nivel de detalle de una fact table en un data warehouse — la decisión de diseño que determina qué preguntas puede responder el modelo dimensional."
translationKey: "glossary_grain"
aka: "Granularidad, Nivel de detalle"
articles:
  - "/posts/data-warehouse/fatto-grana-sbagliata"
---

El **grain** (granularidad) es el nivel de detalle de una fact table en un data warehouse. Define qué representa una fila individual: una transacción, un resumen diario, un total mensual, una línea de factura.

## Cómo funciona

La elección del grain es la primera decisión al diseñar una fact table. Todas las demás decisiones — medidas, dimensiones, ETL — derivan de ella:

- **Grain fino** (ej. línea de factura): máxima flexibilidad en consultas, más filas a gestionar
- **Grain agregado** (ej. total mensual por cliente): menos filas, consultas más rápidas, pero imposibilidad de bajar al detalle

El principio fundamental de Kimball: siempre modelar al nivel de detalle más fino disponible en el sistema fuente.

## Para qué sirve

El grain determina:

- Qué **preguntas** puede responder el data warehouse
- Qué **dimensiones** son necesarias (un grain a nivel de línea requiere dim_producto, un grain mensual no)
- Qué tan grande es la fact table y cuánto dura el ETL
- Si el **drill-down** en los reportes es posible o no

## Cuándo se usa

El grain se define en la fase de diseño del modelo dimensional, antes de escribir cualquier DDL o ETL. Cambiar el grain después del go-live equivale a reconstruir el data warehouse desde cero — razón por la cual la elección inicial es tan crítica.
