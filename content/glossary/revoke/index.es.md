---
title: "REVOKE"
description: "Comando SQL para eliminar privilegios o roles previamente asignados a un usuario o rol, complementario al comando GRANT."
translationKey: "glossary_revoke"
aka: "Revocación de privilegios SQL"
articles:
  - "/posts/oracle/oracle-roles-privileges"
---

**REVOKE** es el comando SQL que elimina privilegios o roles previamente asignados con `GRANT`. Es el complemento indispensable del GRANT y la herramienta principal para restringir permisos cuando se reestructura un modelo de seguridad.

## Cómo funciona

La sintaxis sigue el mismo esquema que GRANT: `REVOKE SELECT ON schema.tabla FROM usuario` o `REVOKE rol FROM usuario`. En Oracle, revocar un rol como `DBA` elimina de un solo golpe todos los system privileges incluidos en ese rol. Antes de ejecutar un REVOKE crítico, es fundamental verificar que el usuario mantenga los privilegios necesarios para sus funciones.

## Cuándo se usa

El caso más común es la reestructuración del modelo de seguridad: eliminar roles excesivos (como DBA de usuarios aplicativos) y sustituirlos por roles personalizados calibrados. También se usa cuando un usuario cambia de función, cuando un servicio se da de baja, o cuando una auditoría revela privilegios asignados en exceso.

## Qué puede salir mal

Un REVOKE mal planificado puede bloquear aplicaciones en producción. Si una aplicación se conecta con un usuario que pierde el privilegio `CREATE SESSION`, deja de funcionar instantáneamente. Por esto la revocación de privilegios críticos debe ir siempre precedida de un análisis de dependencias y un plan de despliegue gradual.
