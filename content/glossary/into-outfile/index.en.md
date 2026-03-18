---
title: "INTO OUTFILE"
description: "MySQL SQL clause that allows writing the result of a SELECT directly to a file on the server's filesystem."
translationKey: "glossary_into-outfile"
articles:
  - "/posts/mysql/mysql-multi-istanza-secure-file-priv"
---

**INTO OUTFILE** is a MySQL SQL clause that allows exporting the result of a query directly to a file on the database server's filesystem. It is the native method for generating CSV, TSV or custom-delimited files.

## How it works

The clause is appended to a `SELECT` statement and specifies the destination file path. The `FIELDS TERMINATED BY`, `ENCLOSED BY` and `LINES TERMINATED BY` parameters control the output format. The file is created by the MySQL system user (not the user running the query), so it must be in a directory with the correct permissions.

## What it's for

`INTO OUTFILE` is useful for bulk data exports from the database to structured text files. It is the complement to `LOAD DATA INFILE`, which does the reverse operation (imports data from files). Together they form MySQL's native mechanism for bulk import/export.

## When to use it

Usage is governed by the `secure-file-priv` directive: the destination file must be within the authorised directory. When `secure-file-priv` blocks the desired path, the alternative is using the mysql command-line client with `-B -e` and redirecting the output, which is not subject to the same restrictions.
