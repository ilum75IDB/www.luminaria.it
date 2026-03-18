---
title: "Anonymous User"
description: "MySQL/MariaDB user with no name automatically created during installation. Represents a security risk as it can interfere with legitimate user matching."
translationKey: "glossary_anonymous-user"
articles:
  - "/posts/mysql/mysql-users-and-hosts"
---

The **Anonymous User** is a MySQL/MariaDB account with an empty username (`''@'localhost'`) that is automatically created during installation. It has no name and often no password.

## How it works

When a user connects, MySQL looks for the most specific match in the `mysql.user` table. The anonymous user `''@'localhost'` is more specific than `'mario'@'%'` for a connection from localhost, because `'localhost'` beats `'%'` in the specificity hierarchy. Consequently, Mario connecting locally gets authenticated as the anonymous user and loses all his privileges.

## What it's for

The anonymous user was intended for development installations where connections without credentials were desired. In production it serves no purpose and represents a security risk: it can capture connections intended for other users and grant unauthorised access.

## When to use it

Never in production. The first operation on any production MySQL/MariaDB installation is to check for and remove anonymous users with `SELECT user, host FROM mysql.user WHERE user = ''` followed by `DROP USER ''@'localhost'`.
