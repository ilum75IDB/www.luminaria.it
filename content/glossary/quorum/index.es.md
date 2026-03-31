---
title: "Quorum"
description: "Mecanismo de consenso basado en la mayoría de nodos, usado en clusters de bases de datos para prevenir el split-brain y garantizar la consistencia de datos."
translationKey: "glossary_quorum"
articles:
  - "/posts/mysql/mysql-group-replication-binlog-migration"
  - "/posts/mysql/galera-cluster-3-nodi"
---

El **quorum** es el número mínimo de nodos que deben estar de acuerdo para considerar el cluster operativo. En un cluster de 3 nodos, el quorum es 2 (la mayoría). Si un nodo se desconecta, los otros dos reconocen que son la mayoría y continúan operando normalmente.

## Cómo funciona

Galera Cluster utiliza un protocolo de comunicación de grupo que verifica continuamente cuántos nodos son alcanzables. El cálculo es simple: quorum = (número total de nodos / 2) + 1. Con 3 nodos el quorum es 2, con 5 nodos es 3. Los nodos que pierden el quorum pasan al estado Non-Primary y rechazan las escrituras para evitar inconsistencias.

## Para qué sirve

El quorum previene el **split-brain**: la situación en la que dos partes del cluster operan independientemente, aceptando escrituras diferentes sobre los mismos datos. Sin quorum, una interrupción de red podría llevar a datos divergentes imposibles de reconciliar automáticamente.

## Cuándo se usa

El quorum está activo automáticamente en cualquier cluster Galera. Por esta razón, **tres nodos es el mínimo en producción**: con dos nodos, la pérdida de uno deja al superviviente sin quorum y por tanto bloqueado.
