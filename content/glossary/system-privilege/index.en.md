---
title: "System Privilege"
description: "Oracle privilege that authorizes global database operations such as CREATE TABLE, CREATE SESSION or ALTER SYSTEM, independent of any specific object."
translationKey: "glossary_system-privilege"
aka: "Oracle System Privilege"
articles:
  - "/posts/oracle/oracle-roles-privileges"
---

A **System Privilege** in Oracle is an authorization that allows performing global database operations, independent of any specific object. Typical examples include `CREATE TABLE`, `CREATE SESSION`, `ALTER SYSTEM`, `CREATE USER`, and `DROP ANY TABLE`.

## How it works

System privileges are granted with `GRANT` and revoked with `REVOKE`. They can be assigned directly to a user or to a role. The predefined `DBA` role includes over 200 system privileges, which is why granting it to application users is a dangerous practice.

## What it's for

System privileges define what a user can do at the database level: create objects, manage users, modify system parameters. They are the highest level of authorization in Oracle and must be managed with extreme care, following the principle of least privilege.

## What can go wrong

A system privilege like `DROP ANY TABLE` allows deleting any table in any schema. If mistakenly granted to an application user, a single command can destroy production data. The distinction between system privileges and object privileges is fundamental to building a robust security model.
