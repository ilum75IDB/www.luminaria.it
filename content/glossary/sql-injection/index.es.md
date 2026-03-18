---
title: "SQL Injection"
description: "Técnica de ataque que inserta código SQL malicioso en los inputs de una aplicación para manipular las queries ejecutadas por la base de datos, potencialmente accediendo a datos no autorizados o comprometiendo el sistema."
translationKey: "glossary_sql-injection"
articles:
  - "/posts/mysql/mysql-multi-istanza-secure-file-priv"
---

La **SQL Injection** es una de las vulnerabilidades más extendidas y peligrosas en aplicaciones web. Se produce cuando los inputs proporcionados por el usuario se insertan directamente en las queries SQL sin validación ni parametrización, permitiendo a un atacante modificar la lógica de la query.

## Cómo funciona

El atacante inserta fragmentos de código SQL en los campos de input de la aplicación (formularios de login, campos de búsqueda, parámetros URL). Si la aplicación concatena estos inputs directamente en las queries SQL, el código malicioso es ejecutado por la base de datos con los privilegios del usuario aplicativo. En combinación con el privilegio `FILE` de MySQL y un `secure-file-priv` no configurado, el atacante puede leer archivos del sistema o escribir archivos arbitrarios en el servidor.

## Para qué sirve

Comprender la SQL injection es fundamental para quien gestiona bases de datos en producción, porque muchas configuraciones de seguridad (como `secure-file-priv`, la gestión de privilegios y la separación de usuarios) existen específicamente para mitigar el impacto de este tipo de ataque.

## Cuándo se usa

El término describe un ataque a prevenir, no una técnica a utilizar. Las contramedidas principales son: queries parametrizadas (prepared statements), validación de inputs, principio de mínimo privilegio para los usuarios de base de datos, y configuración correcta de directivas como `secure-file-priv`.
