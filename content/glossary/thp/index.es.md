---
title: "THP"
description: "Transparent Huge Pages — función del kernel Linux que promueve automáticamente las páginas normales a páginas grandes, pero que causa latencias impredecibles y debe deshabilitarse para Oracle."
translationKey: "glossary_thp"
aka: "Transparent Huge Pages"
articles:
  - "/posts/oracle/oracle-linux-kernel"
---

Las **THP** (Transparent Huge Pages) son una función del kernel Linux que promueve automáticamente las páginas de memoria de 4 KB a 2 MB en segundo plano, sin configuración explícita. A diferencia de las Huge Pages estáticas, son gestionadas por el proceso del kernel `khugepaged`.

## Cómo funciona

Cuando están activas (por defecto `always`), el kernel intenta compactar las páginas normales en páginas grandes en segundo plano. El proceso `khugepaged` trabaja continuamente para encontrar y unir grupos de páginas contiguas, causando micro-congelamientos impredecibles durante las operaciones de compactación.

## Por qué es crítico

Para Oracle son un desastre. Oracle lo dice explícitamente en su documentación: deshabilitar THP. Los "bloqueos de unos segundos" que los usuarios reportan son frecuentemente causados por `khugepaged`. Se deshabilitan con `echo never > /sys/kernel/mm/transparent_hugepage/enabled` y vía GRUB para persistencia tras reboot.

## Qué puede salir mal

La confusión entre Huge Pages (buenas para Oracle, configuradas estáticamente) y THP (dañinas para Oracle, activas por defecto) es uno de los errores más comunes. Un DBA que ve "Huge Pages" en la documentación y no deshabilita THP está empeorando las cosas en lugar de mejorarlas.
