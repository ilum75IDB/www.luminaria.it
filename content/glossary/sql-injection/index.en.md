---
title: "SQL Injection"
description: "Attack technique that inserts malicious SQL code into application inputs to manipulate queries executed by the database, potentially accessing unauthorised data or compromising the system."
translationKey: "glossary_sql-injection"
articles:
  - "/posts/mysql/mysql-multi-istanza-secure-file-priv"
---

**SQL Injection** is one of the most widespread and dangerous vulnerabilities in web applications. It occurs when user-supplied input is inserted directly into SQL queries without validation or parameterisation, allowing an attacker to modify the query logic.

## How it works

The attacker inserts SQL code fragments into application input fields (login forms, search fields, URL parameters). If the application concatenates these inputs directly into SQL queries, the malicious code is executed by the database with the application user's privileges. Combined with MySQL's `FILE` privilege and an unconfigured `secure-file-priv`, the attacker can read system files or write arbitrary files on the server.

## What it's for

Understanding SQL injection is fundamental for anyone managing databases in production, because many security configurations (such as `secure-file-priv`, privilege management and user separation) exist specifically to mitigate the impact of this type of attack.

## When to use it

The term describes an attack to prevent, not a technique to use. The main countermeasures are: parameterised queries (prepared statements), input validation, principle of least privilege for database users, and correct configuration of directives like `secure-file-priv`.
