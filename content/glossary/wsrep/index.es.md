---
title: "WSREP"
description: "Write Set Replication — API y protocolo de replicación síncrona usado por Galera Cluster para mantener los nodos del cluster alineados en tiempo real."
translationKey: "glossary_wsrep"
aka: "Write Set Replication"
articles:
  - "/posts/mysql/galera-cluster-3-nodi"
---

**WSREP** (Write Set Replication) es la API y el protocolo que Galera Cluster utiliza para la replicación síncrona multi-master. Cada transacción se captura como "write set" (conjunto de cambios a nivel de fila) y se replica en todos los nodos del cluster antes del commit.

## Cómo funciona

Cuando un nodo ejecuta una transacción, WSREP la intercepta en el momento del commit, la empaqueta como write set y la envía a todos los nodos del cluster a través del protocolo de comunicación de grupo. Cada nodo ejecuta un proceso de **certification**: verifica que la transacción no entre en conflicto con otras transacciones concurrentes. Si la certification tiene éxito, todos los nodos aplican la transacción. Si falla, la transacción se revierte en el nodo que la originó.

## Para qué sirve

WSREP garantiza que todos los nodos del cluster tengan los mismos datos en todo momento (replicación síncrona). A diferencia de la replicación asíncrona tradicional de MySQL, no hay retardo entre master y slave: cuando una transacción se confirma en un nodo, ya está presente en todos los demás.

## Cuándo se usa

WSREP se activa con el parámetro `wsrep_on=ON` en la configuración de MariaDB/Percona XtraDB Cluster. Las variables de estado que comienzan con `wsrep_` (como `wsrep_cluster_size`, `wsrep_cluster_status`, `wsrep_flow_control_paused`) son los indicadores principales para monitorizar la salud del cluster.
