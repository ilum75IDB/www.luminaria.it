---
title: "REVOKE"
description: "SQL command to remove privileges or roles previously granted to a user or role, complementary to the GRANT command."
translationKey: "glossary_revoke"
aka: "SQL Privilege Revocation"
articles:
  - "/posts/oracle/oracle-roles-privileges"
---

**REVOKE** is the SQL command that removes privileges or roles previously assigned with `GRANT`. It is the indispensable complement to GRANT and the primary tool for restricting permissions when a security model is restructured.

## How it works

The syntax follows the same pattern as GRANT: `REVOKE SELECT ON schema.table FROM user` or `REVOKE role FROM user`. In Oracle, revoking a role like `DBA` removes in one stroke all the system privileges included in that role. Before executing a critical REVOKE, it is essential to verify that the user retains the privileges necessary for their functions.

## When to use it

The most common case is security model restructuring: removing excessive roles (like DBA from application users) and replacing them with calibrated custom roles. It is also used when a user changes function, when a service is decommissioned, or when an audit reveals privileges granted in excess.

## What can go wrong

A poorly planned REVOKE can break production applications. If an application connects with a user that loses the `CREATE SESSION` privilege, it stops working instantly. This is why revoking critical privileges should always be preceded by a dependency analysis and a gradual rollout plan.
