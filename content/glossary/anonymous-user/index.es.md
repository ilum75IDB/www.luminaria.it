---
title: "Anonymous User"
description: "Usuario MySQL/MariaDB sin nombre creado automáticamente durante la instalación. Representa un riesgo de seguridad porque puede interferir con el matching de usuarios legítimos."
translationKey: "glossary_anonymous-user"
aka: "Usuario anónimo"
articles:
  - "/posts/mysql/mysql-users-and-hosts"
---

El **Anonymous User** (usuario anónimo) es una cuenta MySQL/MariaDB con username vacío (`''@'localhost'`) que se crea automáticamente durante la instalación. No tiene nombre y frecuentemente no tiene contraseña.

## Cómo funciona

Cuando un usuario se conecta, MySQL busca la correspondencia más específica en la tabla `mysql.user`. El usuario anónimo `''@'localhost'` es más específico que `'mario'@'%'` para una conexión desde localhost, porque `'localhost'` gana sobre `'%'` en la jerarquía de especificidad. En consecuencia, Mario que se conecta desde local es autenticado como usuario anónimo y pierde todos sus privilegios.

## Para qué sirve

El usuario anónimo fue pensado para instalaciones de desarrollo donde se quería permitir conexiones sin credenciales. En producción no tiene utilidad y representa un riesgo de seguridad: puede capturar conexiones destinadas a otros usuarios y conceder acceso no autorizado.

## Cuándo se usa

Nunca en producción. La primera operación en cualquier instalación MySQL/MariaDB de producción es verificar y eliminar los usuarios anónimos con `SELECT user, host FROM mysql.user WHERE user = ''` seguido de `DROP USER ''@'localhost'`.
