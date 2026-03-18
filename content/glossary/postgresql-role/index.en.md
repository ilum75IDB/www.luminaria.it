---
title: "ROLE"
description: "PostgreSQL's fundamental entity that unifies the concept of user and permission group: a ROLE with LOGIN is a user, without LOGIN it is a privilege container."
translationKey: "glossary_postgresql-role"
aka: "PostgreSQL Role"
articles:
  - "/posts/postgresql/postgresql_roles_and_users"
---

In PostgreSQL, **ROLE** is the only security entity. There is no distinction between "user" and "role": everything is a ROLE. A ROLE with the `LOGIN` attribute behaves as a user; without `LOGIN`, it is a privilege container assignable to other ROLEs.

## How it works

`CREATE USER mario` is simply a shortcut for `CREATE ROLE mario WITH LOGIN`. ROLEs can own objects, inherit privileges from other ROLEs through the `INHERIT` attribute, and be used to build permission hierarchies. A "functional" ROLE (without LOGIN) groups privileges; "user" ROLEs (with LOGIN) inherit them.

## What it's for

The unified model enables designing clean security architectures: create functional ROLEs like `role_readonly` or `role_write`, assign privileges to the functional ROLEs, then assign those ROLEs to real users. When a new colleague arrives, a single `GRANT role_readonly TO new_user` is all it takes.

## Why it matters

The model's simplicity is its strength — but also a trap if misunderstood. Many administrators assign privileges directly to users instead of using functional ROLEs, creating a tangle of GRANTs impossible to maintain. The correct mental model is: privileges go to ROLEs, ROLEs go to users.
