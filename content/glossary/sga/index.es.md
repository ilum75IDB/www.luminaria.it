---
title: "SGA"
description: "System Global Area — área de memoria compartida de Oracle Database que contiene buffer cache, shared pool, redo log buffer y otras estructuras críticas para el rendimiento."
translationKey: "glossary_sga"
aka: "System Global Area"
articles:
  - "/posts/oracle/oracle-linux-kernel"
---

La **SGA** (System Global Area) es el área de memoria compartida principal de Oracle Database. Contiene las estructuras de datos fundamentales: buffer cache (páginas de datos leídas de disco), shared pool (planes de ejecución y diccionario de datos), redo log buffer y large pool.

## Cómo funciona

El tamaño de la SGA se controla con el parámetro `SGA_TARGET` o `SGA_MAX_SIZE`. Oracle asigna la SGA al inicio de la instancia en la memoria compartida del sistema operativo. Los parámetros del kernel Linux `shmmax` y `shmall` deben dimensionarse para permitir la asignación completa de la SGA.

## Para qué sirve

Toda la actividad de lectura y escritura de la base de datos pasa a través de la SGA. Un buffer cache eficiente evita lecturas físicas de disco. Un shared pool bien dimensionado evita el re-parsing de las queries. La SGA es el corazón del rendimiento Oracle — y debe residir en Huge Pages para maximizar la eficiencia.

## Por qué es crítico

Una SGA no asignada en Huge Pages significa millones de entradas en la Page Table y overflow constante del TLB. El resultado son latch free waits, library cache contention y CPU elevada. Configurar las Huge Pages y el parámetro `memlock unlimited` para el usuario oracle es el prerequisito para cualquier tuning serio.
