---
title: "Roles and Users in PostgreSQL: Why Everything Is (Only) a ROLE"
description: "PostgreSQL does not distinguish between users and roles: everything is a ROLE. The correct mental model, a real case, and a complete example to build a truly maintainable read-only account."
date: "2026-02-10T10:00:00+01:00"
lastmod: "2026-02-26T09:34:00+01:00"
draft: false
translationKey: "postgresql_roles_and_users"
tags: ["security", "roles", "privileges", "grant", "revoke"]
categories: ["postgresql"]
image: "postgresql_roles_and_users.cover.jpg"
---

The first time I seriously worked with PostgreSQL I was coming from
years of other databases. I looked for the `CREATE USER` command. I found it.
Then I saw `CREATE ROLE`. Then `ALTER USER`. Then `ALTER ROLE`.\
For a few minutes I thought: "Alright, someone here enjoys confusing
people."

Actually, no. PostgreSQL is far more consistent than it appears.
It is just consistent in its own way.

## In PostgreSQL there are no users. There are roles.

The key is this: **in PostgreSQL everything is a ROLE**.

A ROLE can:

-   have the right to login\
-   not have the right to login\
-   own objects\
-   inherit privileges from other roles\
-   be used as a container of privileges

What in other databases you call a "user" in PostgreSQL is simply a role with the `LOGIN` attribute.

In fact:

``` sql
CREATE USER mario;
```

is nothing more than a shortcut for:

``` sql
CREATE ROLE mario WITH LOGIN;
```

Same for `ALTER USER`: it is only an alias of `ALTER ROLE`.

Why does only `CREATE ROLE` and `ALTER ROLE` really exist?\
Because PostgreSQL does not conceptually distinguish between user and role.
It is the same object with different attributes. Minimalist. Elegant. Consistent.

If a role has `LOGIN`, it behaves like a user.\
If it does not have `LOGIN`, it is a container of privileges.

When you truly understand this, the way you design security changes.

------------------------------------------------------------------------

## The correct mental model

Today I reason like this:

-   I create "functional" roles that represent sets of privileges\
-   I assign those roles to real users\
-   I avoid granting permissions directly to users

Why? Because users change. Roles do not.

If tomorrow a new colleague joins, I do not rewrite half the database grants.\
I assign the correct role and that is it.

Clean architecture. No magic. No chaos.

------------------------------------------------------------------------

## A real story (without embarrassing names)

Some time ago I was asked to create a read-only user for a monitoring system.\
Seemingly simple request: "It must read some tables. No writing."

The classic "it’s just read-only".

The trap is always the same: if you only run a `GRANT SELECT` on
existing tables, it works today.\
Three months later someone creates a new table and monitoring starts
throwing errors.\
And guess who gets called.

The correct solution requires attention at four levels:

1.  Permission to connect to the database\
2.  Permission to use the schema (`USAGE`)\
3.  `SELECT` permissions on existing tables and sequences\
4.  Default privileges for future objects

If you skip a piece, sooner or later you pay the price.

------------------------------------------------------------------------

## Example: creating a proper read-only user

Suppose we want to create a read-only user on two schemas.

First I create the role with login:

``` sql
CREATE ROLE srv_monitoring 
WITH LOGIN 
PASSWORD 'SecurePassword123#';
```

I lock it down:

``` sql
ALTER ROLE srv_monitoring 
NOSUPERUSER NOCREATEDB NOCREATEROLE NOINHERIT;
```

Allow connection to the database:

``` sql
GRANT CONNECT ON DATABASE mydb TO srv_monitoring;
```

Allow usage on schemas:

``` sql
GRANT USAGE ON SCHEMA schema1 TO srv_monitoring;
GRANT USAGE ON SCHEMA schema2 TO srv_monitoring;
```

Grant read permissions on existing objects:

``` sql
GRANT SELECT ON ALL TABLES IN SCHEMA schema1 TO srv_monitoring;
GRANT SELECT ON ALL TABLES IN SCHEMA schema2 TO srv_monitoring;

GRANT SELECT ON ALL SEQUENCES IN SCHEMA schema1 TO srv_monitoring;
GRANT SELECT ON ALL SEQUENCES IN SCHEMA schema2 TO srv_monitoring;
```

And now the part many people forget:

``` sql
ALTER DEFAULT PRIVILEGES IN SCHEMA schema1
GRANT SELECT ON TABLES TO srv_monitoring;

ALTER DEFAULT PRIVILEGES IN SCHEMA schema2
GRANT SELECT ON TABLES TO srv_monitoring;
```

This way future tables will also be readable.

Important note: `ALTER DEFAULT PRIVILEGES` applies to the role that
creates the objects. If multiple owners create tables in the same
schemas, the configuration must be replicated for each of them.

------------------------------------------------------------------------

## Why this model is powerful

The fact that everything is a ROLE allows you to build clean hierarchies.

Advanced example:

``` sql
CREATE ROLE role_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA schema1 TO role_readonly;

CREATE ROLE srv_monitoring WITH LOGIN PASSWORD '...';
GRANT role_readonly TO srv_monitoring;
```

Now I can assign `role_readonly` to ten different users without
duplicating grants.

This is design. Not just syntax.

------------------------------------------------------------------------

## Conclusion

PostgreSQL does not complicate the concept of user. It simplifies it.\
There is only one type of object: the ROLE. It is up to us to use it well.

If you treat it as just a "user with a password", it works.\
If you use it as an architectural building block, it becomes a powerful
tool to design clean, scalable and maintainable security.

The difference is not in the commands.\
It is in the mental model you use when applying them.

------------------------------------------------------------------------

## Glossary

**[ROLE](/en/glossary/postgresql-role/)** — PostgreSQL's fundamental entity that unifies the concept of user and permission group: a ROLE with LOGIN is a user, without LOGIN it is a privilege container.

**[DEFAULT PRIVILEGES](/en/glossary/default-privileges/)** — PostgreSQL mechanism that automatically defines privileges to assign to all future objects created in a schema, avoiding the need to repeat GRANTs manually.

**[Schema](/en/glossary/schema/)** — Logical namespace within a database that groups tables, views, functions and other objects, enabling organization and permission separation.

**[GRANT](/en/glossary/grant/)** — SQL command to assign specific privileges to a user or role on databases, tables, or columns.

**[Least Privilege](/en/glossary/least-privilege/)** — Security principle that prescribes assigning to each user only the permissions strictly necessary to perform their function.
