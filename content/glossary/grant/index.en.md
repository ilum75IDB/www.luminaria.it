---
title: "GRANT"
description: "SQL command to assign specific privileges to a user or role on databases, tables or columns. In MySQL 8 it no longer creates users implicitly."
translationKey: "glossary_grant"
articles:
  - "/posts/mysql/mysql-users-and-hosts"
  - "/posts/postgresql/postgresql_roles_and_users"
---

**GRANT** is the SQL command used to assign privileges to a user or role on specific database objects. In MySQL and MariaDB, privileges are assigned to the `'user'@'host'` pair, not just the username.

## How it works

The basic syntax is `GRANT <privileges> ON <database>.<table> TO 'user'@'host'`. Privileges can be granular (SELECT, INSERT, UPDATE, DELETE) or global (ALL PRIVILEGES). In MySQL 8, GRANT no longer creates users implicitly: an explicit `CREATE USER` is needed first, then GRANT. In MySQL 5.7 and MariaDB, GRANT with `IDENTIFIED BY` creates the user and assigns privileges in a single command.

## What it's for

GRANT is the fundamental mechanism for implementing access control in MySQL/MariaDB databases. Combined with the `user@host` model, it allows calibrating privileges based on the connection origin: full access from localhost for the DBA, read-only from the application server.

## When to use it

Every time a user is created or permissions are modified. The best practice is to always assign the minimum necessary privilege (principle of least privilege) and use `SHOW GRANTS FOR 'user'@'host'` to verify effective privileges.
