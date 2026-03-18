---
title: "Unix Socket"
description: "Mecanismo de comunicación inter-proceso local en sistemas Unix/Linux, usado por MySQL para conexiones más rápidas que TCP cuando cliente y servidor están en el mismo host."
translationKey: "glossary_unix-socket"
articles:
  - "/posts/mysql/mysql-multi-istanza-secure-file-priv"
---

Un **Unix Socket** (o socket de dominio Unix) es un endpoint de comunicación que permite a dos procesos en el mismo sistema operativo intercambiar datos sin pasar por la pila de red TCP/IP. En MySQL, es el método de conexión predeterminado cuando te conectas a `localhost`.

## Cómo funciona

Cuando un cliente MySQL se conecta especificando `-h localhost`, el cliente no usa TCP. Usa el archivo socket Unix (típicamente `/var/run/mysqld/mysqld.sock`) para comunicarse directamente con el proceso del servidor MySQL. Esta comunicación ocurre enteramente en el kernel, sin overhead de red, y es más rápida que una conexión TCP incluso en el mismo host.

## Para qué sirve

En entornos multi-instancia, cada instancia MySQL tiene su propio archivo socket (ej. `mysqld.sock`, `mysqld-app2.sock`). Especificar el socket correcto con `--socket=/path/to/socket` es la única forma fiable de conectarse a la instancia deseada. Sin especificar el socket, el cliente usa el predeterminado — que casi siempre apunta a la instancia primaria.

## Cuándo se usa

Los sockets Unix se usan para todas las conexiones locales a MySQL. En entornos con instancias múltiples, es esencial especificar explícitamente el socket para cada conexión. Para conexiones remotas (desde otro host), se usa TCP con `-h <ip> -P <puerto>`.
