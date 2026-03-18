---
title: "INTO OUTFILE"
description: "Cláusula SQL de MySQL que permite escribir el resultado de un SELECT directamente en un archivo en el filesystem del servidor."
translationKey: "glossary_into-outfile"
articles:
  - "/posts/mysql/mysql-multi-istanza-secure-file-priv"
---

**INTO OUTFILE** es una cláusula SQL de MySQL que permite exportar el resultado de una query directamente a un archivo en el filesystem del servidor de base de datos. Es el método nativo para generar archivos CSV, TSV o con separadores personalizados.

## Cómo funciona

La cláusula se añade al final de un `SELECT` y especifica la ruta del archivo de destino. Los parámetros `FIELDS TERMINATED BY`, `ENCLOSED BY` y `LINES TERMINATED BY` controlan el formato de la salida. El archivo es creado por el usuario de sistema MySQL (no por el usuario que ejecuta la query), por lo que debe estar en un directorio con los permisos correctos.

## Para qué sirve

`INTO OUTFILE` es útil para exports masivos de datos de la base de datos a archivos de texto estructurados. Es el complemento de `LOAD DATA INFILE`, que hace la operación inversa (importa datos desde archivos). Juntos forman el mecanismo nativo de MySQL para import/export masivo.

## Cuándo se usa

Su uso está vinculado a la directiva `secure-file-priv`: el archivo de destino debe estar dentro del directorio autorizado. Cuando `secure-file-priv` bloquea la ruta deseada, la alternativa es usar el cliente mysql desde shell con `-B -e` y redirigir la salida, que no está sujeta a las mismas restricciones.
