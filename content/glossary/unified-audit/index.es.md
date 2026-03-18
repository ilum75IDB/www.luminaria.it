---
title: "Unified Audit"
description: "Sistema de auditoría centralizado introducido en Oracle 12c que unifica todos los tipos de auditoría en una única infraestructura, sustituyendo el antiguo audit tradicional."
translationKey: "glossary_unified-audit"
aka: "Oracle Unified Auditing"
articles:
  - "/posts/oracle/oracle-roles-privileges"
---

**Unified Audit** (Oracle Unified Auditing) es el sistema de auditoría centralizado introducido en Oracle Database 12c que sustituye los mecanismos de auditoría tradicionales con una única infraestructura unificada. Todos los eventos de auditoría convergen en la vista `UNIFIED_AUDIT_TRAIL`.

## Cómo funciona

Unified Audit se basa en **audit policies**: reglas declarativas que especifican qué acciones monitorizar (DDL, DML, logins, operaciones administrativas). Las policies se crean con `CREATE AUDIT POLICY`, se activan con `ALTER AUDIT POLICY ... ENABLE` y pueden aplicarse a usuarios específicos o globalmente. Los registros de auditoría se escriben en una cola interna y luego se persisten en la tabla de sistema.

## Para qué sirve

Responde a la pregunta fundamental de la seguridad: "¿quién hizo qué, cuándo y desde dónde?" Permite rastrear operaciones críticas como DROP TABLE, GRANT, REVOKE, accesos a datos sensibles e intentos de login fallidos. Es esencial para la compliance (GDPR, SOX, PCI-DSS) y para las investigaciones post-incidente.

## Por qué es crítico

El antiguo audit tradicional de Oracle fragmentaba los logs entre archivos del sistema, la tabla SYS.AUD$ y FGA_LOG$, haciendo el análisis complejo. Unified Audit centraliza todo en un único punto, con mejor rendimiento y gestión simplificada. En un entorno sin audit configurado, un incidente de seguridad se vuelve imposible de reconstruir.
