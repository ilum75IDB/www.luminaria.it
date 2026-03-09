---
title: "Data Warehouse"
date: "2026-03-10T10:00:00+01:00"
description: "Arquitectura Data Warehouse en la práctica: modelado dimensional, jerarquías, ETL y estrategias de carga. Cuando los datos no solo deben funcionar, sino servir para decidir."
image: "data-warehouse.cover.jpg"
layout: "list"
---
Un data warehouse no es una base de datos con tablas más grandes.<br>
Es una forma diferente de pensar los datos — orientada al análisis, a la historia, a las decisiones.<br>

La diferencia entre un DWH que funciona y uno que se convierte en un problema está casi siempre en el modelo. Tablas de hechos con la granularidad equivocada, dimensiones mal diseñadas, jerarquías que no soportan las consultas de agregación. Problemas que no se ven en fase de desarrollo, pero explotan cuando el negocio pide reportes que el modelo no puede entregar.<br>

En esta sección comparto casos reales de diseño y reestructuración de data warehouses: modelado dimensional, jerarquías balanceadas, slowly changing dimensions, estrategias de carga. No teoría del libro de Kimball, sino soluciones aplicadas en producción sobre sistemas que sirven decisiones empresariales reales.<br>

Porque un data warehouse no se construye para contener datos.<br>
Se construye para responder preguntas.
