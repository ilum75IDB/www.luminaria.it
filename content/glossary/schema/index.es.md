---
title: "Schema"
description: "Namespace lógico dentro de una base de datos que agrupa tablas, vistas, funciones y otros objetos, permitiendo organización y separación de permisos."
translationKey: "glossary_schema"
aka: "Esquema de base de datos"
articles:
  - "/posts/postgresql/postgresql_roles_and_users"
---

Un **Schema** en una base de datos relacional es un namespace lógico que agrupa objetos como tablas, vistas, funciones y secuencias. Funciona como un contenedor organizativo dentro de una base de datos.

## Cómo funciona

En PostgreSQL, el schema predeterminado es `public`. Para acceder a un objeto en otro schema se necesita el prefijo: `schema1.tabla`. El privilegio `USAGE` sobre un schema es prerequisito para acceder a cualquier objeto dentro de él — sin `USAGE`, ni siquiera un `GRANT SELECT` sobre las tablas funciona.

## Para qué sirve

Los schemas permiten separar lógicamente los datos: un schema para la aplicación, uno para reporting, uno para las tablas de staging. En Oracle, el concepto es diferente: cada usuario es automáticamente un schema, y los objetos creados por ese usuario viven en su schema. En PostgreSQL, schemas y usuarios son entidades independientes.

## Por qué es crítico

La gestión de permisos sobre los schemas es la fuente más común de errores al crear usuarios con acceso limitado. Olvidar el `GRANT USAGE ON SCHEMA` es el error clásico que genera "permission denied for schema" incluso cuando los permisos sobre las tablas son correctos.
