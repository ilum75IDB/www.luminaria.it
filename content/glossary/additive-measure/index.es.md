---
title: "Additive Measure"
description: "Medida numérica en una fact table que puede sumarse a lo largo de todas las dimensiones — importes, cantidades, conteos. Fundamental en el diseño del data warehouse."
translationKey: "glossary_additive_measure"
aka: "Medida aditiva, Fully additive measure"
articles:
  - "/posts/data-warehouse/fatto-grana-sbagliata"
---

Una **additive measure** (medida aditiva) es un valor numérico en una fact table que puede sumarse legítimamente a lo largo de cualquier dimensión: por cliente, por producto, por período, por zona.

## Cómo funciona

Las medidas en las fact tables se clasifican en tres categorías:

- **Aditivas**: pueden sumarse a lo largo de todas las dimensiones (ej. importe de venta, cantidad, coste). Son las más comunes y las más útiles
- **Semi-aditivas**: pueden sumarse a lo largo de algunas dimensiones pero no a lo largo del tiempo (ej. saldo de una cuenta: sumable por sucursal, no por mes)
- **No aditivas**: no pueden sumarse de ninguna manera (ej. porcentajes, ratios, promedios precalculados)

## Para qué sirve

Las medidas aditivas son el corazón de toda fact table porque permiten las agregaciones que el negocio requiere: totales por período, por región, por producto. La regla clave: almacenar siempre los valores atómicos (el detalle), nunca los agregados. De un importe por línea de factura puedes obtener el total mensual; del total mensual no puedes reconstruir las líneas individuales.

## Cuándo se usa

Al diseñar la fact table, cada medida debe clasificarse como aditiva, semi-aditiva o no aditiva. Esto determina qué agregaciones son válidas en los reportes y cuáles producirían resultados incorrectos. Un error común es tratar una medida semi-aditiva (como un saldo) como si fuera aditiva — sumando saldos mensuales para obtener un "total" que no tiene significado de negocio.
