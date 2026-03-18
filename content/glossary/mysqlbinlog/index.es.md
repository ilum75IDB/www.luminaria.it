---
title: "mysqlbinlog"
description: "Utilidad de línea de comandos de MySQL para leer, filtrar y reaplicar el contenido de los archivos binary log."
translationKey: "glossary_mysqlbinlog"
articles:
  - "/posts/mysql/binary-log-mysql"
---

**mysqlbinlog** es la utilidad de línea de comandos proporcionada con MySQL para leer y decodificar el contenido de los archivos binary log. Es la única herramienta capaz de convertir el formato binario de los binlog en salida legible o en instrucciones SQL re-ejecutables.

## Cómo funciona

mysqlbinlog lee los archivos binlog y produce salida en formato texto. Soporta varios filtros:

- **Por intervalo temporal**: `--start-datetime` y `--stop-datetime` para limitar la salida a una ventana temporal
- **Por base de datos**: `--database` para filtrar los eventos de una base de datos específica
- **Por posición**: `--start-position` y `--stop-position` para seleccionar eventos específicos

Con formato ROW, el flag `--verbose` decodifica los cambios fila por fila en formato pseudo-SQL comentado, de lo contrario la salida es un blob binario ilegible.

## Para qué sirve

mysqlbinlog se utiliza en dos escenarios principales:

- **Point-in-time recovery**: extraer y reaplicar los eventos desde el backup hasta el momento deseado, enviando la salida directamente al cliente mysql
- **Debug de replicación**: analizar los eventos para entender qué se ha replicado, identificar transacciones problemáticas o reconstruir la secuencia de operaciones que causó un problema

## Cuándo se usa

mysqlbinlog es esencial cada vez que se necesita inspeccionar qué pasó en la base de datos después de un incidente, o cuando se debe ejecutar un point-in-time recovery. Requiere acceso a los archivos binlog en el filesystem del servidor o la posibilidad de conectarse al servidor con `--read-from-remote-server`.
