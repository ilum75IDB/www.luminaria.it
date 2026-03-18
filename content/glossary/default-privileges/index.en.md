---
title: "DEFAULT PRIVILEGES"
description: "PostgreSQL mechanism that automatically defines privileges to assign to all future objects created in a schema, avoiding the need to repeat GRANTs manually."
translationKey: "glossary_default-privileges"
aka: "ALTER DEFAULT PRIVILEGES"
articles:
  - "/posts/postgresql/postgresql_roles_and_users"
---

**DEFAULT PRIVILEGES** is a PostgreSQL mechanism that allows defining in advance the privileges that will be automatically assigned to all future objects created in a schema. It is configured with the `ALTER DEFAULT PRIVILEGES` command.

## How it works

The command `ALTER DEFAULT PRIVILEGES IN SCHEMA schema1 GRANT SELECT ON TABLES TO srv_monitoring` ensures that every new table created in `schema1` is automatically readable by `srv_monitoring`. Without this configuration, future tables would require a manual GRANT each time.

## What it's for

It is the part that most administrators forget when creating read-only users. GRANTs on `ALL TABLES IN SCHEMA` cover only existing tables. Tables created afterwards require new GRANTs — unless DEFAULT PRIVILEGES are used. Without them, the monitoring user stops working at the first new table.

## What can go wrong

DEFAULT PRIVILEGES apply to the ROLE that creates the objects. If multiple users create tables in a schema, default privileges must be configured for each creator. This detail often causes hard-to-diagnose errors: "the GRANT is there, but the new table isn't readable."
