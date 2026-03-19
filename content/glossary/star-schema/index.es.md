---
title: "Star schema"
description: "Modelo de datos típico del data warehouse: una fact table en el centro conectada a múltiples tablas dimensionales mediante claves foráneas."
translationKey: "glossary_star_schema"
aka: "Esquema en estrella"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
  - "/posts/data-warehouse/fatto-grana-sbagliata"
---

El **star schema** (esquema en estrella) es el modelo de datos más utilizado en el data warehouse. Toma su nombre de su forma: una tabla central de hechos (fact table) conectada a múltiples tablas dimensionales que la rodean, como los rayos de una estrella.

## Estructura

- **Fact table** en el centro: contiene las medidas numéricas y las claves foráneas hacia las dimensiones
- **Dimension tables** alrededor: contienen los atributos descriptivos (quién, qué, dónde, cuándo) con estructura desnormalizada

Las dimensiones en un star schema están típicamente desnormalizadas — todos los atributos en una sola tabla plana, sin jerarquías normalizadas. Esto simplifica las consultas y mejora el rendimiento de las agregaciones.

## Por qué funciona

El star schema está optimizado para consultas analíticas:

- Los joins son simples: la fact se conecta directamente a cada dimensión con un solo join
- Las agregaciones son rápidas: los optimizadores de las bases de datos reconocen el patrón y lo optimizan
- Es intuitivo para los usuarios de negocio: la estructura refleja cómo piensan sobre los datos (ventas por producto, por región, por período)

## Star schema vs Snowflake

El **snowflake schema** normaliza las dimensiones, dividiéndolas en sub-tablas. Ahorra espacio pero complica las consultas con joins adicionales. En la práctica, el star schema es preferido en la mayoría de los casos porque la simplicidad de las consultas compensa ampliamente el coste del espacio extra en las dimensiones.
