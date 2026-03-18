---
title: "Data Warehouse"
description: "Sistema centralizado de recopilación e historización de datos de fuentes diversas, diseñado para el análisis y el soporte a las decisiones empresariales."
translationKey: "glossary_data-warehouse"
articles:
  - "/posts/project-management/4-milioni-nessun-software"
---

Un **Data Warehouse** (DWH) es un sistema de almacenamiento de datos diseñado específicamente para el análisis, el reporting y el soporte a las decisiones empresariales. A diferencia de las bases de datos operacionales (OLTP), un DWH recopila datos de múltiples fuentes, los transforma y los organiza en estructuras optimizadas para consultas analíticas.

## Cómo funciona

Los datos se extraen de los sistemas fuente (gestionales, CRM, ERP), se transforman mediante procesos ETL que los limpian, normalizan y enriquecen, y finalmente se cargan en el DWH. El modelo de datos típico es el star schema: una fact table central con las medidas numéricas conectada a tablas dimensionales que describen el contexto (tiempo, cliente, producto, geografía).

## Para qué sirve

Un DWH permite responder a preguntas de negocio que los sistemas operacionales no pueden gestionar: tendencias históricas, análisis comparativos entre períodos, agregaciones cross-system, KPIs empresariales. Separa la carga analítica de la transaccional, evitando que las queries de reporting impacten el rendimiento de las aplicaciones operativas.

## Cuándo se usa

Un DWH es necesario cuando una empresa necesita integrar datos de fuentes diversas para producir análisis consolidados. La complejidad y los costes dependen del número de sistemas fuente, el volumen de datos y la frecuencia de actualización requerida.
