---
title: "Kimball"
description: "Ralph Kimball — metodología de diseño de data warehouse basada en dimensional modeling, star schemas y procesos ETL bottom-up."
translationKey: "glossary_kimball"
aka: "Metodología Kimball, Dimensional Modeling"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
---

**Kimball** se refiere a Ralph Kimball y su metodología de diseño de data warehouses, descrita en el libro *The Data Warehouse Toolkit* (primera edición 1996, tercera edición 2013).

## El enfoque

La metodología Kimball se basa en tres pilares:

- **Dimensional modeling**: organizar los datos en star schemas con fact tables y dimension tables, optimizados para consultas analíticas
- **Bottom-up**: construir el DWH partiendo de los data marts departamentales individuales, integrándolos progresivamente a través de dimensiones conformes (conformed dimensions)
- **Bus architecture**: un framework para garantizar coherencia entre los data marts a través de dimensiones y hechos compartidos

## Slowly Changing Dimensions

Kimball definió la clasificación de las SCD (Slowly Changing Dimensions) en los tipos del 0 al 7, que se ha convertido en el estándar de facto en la industria. El Tipo 2 — con claves subrogadas y fechas de validez — es el más utilizado para rastrear la historia de las dimensiones.

## Kimball vs Inmon

La alternativa principal es la metodología de Bill Inmon, que propone un enfoque top-down con un enterprise data warehouse normalizado (3NF) del que se derivan los data marts. Las dos metodologías no son mutuamente excluyentes y muchos proyectos reales adoptan elementos de ambas.
