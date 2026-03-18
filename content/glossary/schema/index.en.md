---
title: "Schema"
description: "Logical namespace within a database that groups tables, views, functions and other objects, enabling organization and permission separation."
translationKey: "glossary_schema"
aka: "Database Schema"
articles:
  - "/posts/postgresql/postgresql_roles_and_users"
---

A **Schema** in a relational database is a logical namespace that groups objects such as tables, views, functions, and sequences. It functions as an organizational container within a database.

## How it works

In PostgreSQL, the default schema is `public`. To access an object in another schema, the prefix is required: `schema1.table`. The `USAGE` privilege on a schema is a prerequisite for accessing any object within it — without `USAGE`, even a `GRANT SELECT` on tables does not work.

## What it's for

Schemas allow logical data separation: one schema for the application, one for reporting, one for staging tables. In Oracle, the concept is different: each user is automatically a schema, and objects created by that user live in their schema. In PostgreSQL, schemas and users are independent entities.

## Why it matters

Schema permission management is the most common source of errors when creating users with limited access. Forgetting `GRANT USAGE ON SCHEMA` is the classic mistake that generates "permission denied for schema" even when table permissions are correct.
