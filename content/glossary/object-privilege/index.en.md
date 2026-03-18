---
title: "Object Privilege"
description: "Oracle privilege that authorizes operations on a specific database object such as SELECT, INSERT, UPDATE or EXECUTE on a table, view or procedure."
translationKey: "glossary_object-privilege"
aka: "Oracle Object Privilege"
articles:
  - "/posts/oracle/oracle-roles-privileges"
---

An **Object Privilege** in Oracle is an authorization that allows performing operations on a specific database object: a table, view, sequence, or PL/SQL procedure. Typical examples include `SELECT ON schema.table`, `INSERT ON schema.table`, and `EXECUTE ON schema.procedure`.

## How it works

Object privileges are granted with `GRANT` specifying the operation and the target object: `GRANT SELECT ON app_owner.customers TO srv_report`. They can be assigned to individual users or roles. Unlike system privileges, they operate on a single object and do not confer global powers over the database.

## What it's for

Object privileges are the primary tool for implementing the principle of least privilege in Oracle. They allow building granular access models: a reporting user gets only SELECT, an application user gets SELECT + INSERT + UPDATE on operational tables, and so on. Combined with custom roles, they create clean and maintainable security architectures.

## Why it matters

The difference between `GRANT SELECT ON app_owner.customers` and `GRANT DBA` is the difference between giving the key to one room and giving the keys to the entire building. In environments with hundreds of tables, object privileges are typically managed through PL/SQL blocks that automatically generate grants for all tables in a schema.
