---
title: "Users, Roles and Privileges in Oracle: Why GRANT ALL Is Never the Answer"
description: "A client where every application user connected as the schema owner with the DBA role. How I restructured the Oracle security model by applying the principle of least privilege — with real SQL, custom roles and Unified Audit."
date: "2026-01-27T10:00:00+01:00"
draft: false
translationKey: "oracle_roles_privileges"
tags: ["security", "roles", "privileges", "grant", "revoke", "audit"]
categories: ["oracle"]
image: "oracle-roles-privileges.cover.jpg"
---

It has happened to me more than once: I walk into an Oracle environment and find the same situation. Every application user connected as the schema owner, with the DBA role granted. Developers, batch jobs, reporting tools — all running with the same privileges as the user that owns the tables.

When you ask why, the answer is always some variation of: "This way everything works without permission issues."

Sure. Everything works. Until the day a developer runs a `DROP TABLE` on the wrong table. Or a batch import does a `TRUNCATE` on a production table thinking it is in the test environment. Or someone runs a `DELETE FROM customers` without a `WHERE` clause.

That day the problem is no longer about permissions. It is that you have no idea who did what, and no tool to prevent it from happening again.

---

## The context: a pattern that keeps repeating

The client was a mid-sized company with an ERP application running on Oracle 19c. About twenty users — a mix of developers, application accounts and operators. The application schema — let us call it `APP_OWNER` — held roughly 300 tables, about sixty views and a few dozen PL/SQL procedures.

The problem was easy to describe:

- Everyone connected as `APP_OWNER`
- `APP_OWNER` had the `DBA` role
- No audit configured
- No separation between readers and writers
- Passwords were shared via email

It was not negligence. It was inertia. The system had grown that way over the years, and nobody had ever stopped to rethink the model. It worked, and that was enough.

Until an operator accidentally deleted an entire quarter's invoicing data. No log, no trail, no identifiable culprit. Only a two-day-old backup and a data gap that took weeks to fill.

---

## How Oracle security works: the model

Before describing what I did, it helps to understand how Oracle structures security. The model is different from PostgreSQL and MySQL, and the differences are not cosmetic.

### User and schema: the same thing (almost)

In Oracle, **creating a user means creating a schema**. They are not separate concepts: the user `APP_OWNER` is also the schema `APP_OWNER`, and the objects created by that user live in that schema.

``` sql
CREATE USER app_read IDENTIFIED BY "PasswordSecure#2026"
DEFAULT TABLESPACE users
TEMPORARY TABLESPACE temp
QUOTA 0 ON users;
```

The `QUOTA 0` is intentional: this user is not supposed to create objects. It is a consumer, not an owner.

### System privileges vs object privileges

Oracle draws a clear line between:

- **System privileges**: global operations like `CREATE TABLE`, `CREATE SESSION`, `ALTER SYSTEM`
- **Object privileges**: operations on specific objects like `SELECT ON app_owner.customers`, `EXECUTE ON app_owner.pkg_invoices`

The `DBA` role includes over 200 system privileges. Granting it to an application user is like handing the keys to the entire building to someone who only needs to enter one room.

### Roles: predefined and custom

Oracle offers predefined roles (`CONNECT`, `RESOURCE`, `DBA`) and allows custom ones. The predefined roles carry a historical problem: `CONNECT` and `RESOURCE` used to include excessive privileges in older versions. From Oracle 12c onward they were trimmed, but the habit of granting them without a second thought dies hard.

The right path is building custom roles calibrated to actual needs.

---

## The implementation: three roles, zero ambiguity

I designed three roles: read, write and application administration.

### 1. Read-only role

``` sql
CREATE ROLE app_read_role;

-- Table privileges
GRANT SELECT ON app_owner.customers     TO app_read_role;
GRANT SELECT ON app_owner.orders        TO app_read_role;
GRANT SELECT ON app_owner.invoices      TO app_read_role;
GRANT SELECT ON app_owner.products      TO app_read_role;
GRANT SELECT ON app_owner.transactions  TO app_read_role;

-- View privileges
GRANT SELECT ON app_owner.v_sales_report    TO app_read_role;
GRANT SELECT ON app_owner.v_order_status    TO app_read_role;
```

In an environment with 300 tables you do not list them one by one manually. I used a PL/SQL block to generate the grants:

``` sql
BEGIN
  FOR t IN (SELECT table_name FROM dba_tables
            WHERE owner = 'APP_OWNER') LOOP
    EXECUTE IMMEDIATE 'GRANT SELECT ON app_owner.'
      || t.table_name || ' TO app_read_role';
  END LOOP;
END;
/
```

Simple, repeatable, and above all: documented. Because six months from now someone will need to understand what was done and why.

### 2. Read-write role

``` sql
CREATE ROLE app_write_role;

-- Inherits everything from the read role
GRANT app_read_role TO app_write_role;

-- Adds DML on operational tables
GRANT INSERT, UPDATE, DELETE ON app_owner.orders        TO app_write_role;
GRANT INSERT, UPDATE, DELETE ON app_owner.transactions  TO app_write_role;
GRANT INSERT, UPDATE ON app_owner.customers             TO app_write_role;

-- Execute permission on application procedures
GRANT EXECUTE ON app_owner.pkg_orders   TO app_write_role;
GRANT EXECUTE ON app_owner.pkg_invoices TO app_write_role;
```

Note: no `DELETE` on the `customers` table. Not because it is technically impossible, but because the application process calls for deactivation, not deletion. The privilege reflects the process, not convenience.

### 3. Application administration role

``` sql
CREATE ROLE app_admin_role;

-- Inherits the write role
GRANT app_write_role TO app_admin_role;

-- Adds controlled DDL
GRANT CREATE VIEW TO app_admin_role;
GRANT CREATE PROCEDURE TO app_admin_role;
GRANT CREATE SYNONYM TO app_admin_role;

-- Can manage configuration tables
GRANT INSERT, UPDATE, DELETE ON app_owner.parameters    TO app_admin_role;
GRANT INSERT, UPDATE, DELETE ON app_owner.lookup_types  TO app_admin_role;
```

No `CREATE TABLE`, no `DROP ANY`, no `ALTER SYSTEM`. The application admin manages logic, not physical structure.

---

## User creation and role assignment

``` sql
-- Reporting user (read-only)
CREATE USER srv_report IDENTIFIED BY "RptSecure#2026"
DEFAULT TABLESPACE users TEMPORARY TABLESPACE temp
QUOTA 0 ON users;
GRANT CREATE SESSION TO srv_report;
GRANT app_read_role TO srv_report;

-- Application user (read-write)
CREATE USER srv_app IDENTIFIED BY "AppSecure#2026"
DEFAULT TABLESPACE users TEMPORARY TABLESPACE temp
QUOTA 0 ON users;
GRANT CREATE SESSION TO srv_app;
GRANT app_write_role TO srv_app;

-- Application DBA (administration)
CREATE USER dba_app IDENTIFIED BY "DbaSecure#2026"
DEFAULT TABLESPACE users TEMPORARY TABLESPACE temp
QUOTA 10M ON users;
GRANT CREATE SESSION TO dba_app;
GRANT app_admin_role TO dba_app;
```

Each user has its own password, a specific role and a disk quota consistent with its purpose. `srv_report` has no quota because it should not create anything. `dba_app` gets 10 MB because it needs to create views and procedures.

---

## Revoking the DBA role

The most delicate step: removing `DBA` from `APP_OWNER`.

``` sql
REVOKE DBA FROM app_owner;
```

One line. But before running it, I verified that `APP_OWNER` still had the privileges needed to own its objects:

``` sql
SELECT privilege FROM dba_sys_privs WHERE grantee = 'APP_OWNER';
SELECT granted_role FROM dba_role_privs WHERE grantee = 'APP_OWNER';
```

Then I granted only what was strictly necessary:

``` sql
GRANT CREATE SESSION TO app_owner;
GRANT CREATE TABLE TO app_owner;
GRANT CREATE VIEW TO app_owner;
GRANT CREATE PROCEDURE TO app_owner;
GRANT CREATE SEQUENCE TO app_owner;
GRANT UNLIMITED TABLESPACE TO app_owner;
```

`APP_OWNER` remains the owner of the objects but no longer has the power to do anything on the database. It is an owner, not a god.

---

## Audit: knowing who did what

Having the right roles is not enough. You need to know who did what, especially for critical operations.

Since version 12c, Oracle offers **Unified Audit**, replacing the old traditional audit with a centralized system.

``` sql
-- Audit critical DDL operations
CREATE AUDIT POLICY pol_critical_ddl
ACTIONS CREATE TABLE, DROP TABLE, ALTER TABLE,
        TRUNCATE TABLE, CREATE USER, DROP USER,
        ALTER USER, GRANT, REVOKE;

ALTER AUDIT POLICY pol_critical_ddl ENABLE;

-- Audit sensitive data access
CREATE AUDIT POLICY pol_data_access
ACTIONS SELECT ON app_owner.customers,
        DELETE ON app_owner.invoices,
        UPDATE ON app_owner.invoices;

ALTER AUDIT POLICY pol_data_access ENABLE;

-- Audit failed logins
CREATE AUDIT POLICY pol_failed_logins
ACTIONS LOGON;
ALTER AUDIT POLICY pol_failed_logins
ENABLE WHENEVER NOT SUCCESSFUL;
```

To check what is being recorded:

``` sql
SELECT * FROM unified_audit_trail
WHERE event_timestamp > SYSDATE - 7
ORDER BY event_timestamp DESC;
```

Audit is not paranoia. It is the only way to answer the question "who did what?" without relying on guesswork.

---

## Comparison with PostgreSQL and MySQL

This article is the third in a series on security management in relational databases. The first two cover [PostgreSQL](/en/posts/postgresql/security/postgresql_roles_and_users/) and [MySQL](/en/posts/mysql/security/mysql-users-and-hosts/).

The differences among the three systems are substantial:

| Aspect | Oracle | PostgreSQL | MySQL |
|---|---|---|---|
| User = schema? | Yes | No (independent) | Yes (separate databases) |
| Role model | Predefined + custom | Everything is a ROLE | Roles from MySQL 8.0 |
| Identity | Username | Username | user@host pair |
| Native audit | Unified Audit (12c+) | pgAudit (extension) | Audit plugin |
| Granular privileges | System + Object | Database/Schema/Object | Global/DB/Table/Column |
| GRANT ALL | Exists but dangerous | Exists, discouraged | Exists, discouraged |

In PostgreSQL everything is a ROLE, and the simplicity of the model is its strength. In MySQL identity is tied to the originating host, adding a layer of complexity (and security) that the others lack. In Oracle the model is the richest and the most granular, but also the easiest to misconfigure because of the sheer number of options.

The principle remains the same everywhere: **give each user only what they need, not one privilege more**.

---

## What changed afterwards

The transition was gradual — two weeks for the full rollout, with testing on every application and procedure. A few scripts stopped working because they took for granted privileges they were never entitled to. Every error was actually a hidden problem that had been invisible before.

The result:

- **20 named users** instead of a single shared schema
- **3 custom roles** instead of the DBA role
- **Active audit** on DDL and sensitive operations
- **Zero incidents** of accidental deletion in the following months

The client did not notice performance improvements. That was not the goal. What they noticed was that when someone made a mistake, the damage was contained and traceable. And in a production environment, that is worth more than any optimization.

---

## Conclusion

`GRANT ALL PRIVILEGES` and the `DBA` role are shortcuts. They work in the sense that they eliminate permission errors. But they also eliminate every layer of protection.

Security in Oracle is not a tooling problem — the tools are there, and they are powerful. It is a design problem: deciding who can do what, documenting it, implementing it and then verifying that it works.

It is not the most glamorous work in the world. But it is the work that makes the difference between a database that merely survives and one that is truly under control.
