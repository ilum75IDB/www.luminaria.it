---
title: "secure-file-priv"
description: "Directiva de seguridad MySQL que limita los directorios donde el servidor puede leer y escribir archivos, protegiendo el filesystem de operaciones no autorizadas."
translationKey: "glossary_secure-file-priv"
articles:
  - "/posts/mysql/mysql-multi-istanza-secure-file-priv"
---

**secure-file-priv** es una variable de sistema MySQL que controla dónde las instrucciones `LOAD DATA INFILE`, `SELECT INTO OUTFILE` y la función `LOAD_FILE()` pueden operar en el filesystem del servidor.

## Cómo funciona

La variable acepta tres valores: una ruta específica (ej. `/var/lib/mysql-files/`), que limita las operaciones sobre archivos a ese directorio; una cadena vacía (`""`), que no impone restricciones; o `NULL`, que deshabilita completamente las operaciones sobre archivos. El valor solo se puede configurar en el archivo de configuración (`my.cnf`) y requiere reinicio del servicio para modificarse — no se puede cambiar en tiempo de ejecución.

## Para qué sirve

La directiva previene el acceso arbitrario al filesystem por parte de usuarios MySQL con el privilegio `FILE`. Sin esta protección, un atacante que explota una SQL injection podría leer archivos del sistema (ej. `/etc/passwd`, llaves SSH) o escribir web shells en el webroot de un servidor web en el mismo host.

## Cuándo se usa

`secure-file-priv` debe configurarse al momento del setup de cada instancia MySQL, especificando un directorio dedicado. En entornos multi-instancia, cada instancia debería tener su propio directorio `secure-file-priv`. Si el export a archivo está bloqueado, la alternativa recomendada es usar el cliente mysql desde shell con las opciones `-B` y `-e` para redirigir la salida.
