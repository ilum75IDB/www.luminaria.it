---
title: "Unix Socket"
description: "Local inter-process communication mechanism on Unix/Linux systems, used by MySQL for faster connections than TCP when client and server are on the same host."
translationKey: "glossary_unix-socket"
articles:
  - "/posts/mysql/mysql-multi-istanza-secure-file-priv"
---

A **Unix Socket** (or Unix domain socket) is a communication endpoint that allows two processes on the same operating system to exchange data without going through the TCP/IP network stack. In MySQL, it is the default connection method when connecting to `localhost`.

## How it works

When a MySQL client connects specifying `-h localhost`, the client does not use TCP. It uses the Unix socket file (typically `/var/run/mysqld/mysqld.sock`) to communicate directly with the MySQL server process. This communication happens entirely within the kernel, with no network overhead, and is faster than a TCP connection even on the same host.

## What it's for

In multi-instance environments, each MySQL instance has its own socket file (e.g. `mysqld.sock`, `mysqld-app2.sock`). Specifying the correct socket with `--socket=/path/to/socket` is the only reliable way to connect to the intended instance. Without specifying the socket, the client uses the default one — which almost always points to the primary instance.

## When to use it

Unix sockets are used for all local connections to MySQL. In environments with multiple instances, it is essential to explicitly specify the socket for each connection. For remote connections (from another host), TCP with `-h <ip> -P <port>` is used instead.
