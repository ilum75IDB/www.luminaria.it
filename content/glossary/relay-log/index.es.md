---
title: "Relay log"
description: "Archivo de log intermedio en el slave MySQL que recibe los eventos del binary log del master antes de ser ejecutados localmente."
translationKey: "glossary_relay-log"
articles:
  - "/posts/mysql/binary-log-mysql"
---

El **relay log** es un archivo de log intermedio presente en el slave en una arquitectura de replicación MySQL. Contiene los eventos recibidos del binary log del master, a la espera de ser ejecutados localmente por el hilo SQL del slave.

## Cómo funciona

El flujo de la replicación MySQL pasa por el relay log en tres fases:

1. El **I/O thread** del slave se conecta al master y lee los binary log
2. Los eventos recibidos se escriben en el relay log local
3. El **SQL thread** del slave lee los eventos del relay log y los ejecuta en la base de datos local

Esta arquitectura de dos hilos permite desacoplar la recepción de datos de su aplicación: el slave puede seguir recibiendo eventos del master incluso si la ejecución local está temporalmente ralentizada.

## Para qué sirve

El relay log es el mecanismo que garantiza la consistencia de la replicación. Actúa como buffer entre el master y la aplicación local de los eventos, permitiendo al slave gestionar diferencias de velocidad sin perder datos.

## Cuándo se usa

El relay log se crea automáticamente cuando se configura una réplica MySQL. No requiere gestión manual directa, pero su estado (posición actual, posible retardo) es visible a través de `SHOW REPLICA STATUS` y es fundamental para diagnosticar problemas de replica lag.
