---
title: "Drill-down"
description: "Navegación en reportes desde un nivel agregado hasta un nivel de detalle, típica del análisis OLAP y los data warehouses."
translationKey: "glossary_drill_down"
aka: "Navegación jerárquica, Detalle progresivo"
articles:
  - "/posts/data-warehouse/ragged-hierarchies"
  - "/posts/data-warehouse/fatto-grana-sbagliata"
---

El **drill-down** es la operación de navegación en reportes que permite pasar de un nivel agregado a un nivel de mayor detalle, descendiendo por una jerarquía.

## Cómo funciona

En una jerarquía Top Group → Group → Client:

1. Se parte del nivel más alto: facturación total por Top Group
2. Se hace clic en un Top Group para ver sus Groups (drill-down de primer nivel)
3. Se hace clic en un Group para ver los Clients individuales (drill-down de segundo nivel)

La operación inversa — subir del detalle al agregado — se llama **drill-up** (o roll-up).

## Requisitos para un drill-down correcto

Para funcionar sin errores, el drill-down requiere:

- Una jerarquía **completa**: ningún nivel faltante (sin NULLs)
- **Coherencia de totales**: la suma de los valores a nivel de detalle debe corresponder al total del nivel superior
- **Estructura balanceada**: todas las ramas de la jerarquía deben tener la misma profundidad

Si la jerarquía está desequilibrada (ragged hierarchy), el drill-down produce resultados incompletos o erróneos. El self-parenting resuelve esto balanceando la estructura aguas arriba.

## Drill-down vs filtro

El drill-down no es un simple filtro: es una navegación estructurada a lo largo de una jerarquía predefinida. Un filtro muestra un subconjunto de datos; un drill-down muestra el siguiente nivel de detalle dentro de un contexto jerárquico.
