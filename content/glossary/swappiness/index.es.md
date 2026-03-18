---
title: "Swappiness"
description: "Parámetro del kernel Linux (vm.swappiness) que controla la propensión del sistema a mover páginas de memoria al swap, crítico para servidores de base de datos donde la SGA debe permanecer en RAM."
translationKey: "glossary_swappiness"
aka: "vm.swappiness"
articles:
  - "/posts/oracle/oracle-linux-kernel"
---

La **Swappiness** (`vm.swappiness`) es un parámetro del kernel Linux que controla cuán agresivamente el sistema mueve páginas de memoria de la RAM al swap en disco. El valor va de 0 (swap solo en caso extremo) a 100 (swap agresivo). El valor por defecto es 60.

## Cómo funciona

Con el valor por defecto 60, Linux comienza a hacer swap cuando la presión sobre la memoria es aún relativamente baja. Para un servidor de base de datos dedicado, esto es inaceptable: la SGA debe permanecer en RAM, siempre. El valor recomendado para Oracle es 1 — no 0, que deshabilitaría completamente el swap y podría provocar el OOM killer.

## Para qué sirve

El valor 1 le dice al kernel: "Haz swap solo si realmente no hay alternativa." Esto garantiza que la SGA y las estructuras críticas de Oracle permanezcan en memoria física, evitando lecturas de swap (órdenes de magnitud más lentas que la RAM) durante la ejecución de queries.

## Por qué es crítico

Con swappiness en 60, un servidor con 128 GB de RAM y una SGA de 64 GB puede empezar a hacer swap de partes de la SGA incluso con 20-30 GB de RAM libre. El resultado son rendimientos degradados de forma impredecible, con picos de latencia que parecen problemas aplicativos pero son en realidad el sistema operativo moviendo memoria a disco.
