---
title: "FLUSH PRIVILEGES"
description: "MySQL/MariaDB command that reloads grant tables from mysql.user, making manual privilege changes effective."
translationKey: "glossary_flush-privileges"
articles:
  - "/posts/mysql/mysql-users-and-hosts"
---

**FLUSH PRIVILEGES** is a MySQL/MariaDB command that forces the server to reload the privilege tables from the `mysql` database into memory. It makes permission changes immediately effective.

## How it works

MySQL keeps an in-memory cache of the grant tables (`mysql.user`, `mysql.db`, `mysql.tables_priv`). When using `CREATE USER` and `GRANT`, MySQL updates both the tables and the cache automatically. But if grant tables are modified directly with `INSERT`, `UPDATE` or `DELETE`, the cache is not updated. `FLUSH PRIVILEGES` forces a cache reload from the tables.

## What it's for

The command is needed after: direct deletion of users from the `mysql.user` table, manual privilege changes via DML, or after a `DROP USER` of anonymous users as part of security hardening. Without the FLUSH, changes don't take effect until the next server restart.

## When to use it

After any direct modification to the grant tables. If exclusively using `CREATE USER`, `GRANT`, `REVOKE` and `DROP USER`, FLUSH is not technically necessary because these commands update the cache automatically. However, running it after a `DROP USER` of anonymous users is good practice to ensure consistency.
