---
title: "secure-file-priv"
description: "MySQL security directive that limits the directories where the server can read and write files, protecting the filesystem from unauthorised operations."
translationKey: "glossary_secure-file-priv"
articles:
  - "/posts/mysql/mysql-multi-istanza-secure-file-priv"
---

**secure-file-priv** is a MySQL system variable that controls where `LOAD DATA INFILE`, `SELECT INTO OUTFILE` and the `LOAD_FILE()` function can operate on the server's filesystem.

## How it works

The variable accepts three values: a specific path (e.g. `/var/lib/mysql-files/`), which limits file operations to that directory; an empty string (`""`), which imposes no restrictions; or `NULL`, which completely disables file operations. The value can only be set in the configuration file (`my.cnf`) and requires a service restart to change — it cannot be modified at runtime.

## What it's for

The directive prevents arbitrary filesystem access by MySQL users with the `FILE` privilege. Without this protection, an attacker exploiting SQL injection could read system files (e.g. `/etc/passwd`, SSH keys) or write web shells into the webroot of a web server on the same host.

## When to use it

`secure-file-priv` should be configured at setup time for every MySQL instance, specifying a dedicated directory. In multi-instance environments, each instance should have its own `secure-file-priv` directory. If file export is blocked, the recommended alternative is using the mysql command-line client with `-B` and `-e` options to redirect output.
