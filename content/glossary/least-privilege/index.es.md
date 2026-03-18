---
title: "Least Privilege"
description: "Principio de seguridad que prevé asignar a cada usuario o proceso solo los permisos estrictamente necesarios para desempeñar su función."
translationKey: "glossary_least-privilege"
aka: "Principio del privilegio mínimo"
articles:
  - "/posts/mysql/mysql-users-and-hosts"
  - "/posts/oracle/oracle-roles-privileges"
  - "/posts/postgresql/postgresql_roles_and_users"
---

El **Least Privilege** (principio del privilegio mínimo) es un principio fundamental de la seguridad informática: cada usuario, proceso o sistema debe tener solo los permisos estrictamente necesarios para desempeñar su función, nada más.

## Cómo funciona

En el ámbito de bases de datos, el principio se aplica asignando privilegios granulares: `SELECT` si el usuario solo debe leer, `SELECT + INSERT + UPDATE` si también debe escribir, nunca `ALL PRIVILEGES` si no es estrictamente necesario. Combinado con el modelo `usuario@host` de MySQL, el principio puede aplicarse también según el origen de la conexión.

## Para qué sirve

Limitar los privilegios reduce la superficie de ataque. Si una aplicación es comprometida, el atacante hereda los privilegios del usuario de base de datos de la aplicación. Si ese usuario tiene solo SELECT sobre una base de datos específica, el daño es contenido. Si tiene ALL PRIVILEGES, todo el servidor está en riesgo.

## Cuándo se usa

Siempre. El principio del least privilege es aplicable en cada contexto: usuarios de base de datos, usuarios de sistema operativo, roles aplicativos, cuentas de servicio. La tentación de asignar privilegios amplios "para no tener problemas" es la causa más común de incidentes de seguridad evitables.
