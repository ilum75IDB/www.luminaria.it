---
title: "systemd"
description: "Linux init system and service manager, used to manage multiple MySQL/MariaDB instances on the same server through separate unit files."
translationKey: "glossary_systemd"
articles:
  - "/posts/mysql/mysql-multi-istanza-secure-file-priv"
---

**systemd** is the default init system and service manager on modern Linux distributions (CentOS/RHEL 7+, Ubuntu 16.04+, Debian 8+). In the database context, it is the mechanism that starts, stops and monitors MySQL or MariaDB instances.

## How it works

Each service is defined by a unit file (e.g. `mysqld.service`) that specifies the startup command, configuration file, dependencies and crash behaviour. In a multi-instance setup, separate unit files are created for each instance (e.g. `mysqld-app2.service`, `mysqld-reporting.service`), each with its own `--defaults-file` pointing to a different `my.cnf`.

## What it's for

systemd allows managing MySQL instances as independent services: starting, stopping, restarting and monitoring them separately. The `systemctl cat <service>` command is essential for tracing from the service name back to the instance's configuration file, and from there to port, socket and datadir.

## When to use it

systemd is automatically active on any modern Linux server. In DBA work, you interact with it via `systemctl start/stop/status/restart <service>`. In multi-instance environments, `systemctl list-units --type=service | grep mysql` is the first command for identifying how many instances are running on a server.
