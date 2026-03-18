---
title: "System Privilege"
description: "Privilegio Oracle que autoriza operaciones globales en la base de datos como CREATE TABLE, CREATE SESSION o ALTER SYSTEM, independientes de cualquier objeto específico."
translationKey: "glossary_system-privilege"
aka: "Privilegio de sistema Oracle"
articles:
  - "/posts/oracle/oracle-roles-privileges"
---

Un **System Privilege** en Oracle es una autorización que permite ejecutar operaciones globales en la base de datos, independientemente de un objeto específico. Ejemplos típicos incluyen `CREATE TABLE`, `CREATE SESSION`, `ALTER SYSTEM`, `CREATE USER` y `DROP ANY TABLE`.

## Cómo funciona

Los system privileges se asignan con `GRANT` y se revocan con `REVOKE`. Pueden asignarse directamente a un usuario o a un rol. El rol predefinido `DBA` incluye más de 200 system privileges, razón por la cual asignarlo a usuarios aplicativos es una práctica peligrosa.

## Para qué sirve

Los system privileges definen lo que un usuario puede hacer a nivel de base de datos: crear objetos, gestionar usuarios, modificar parámetros de sistema. Son el nivel más alto de autorización en Oracle y deben gestionarse con extrema cautela, siguiendo el principio del mínimo privilegio.

## Qué puede salir mal

Un system privilege como `DROP ANY TABLE` permite eliminar cualquier tabla de cualquier schema. Si se asigna por error a un usuario aplicativo, un solo comando puede destruir datos de producción. La distinción entre system privileges y object privileges es fundamental para construir un modelo de seguridad robusto.
