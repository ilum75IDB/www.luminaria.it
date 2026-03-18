---
title: "Lift-and-Shift"
description: "Estrategia de migración que traslada un sistema de un entorno a otro sin modificar su arquitectura, código o configuración."
translationKey: "glossary_lift-and-shift"
aka: "Rehosting"
articles:
  - "/posts/project-management/tecnica-si-e-yes-and"
---

**Lift-and-Shift** (rehosting) es una estrategia de migración que consiste en trasladar un sistema de un entorno a otro — típicamente de on-premise a cloud — sin modificar su arquitectura, código aplicativo o configuración. Se toma el sistema tal cual y se "levanta y traslada".

## Cómo funciona

La infraestructura se replica en el entorno de destino: mismas máquinas virtuales, mismas bases de datos, mismo middleware. La ventaja es la velocidad: no hay reescritura de código, no hay rediseño arquitectónico. El riesgo es llevarse todos los problemas del entorno original, incluidas ineficiencias y deuda técnica.

## Cuándo se usa

Cuando la prioridad es salir rápidamente de un datacenter (fin de contrato, desmantelamiento de hardware), cuando el presupuesto no permite una rearquitectura, o como primera fase de una migración incremental donde los componentes se modernizan uno a uno.

## Qué puede salir mal

Un lift-and-shift al cloud sin optimización puede costar más que la infraestructura on-premise original. Las aplicaciones no diseñadas para cloud no aprovechan la elasticidad, el auto-scaling y los servicios gestionados. El resultado es frecuentemente un datacenter privado reconstruido en el cloud a un precio superior.
