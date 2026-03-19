---
title: "ETL"
description: "Extract, Transform, Load — proceso de extraccion, transformacion y carga de datos desde los sistemas fuente al data warehouse."
translationKey: "glossary_etl"
aka: "Extract, Transform, Load"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
  - "/posts/data-warehouse/ragged-hierarchies"
  - "/posts/data-warehouse/fatto-grana-sbagliata"
---

**ETL** (Extract, Transform, Load) es el proceso fundamental a traves del cual los datos se mueven desde los sistemas fuente (bases de datos operacionales, archivos, APIs) al data warehouse.

## Las tres fases

- **Extract**: extraccion de datos de los sistemas fuente. Puede ser completa (full load) o incremental (solo datos nuevos o modificados)
- **Transform**: limpieza, validacion, estandarizacion y enriquecimiento de los datos. Aqui se aplican las reglas de negocio, las lookup sobre las dimensiones, los calculos derivados
- **Load**: carga de los datos transformados en las tablas del data warehouse (fact y dimension)

## Por que es critico

El ETL es la parte menos visible pero mas critica de un data warehouse. Si los datos se extraen de forma incompleta, se transforman con reglas erroneas o se cargan sin controles, todo lo que esta encima — reportes, dashboards, decisiones — sera incorrecto.

Un ETL bien disenado tambien determina la ventana de carga: cuanto tiempo se necesita para actualizar el data warehouse. En entornos reales, pasar de 4 horas a 25 minutos puede hacer la diferencia entre datos actualizados por la manana o por la tarde.

## ELT vs ETL

Con la llegada de los data warehouses cloud y los motores columnares de alto rendimiento, se ha difundido el patron **ELT** (Extract, Load, Transform): los datos se cargan sin procesar en el warehouse y se transforman directamente alli, aprovechando la potencia de calculo del motor SQL. El concepto de fondo es el mismo, lo que cambia es donde ocurre la transformacion.
