---
title: "DEFAULT PRIVILEGES"
description: "Mecanismo PostgreSQL que define automáticamente los privilegios a asignar a todos los objetos futuros creados en un schema, evitando repetir los GRANT manualmente."
translationKey: "glossary_default-privileges"
aka: "ALTER DEFAULT PRIVILEGES"
articles:
  - "/posts/postgresql/postgresql_roles_and_users"
---

**DEFAULT PRIVILEGES** es un mecanismo de PostgreSQL que permite definir con antelación los privilegios que se asignarán automáticamente a todos los objetos futuros creados en un schema. Se configura con el comando `ALTER DEFAULT PRIVILEGES`.

## Cómo funciona

El comando `ALTER DEFAULT PRIVILEGES IN SCHEMA schema1 GRANT SELECT ON TABLES TO srv_monitoreo` hace que cada nueva tabla creada en `schema1` sea automáticamente legible por `srv_monitoreo`. Sin esta configuración, las tablas futuras requerirían un GRANT manual cada vez.

## Para qué sirve

Es la parte que la mayoría de los administradores olvida al crear usuarios de solo lectura. Los GRANT sobre `ALL TABLES IN SCHEMA` cubren solo las tablas existentes. Las tablas creadas después requieren nuevos GRANT — a menos que se usen los DEFAULT PRIVILEGES. Sin ellos, el usuario de monitoreo deja de funcionar con la primera tabla nueva.

## Qué puede salir mal

Los DEFAULT PRIVILEGES aplican al ROLE que crea los objetos. Si en un schema varios usuarios crean tablas, los default privileges deben configurarse para cada creador. Este detalle causa frecuentemente errores difíciles de diagnosticar: "el GRANT está, pero la nueva tabla no es legible."
