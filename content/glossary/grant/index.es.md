---
title: "GRANT"
description: "Comando SQL para asignar privilegios específicos a un usuario o rol sobre bases de datos, tablas o columnas. En MySQL 8 ya no crea usuarios implícitamente."
translationKey: "glossary_grant"
articles:
  - "/posts/mysql/mysql-users-and-hosts"
  - "/posts/postgresql/postgresql_roles_and_users"
---

**GRANT** es el comando SQL usado para asignar privilegios a un usuario o rol sobre objetos específicos de la base de datos. En MySQL y MariaDB, los privilegios se asignan al par `'usuario'@'host'`, no solo al nombre de usuario.

## Cómo funciona

La sintaxis básica es `GRANT <privilegios> ON <base_de_datos>.<tabla> TO 'usuario'@'host'`. Los privilegios pueden ser granulares (SELECT, INSERT, UPDATE, DELETE) o globales (ALL PRIVILEGES). En MySQL 8, GRANT ya no crea usuarios implícitamente: se necesita primero un `CREATE USER` explícito, luego el GRANT. En MySQL 5.7 y MariaDB, GRANT con `IDENTIFIED BY` crea el usuario y asigna privilegios en un solo comando.

## Para qué sirve

GRANT es el mecanismo fundamental para implementar el control de acceso en bases de datos MySQL/MariaDB. Combinado con el modelo `usuario@host`, permite calibrar los privilegios según el origen de la conexión: acceso completo desde localhost para el DBA, solo lectura desde el application server.

## Cuándo se usa

Cada vez que se crea un usuario o se modifican los permisos. La best practice es asignar siempre el mínimo privilegio necesario (principio del least privilege) y usar `SHOW GRANTS FOR 'usuario'@'host'` para verificar los privilegios efectivos.
