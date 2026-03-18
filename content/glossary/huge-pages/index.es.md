---
title: "Huge Pages"
description: "Páginas de memoria de 2 MB (en lugar de los 4 KB estándar) que reducen drásticamente la presión sobre la MMU y el TLB, mejorando el rendimiento de Oracle en Linux."
translationKey: "glossary_huge-pages"
aka: "HugePages / Large Pages"
articles:
  - "/posts/oracle/oracle-linux-kernel"
---

Las **Huge Pages** son páginas de memoria de 2 MB, frente a los 4 KB estándar de Linux. Para una SGA Oracle de 64 GB, pasar de páginas de 4 KB (16,7 millones de páginas) a Huge Pages de 2 MB (32.768 páginas) reduce en 500 veces el número de entradas en la Page Table.

## Cómo funciona

Se configuran mediante el parámetro del kernel `vm.nr_hugepages` en `/etc/sysctl.d/`. El número necesario se calcula dividiendo el tamaño de la SGA entre 2 MB y añadiendo un margen del 1,5%. Tras reiniciar la instancia Oracle, la SGA se asigna en Huge Pages, verificable desde `/proc/meminfo`.

## Para qué sirve

Reducen la presión sobre el TLB (Translation Lookaside Buffer) de la CPU, que solo puede almacenar unas pocas miles de traducciones de dirección. Con páginas normales, el TLB se desborda constantemente y la MMU debe gestionar millones de traducciones — con un impacto medible en latch free waits y library cache contention.

## Por qué es crítico

Es el parámetro individual más impactante para Oracle en Linux, y el que se ignora con más frecuencia. El asistente de instalación no lo configura, la documentación está en una nota MOS, y el sistema "funciona sin él." Pero las métricas antes/después hablan claro: library cache hit ratio del 92% al 99,7%, CPU del 78% al 41%.
