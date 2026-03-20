---
title: "Fact table"
description: "Tabla central del star schema que contiene las medidas numéricas y las claves foráneas hacia las tablas dimensionales."
translationKey: "glossary_fact_table"
aka: "Tabla de hechos"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
  - "/posts/data-warehouse/fatto-grana-sbagliata"
  - "/posts/data-warehouse/partitioning-dwh"
---

La **fact table** (tabla de hechos) es la tabla central de un star schema en el data warehouse. Contiene las medidas numéricas — importes, cantidades, conteos, duraciones — y las claves foráneas que la conectan con las tablas dimensionales.

## Estructura

Cada fila de la fact table representa un evento o una transacción de negocio: una venta, un siniestro, un envío, un acceso. Las columnas se dividen en dos categorías:

- **Claves foráneas** (foreign keys): apuntan a las tablas dimensionales (quién, qué, dónde, cuándo)
- **Medidas**: los valores numéricos a agregar (importe, cantidad, margen)

## Tipos de fact tables

- **Transaction fact**: una fila por cada evento (ej. cada venta)
- **Periodic snapshot**: una fila por período por entidad (ej. saldo mensual por cuenta)
- **Accumulating snapshot**: una fila por proceso, actualizada en cada milestone (ej. ciclo pedido-envío-facturación)

## Relación con las SCD

Cuando las dimensiones usan SCD Tipo 2, la fact table apunta a la clave subrogada de la dimensión — no a la clave natural. Esto garantiza que cada hecho esté asociado a la versión de la dimensión correcta para el momento en que ocurrió.
