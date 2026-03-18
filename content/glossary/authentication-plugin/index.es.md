---
title: "Authentication Plugin"
description: "Módulo MySQL/MariaDB que gestiona el método de verificación de credenciales durante la conexión. El default cambia entre versiones y puede causar problemas de compatibilidad."
translationKey: "glossary_authentication-plugin"
articles:
  - "/posts/mysql/mysql-users-and-hosts"
---

Un **Authentication Plugin** es el módulo que MySQL o MariaDB usa para verificar las credenciales de un usuario al momento de la conexión. Cada usuario en el sistema está asociado a un plugin específico que determina cómo la contraseña es hasheada, transmitida y verificada.

## Cómo funciona

Los plugins principales son: `mysql_native_password` (default en MySQL 5.7 y MariaDB), que usa un hash SHA1 doble; `caching_sha2_password` (default en MySQL 8.0+), que usa SHA-256 con caching para mejorar seguridad y rendimiento. Cuando un cliente se conecta, debe soportar el plugin del usuario al que intenta autenticarse.

## Para qué sirve

El conocimiento de los plugins de autenticación es esencial durante las migraciones entre versiones o entre MySQL y MariaDB. Un cliente que solo soporta `mysql_native_password` no logra conectarse a un usuario con `caching_sha2_password` — y el error resultante es frecuentemente críptico y difícil de diagnosticar.

## Cuándo se usa

El plugin se especifica al momento de crear el usuario (`CREATE USER ... IDENTIFIED WITH <plugin> BY 'password'`) o se puede verificar y modificar con `ALTER USER`. Cuando se escriben scripts de provisioning que deben funcionar en versiones diferentes de MySQL/MariaDB, es importante especificar explícitamente el plugin.
