---
title: "ROLE"
description: "Entidad fundamental de PostgreSQL que unifica el concepto de usuario y grupo de permisos: un ROLE con LOGIN es un usuario, sin LOGIN es un contenedor de privilegios."
translationKey: "glossary_postgresql-role"
aka: "Rol PostgreSQL"
articles:
  - "/posts/postgresql/postgresql_roles_and_users"
---

En PostgreSQL, **ROLE** es la única entidad de seguridad. No existe distinción entre "usuario" y "rol": todo es un ROLE. Un ROLE con el atributo `LOGIN` se comporta como un usuario; sin `LOGIN`, es un contenedor de privilegios asignable a otros ROLEs.

## Cómo funciona

`CREATE USER mario` es simplemente un atajo para `CREATE ROLE mario WITH LOGIN`. Los ROLEs pueden poseer objetos, heredar privilegios de otros ROLEs a través del atributo `INHERIT`, y ser utilizados para construir jerarquías de permisos. Un ROLE "funcional" (sin LOGIN) agrupa privilegios; los ROLEs "usuario" (con LOGIN) los heredan.

## Para qué sirve

El modelo unificado permite diseñar arquitecturas de seguridad limpias: se crean ROLEs funcionales como `role_readonly` o `role_write`, se asignan privilegios a los ROLEs funcionales, y luego se asignan esos ROLEs a los usuarios reales. Cuando llega un nuevo colega, basta un `GRANT role_readonly TO nuevo_usuario`.

## Por qué es crítico

La simplicidad del modelo es su punto fuerte — pero también una trampa si no se comprende. Muchos administradores asignan privilegios directamente a los usuarios en lugar de usar ROLEs funcionales, creando una maraña de GRANTs imposible de mantener. El modelo mental correcto es: los privilegios van a los ROLEs, los ROLEs van a los usuarios.
