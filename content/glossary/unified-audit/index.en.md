---
title: "Unified Audit"
description: "Centralized auditing system introduced in Oracle 12c that unifies all audit types into a single infrastructure, replacing the legacy traditional audit."
translationKey: "glossary_unified-audit"
aka: "Oracle Unified Auditing"
articles:
  - "/posts/oracle/oracle-roles-privileges"
---

**Unified Audit** (Oracle Unified Auditing) is the centralized auditing system introduced in Oracle Database 12c that replaces legacy audit mechanisms with a single unified infrastructure. All audit events converge in the `UNIFIED_AUDIT_TRAIL` view.

## How it works

Unified Audit is based on **audit policies**: declarative rules that specify which actions to monitor (DDL, DML, logins, administrative operations). Policies are created with `CREATE AUDIT POLICY`, enabled with `ALTER AUDIT POLICY ... ENABLE`, and can be applied to specific users or globally. Audit records are written to an internal queue and then persisted in the system table.

## What it's for

It answers the fundamental security question: "who did what, when, and from where?" It enables tracking of critical operations such as DROP TABLE, GRANT, REVOKE, access to sensitive data, and failed login attempts. It is essential for compliance (GDPR, SOX, PCI-DSS) and post-incident investigations.

## Why it matters

Oracle's legacy traditional audit fragmented logs across OS files, the SYS.AUD$ table, and FGA_LOG$, making analysis complex. Unified Audit centralizes everything into a single point with better performance and simplified management. In an environment with no audit configured, a security incident becomes impossible to reconstruct.
