---
title: "Split-brain"
description: "Condición crítica en un cluster de bases de datos donde dos o más partes operan independientemente, aceptando escrituras divergentes sobre los mismos datos."
translationKey: "glossary_split-brain"
articles:
  - "/posts/mysql/galera-cluster-3-nodi"
---

El **split-brain** es una condición crítica que se produce cuando un cluster de bases de datos se divide en dos o más particiones que no pueden comunicarse entre sí, y cada partición continúa aceptando escrituras independientemente. El resultado son datos divergentes imposibles de reconciliar automáticamente.

## Cómo funciona

En un cluster de 3 nodos, si la red entre el Nodo 1 y los Nodos 2-3 se interrumpe, sin protección del quorum ambas partes podrían seguir aceptando escrituras. Cuando la red se restablece, el cluster se encontraría con dos versiones diferentes de los mismos datos. El mecanismo de quorum previene este escenario: solo la partición con la mayoría de nodos (quorum) puede seguir operando.

## Para qué sirve

Comprender el split-brain es fundamental para diseñar clusters de bases de datos fiables. Es la razón principal por la que Galera Cluster requiere un número impar de nodos (3, 5, 7) e implementa el mecanismo de quorum. Con un número par de nodos, una partición de red puede dividir el cluster en dos mitades iguales, ninguna de las cuales tiene quorum.

## Cuándo se usa

El término split-brain describe un riesgo a evitar, no una funcionalidad a activar. En Galera, la protección es automática: los nodos que pierden el quorum pasan al estado Non-Primary y rechazan las escrituras. El parámetro `pc.ignore_quorum` desactiva esta protección, pero usarlo en producción está fuertemente desaconsejado.
