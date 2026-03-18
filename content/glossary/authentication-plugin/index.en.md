---
title: "Authentication Plugin"
description: "MySQL/MariaDB module that handles the credential verification method during connection. The default changes between versions and can cause compatibility issues."
translationKey: "glossary_authentication-plugin"
articles:
  - "/posts/mysql/mysql-users-and-hosts"
---

An **Authentication Plugin** is the module that MySQL or MariaDB uses to verify a user's credentials at connection time. Every user in the system is associated with a specific plugin that determines how the password is hashed, transmitted and verified.

## How it works

The main plugins are: `mysql_native_password` (default in MySQL 5.7 and MariaDB), which uses a double SHA1 hash; `caching_sha2_password` (default in MySQL 8.0+), which uses SHA-256 with caching to improve security and performance. When a client connects, it must support the plugin of the user it's trying to authenticate to.

## What it's for

Knowledge of authentication plugins is essential during migrations between versions or between MySQL and MariaDB. A client that only supports `mysql_native_password` cannot connect to a user with `caching_sha2_password` — and the resulting error is often cryptic and hard to diagnose.

## When to use it

The plugin is specified at user creation time (`CREATE USER ... IDENTIFIED WITH <plugin> BY 'password'`) or can be checked and changed with `ALTER USER`. When writing provisioning scripts that must work across different MySQL/MariaDB versions, it's important to explicitly specify the plugin.
