---
title: "Least Privilege"
description: "Security principle that prescribes assigning to each user or process only the permissions strictly necessary to perform their function."
translationKey: "glossary_least-privilege"
aka: "Principle of Least Privilege"
articles:
  - "/posts/mysql/mysql-users-and-hosts"
  - "/posts/oracle/oracle-roles-privileges"
  - "/posts/postgresql/postgresql_roles_and_users"
---

**Least Privilege** is a fundamental information security principle: every user, process or system should have only the permissions strictly necessary to perform their function, nothing more.

## How it works

In the database context, the principle is applied by assigning granular privileges: `SELECT` if the user only needs to read, `SELECT + INSERT + UPDATE` if they also need to write, never `ALL PRIVILEGES` unless strictly necessary. Combined with MySQL's `user@host` model, the principle can also be applied based on the connection origin.

## What it's for

Limiting privileges reduces the attack surface. If an application is compromised, the attacker inherits the privileges of the application's database user. If that user has only SELECT on a specific database, the damage is contained. If it has ALL PRIVILEGES, the entire server is at risk.

## When to use it

Always. The principle of least privilege applies in every context: database users, operating system users, application roles, service accounts. The temptation to assign broad privileges "to avoid problems" is the most common cause of avoidable security incidents.
