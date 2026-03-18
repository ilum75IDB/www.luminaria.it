---
title: "Object Privilege"
description: "Privilegio Oracle que autoriza operaciones sobre un objeto específico de la base de datos como SELECT, INSERT, UPDATE o EXECUTE sobre una tabla, vista o procedimiento."
translationKey: "glossary_object-privilege"
aka: "Privilegio de objeto Oracle"
articles:
  - "/posts/oracle/oracle-roles-privileges"
---

Un **Object Privilege** en Oracle es una autorización que permite ejecutar operaciones sobre un objeto específico de la base de datos: una tabla, una vista, una secuencia o un procedimiento PL/SQL. Ejemplos típicos incluyen `SELECT ON schema.tabla`, `INSERT ON schema.tabla` y `EXECUTE ON schema.procedimiento`.

## Cómo funciona

Los object privileges se asignan con `GRANT` especificando la operación y el objeto destino: `GRANT SELECT ON app_owner.clientes TO srv_report`. Pueden asignarse a usuarios individuales o a roles. A diferencia de los system privileges, operan sobre un único objeto y no confieren poderes globales sobre la base de datos.

## Para qué sirve

Los object privileges son la herramienta principal para implementar el principio del mínimo privilegio en Oracle. Permiten construir modelos de acceso granulares: un usuario de reporting obtiene solo SELECT, un usuario aplicativo obtiene SELECT + INSERT + UPDATE sobre las tablas operativas, y así sucesivamente. Combinados con roles personalizados, crean arquitecturas de seguridad limpias y mantenibles.

## Por qué es crítico

La diferencia entre un `GRANT SELECT ON app_owner.clientes` y un `GRANT DBA` es la diferencia entre dar la llave de una habitación y dar las llaves de todo el edificio. En entornos con cientos de tablas, los object privileges se gestionan típicamente mediante bloques PL/SQL que generan los grants automáticamente para todas las tablas de un schema.
