---
title: "SST"
description: "State Snapshot Transfer — mecanismo de Galera Cluster para transferir una copia completa de los datos a un nodo que se une al cluster."
translationKey: "glossary_sst"
aka: "State Snapshot Transfer"
articles:
  - "/posts/mysql/galera-cluster-3-nodi"
---

**SST** (State Snapshot Transfer) es el mecanismo por el cual un nodo Galera que se une al cluster (o que ha estado offline demasiado tiempo) recibe una copia completa del dataset entero desde un nodo donante.

## Cómo funciona

Cuando un nodo se une al cluster y el gap de transacciones faltantes supera el tamaño del gcache, el cluster inicia un SST. El nodo donante crea una instantánea completa de la base de datos y la transfiere al nodo receptor. Los métodos disponibles son: `mariabackup` (no bloquea al donante), `rsync` (rápido pero bloquea al donante en lectura), y `mysqldump` (lento y bloqueante).

## Para qué sirve

SST es esencial para dos escenarios: la adición de un nuevo nodo al cluster (primer join) y la recuperación de un nodo que ha estado offline tanto tiempo que las transacciones faltantes ya no están disponibles en el gcache del donante.

## Cuándo se usa

SST se activa automáticamente por Galera cuando es necesario. La elección del método SST (`wsrep_sst_method`) se hace durante la configuración. En producción, `mariabackup` es la opción recomendada porque no bloquea al nodo donante, evitando degradar el cluster durante la transferencia.
