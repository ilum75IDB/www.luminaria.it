---
title: "MySQL Users: Why 'mario' and 'mario'@'localhost' Are Not the Same Person"
description: "In MySQL and MariaDB a user's identity depends on the host they connect from. A real case, the authentication model explained in depth and the most common mistakes in access management."
date: "2026-03-08T10:00:00+01:00"
draft: false
translationKey: "mysql_users_and_hosts"
tags: ["mysql", "mariadb", "security", "users", "privileges", "authentication"]
categories: ["mysql", "security"]
image: "mysql-users-and-hosts.cover.jpg"
---

A few weeks ago a client calls me. Pragmatic tone, seemingly trivial request:

> "I need to create a user on MySQL for an application that needs to access a database. Can you take care of it?"

Sure. `CREATE USER`, `GRANT`, next.

Then he adds: "The application runs on two different servers. And sometimes we'll also connect locally for maintenance."

Right. This is where it stops being trivial. Because in MySQL, creating "a user" does not mean what you think.

---

## MySQL's Authentication Model: User + Host

The first thing to understand — and that many DBAs coming from Oracle or PostgreSQL learn the hard way — is that in MySQL **a user's identity is not just their name**.

It is the pair `'user'@'host'`.

This means that:

``` sql
'mario'@'localhost'
'mario'@'192.168.1.10'
'mario'@'%'
```

are not the same user. They are **three different users**. With different passwords, different privileges, different behaviors.

When MySQL receives a connection, it looks at two things:
1. The username provided
2. The IP address (or hostname) from which the connection originates

Then it searches the `mysql.user` table for the row that matches the most specific pair. Not the first one found. The most specific one.

---

## Why This Model?

The design choice is not random. MySQL was born in 1995 for the web. Environments where the same database serves applications running on different machines, different networks, with different access requirements.

The `user@host` model allows you to:

- grant full access from localhost (for the DBA)
- grant limited access from a specific application server
- block everything else

No firewall. No VPN. Directly in the authentication engine.

It is a powerful model. But if you don't understand it, it bites.

---

## The Client's Case: How I Solved It

Back to the request. The application runs on two servers (`192.168.1.20` and `192.168.1.21`) and local access for maintenance is also needed.

The temptation is to create a single user with `'%'` (wildcard = any host):

``` sql
CREATE USER 'app_sales'@'%' IDENTIFIED BY 'SecurePassword#2026';
GRANT SELECT, INSERT, UPDATE ON sales_db.* TO 'app_sales'@'%';
```

Does it work? Yes. Is it correct? No.

The problem with `'%'` is that it accepts connections from **any IP**. If someone finds the password tomorrow, they can connect from anywhere in the network. Or the world, if the database is exposed.

The correct solution is to create **specific users for each source**:

``` sql
-- Access from the primary application server
CREATE USER 'app_sales'@'192.168.1.20' IDENTIFIED BY 'SecurePassword#2026';
GRANT SELECT, INSERT, UPDATE ON sales_db.* TO 'app_sales'@'192.168.1.20';

-- Access from the secondary application server
CREATE USER 'app_sales'@'192.168.1.21' IDENTIFIED BY 'SecurePassword#2026';
GRANT SELECT, INSERT, UPDATE ON sales_db.* TO 'app_sales'@'192.168.1.21';

-- Local access for maintenance (different privileges)
CREATE USER 'app_sales'@'localhost' IDENTIFIED BY 'MaintPassword#2026';
GRANT SELECT ON sales_db.* TO 'app_sales'@'localhost';
```

Three users. Same name. Calibrated privileges.

The local user has only `SELECT` because it's for checks, not for writing data. Different password because the usage context is different.

Principle of least privilege. Applied at the right point.

---

## The Matching Trap: Who Wins?

This is where most errors originate.

If both `'mario'@'%'` and `'mario'@'localhost'` exist, and Mario connects from localhost, which user is used?

Answer: **`'mario'@'localhost'`**.

MySQL sorts the rows in the `mysql.user` table from most specific to least specific:

1. Exact literal host (`192.168.1.20`)
2. Pattern with wildcard (`192.168.1.%`)
3. Full wildcard (`%`)

And uses the **first match** in specificity order.

The classic problem is this: you create `'mario'@'%'` with all privileges. Then someone creates `'mario'@'localhost'` without privileges (or with a different password). From that moment, Mario can no longer log in from local and nobody understands why.

I have seen this scenario at least a dozen times in production. The solution is always the same: **check what exists before you create**.

``` sql
SELECT user, host, authentication_string
FROM mysql.user
WHERE user = 'mario';
```

If you don't do it before, you'll do it after. With more urgency and less calm.

---

## MySQL vs MariaDB: The Differences That Matter

The `user@host` model is identical between MySQL and MariaDB. But there are implementation differences worth knowing.

**Default authentication:**

| Version | Default Plugin |
|---|---|
| MySQL 5.7 | `mysql_native_password` |
| MySQL 8.0+ | `caching_sha2_password` |
| MariaDB 10.x | `mysql_native_password` |

If you migrate from MariaDB to MySQL 8 (or vice versa), clients might fail to connect because the authentication plugin is different. It's not a bug. It's a default change.

**User creation:**

In MySQL 8, `GRANT` no longer creates users implicitly. You must do `CREATE USER` first and `GRANT` after. Always.

``` sql
-- MySQL 8: correct
CREATE USER 'app'@'10.0.0.5' IDENTIFIED BY 'pwd123';
GRANT SELECT ON mydb.* TO 'app'@'10.0.0.5';

-- MySQL 5.7 / MariaDB: still works (but deprecated)
GRANT SELECT ON mydb.* TO 'app'@'10.0.0.5' IDENTIFIED BY 'pwd123';
```

If you are writing provisioning scripts, this detail can break an entire CI/CD pipeline.

**Roles:**

MySQL 8.0 introduced roles. MariaDB supports them since 10.0.5, but with slightly different syntax.

``` sql
-- MySQL 8.0
CREATE ROLE 'role_readonly';
GRANT SELECT ON sales_db.* TO 'role_readonly';
GRANT 'role_readonly' TO 'app_sales'@'192.168.1.20';
SET DEFAULT ROLE 'role_readonly' FOR 'app_sales'@'192.168.1.20';

-- MariaDB 10.x
CREATE ROLE role_readonly;
GRANT SELECT ON sales_db.* TO role_readonly;
GRANT role_readonly TO 'app_sales'@'192.168.1.20';
SET DEFAULT ROLE role_readonly FOR 'app_sales'@'192.168.1.20';
```

The difference looks cosmetic (quotes or not), but in automated scripts it can generate syntax errors.

---

## The Anonymous User: The Ghost Nobody Invited

MySQL ships with an anonymous user: `''@'localhost'`. No name, no password.

This user is a historical artifact from development installations. In production it is a pure security risk.

The anonymous user wins over `'mario'@'%'` when the connection comes from localhost, because `'localhost'` is more specific than `'%'`.

Result: Mario connects locally, MySQL authenticates him as the anonymous user, and Mario's privileges vanish.

The first thing to do on any MySQL/MariaDB production installation:

``` sql
SELECT user, host FROM mysql.user WHERE user = '';

-- If found:
DROP USER ''@'localhost';
DROP USER ''@'%';  -- if it exists
FLUSH PRIVILEGES;
```

It's not paranoia. It's hygiene.

---

## Operational Checklist

After the client experience, I formalized a checklist that I use every time I need to create users on MySQL or MariaDB:

1. **Check existing users** with the same name on different hosts
2. **Remove anonymous users** if present
3. **Create users with specific hosts**, never with `'%'` in production unless strictly necessary
4. **Grant only the necessary privileges** — `SELECT` if `SELECT` is enough
5. **Use separate `CREATE USER` + `GRANT`** (mandatory on MySQL 8)
6. **Check the authentication plugin** if clients have connection issues
7. **Document the user/host pairs** — in six months nobody will remember why three "app_sales" exist

---

## Conclusion

In MySQL and MariaDB a user is not a name. It is a name bound to a point of origin.

This model is powerful because it allows you to segment access without additional infrastructure. But it is also a source of subtle errors if you don't understand it thoroughly.

The next time someone asks you to "create a user on MySQL", before writing the first `CREATE USER`, ask yourself: **where will they connect from?**

The answer to that question changes everything.
