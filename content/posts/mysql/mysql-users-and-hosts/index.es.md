---
title: "Usuarios MySQL: por qué 'mario' y 'mario'@'localhost' no son la misma persona"
description: "En MySQL y MariaDB la identidad de un usuario depende del host desde el que se conecta. Un caso real, el modelo de autenticación explicado a fondo y los errores más comunes en la gestión de accesos."
date: "2026-01-13T10:00:00+01:00"
draft: false
translationKey: "mysql_users_and_hosts"
tags: ["mariadb", "security", "users", "privileges", "authentication"]
categories: ["mysql"]
image: "mysql-users-and-hosts.cover.jpg"
---

Hace unas semanas un cliente me llama. Tono pragmático, petición aparentemente banal:

> "Necesito crear un usuario en MySQL para una aplicación que debe acceder a una base de datos. ¿Puedes encargarte?"

Claro. `CREATE USER`, {{< glossary term="grant" >}}`GRANT`{{< /glossary >}}, siguiente.

Solo que después añade: "La aplicación corre en dos servidores diferentes. Y a veces también nos conectaremos en local para mantenimiento."

Ahí es donde la cosa deja de ser banal. Porque en MySQL, crear "un usuario" no significa lo que piensas.

---

## El modelo de autenticación de MySQL: usuario + host

Lo primero que hay que entender — y que muchos DBA con experiencia en Oracle o PostgreSQL descubren a costa propia — es que en MySQL **la identidad de un usuario no es solo su nombre**.

Es el par `'usuario'@'host'`.

Esto significa que:

``` sql
'mario'@'localhost'
'mario'@'192.168.1.10'
'mario'@'%'
```

no son el mismo usuario. Son **tres usuarios diferentes**. Con contraseñas diferentes, privilegios diferentes, comportamientos diferentes.

Cuando MySQL recibe una conexión, mira dos cosas:
1. El nombre de usuario proporcionado
2. La dirección IP (o hostname) desde la que llega la conexión

Luego busca en la tabla `mysql.user` la fila que corresponde al par más específico. No la primera encontrada. La más específica.

---

## ¿Por qué este modelo?

La elección de diseño no es casual. MySQL nació en 1995 para la web. Entornos donde la misma base de datos sirve aplicaciones que corren en máquinas diferentes, redes diferentes, con necesidades de acceso diferentes.

El modelo `usuario@host` permite:

- dar acceso completo desde localhost (para el DBA)
- dar acceso limitado desde un application server específico
- bloquear todo lo demás

Sin firewall. Sin VPN. Directamente en el motor de autenticación.

Es un modelo potente. Pero si no lo entiendes, te muerde.

---

## El caso del cliente: cómo lo resolví

Volvamos a la petición. La aplicación corre en dos servidores (`192.168.1.20` y `192.168.1.21`) y también se necesita acceso local para mantenimiento.

La tentación es crear un único usuario con `'%'` (comodín = cualquier host):

``` sql
CREATE USER 'app_ventas'@'%' IDENTIFIED BY 'PasswordSegura#2026';
GRANT SELECT, INSERT, UPDATE ON ventas_db.* TO 'app_ventas'@'%';
```

¿Funciona? Sí. ¿Es correcto? No.

El problema del `'%'` es que acepta conexiones desde **cualquier IP**. Si mañana alguien encuentra la contraseña, puede conectarse desde cualquier punto de la red. O del mundo, si la base de datos está expuesta.

La solución correcta es crear **usuarios específicos para cada origen**:

``` sql
-- Acceso desde el application server primario
CREATE USER 'app_ventas'@'192.168.1.20' IDENTIFIED BY 'PasswordSegura#2026';
GRANT SELECT, INSERT, UPDATE ON ventas_db.* TO 'app_ventas'@'192.168.1.20';

-- Acceso desde el application server secundario
CREATE USER 'app_ventas'@'192.168.1.21' IDENTIFIED BY 'PasswordSegura#2026';
GRANT SELECT, INSERT, UPDATE ON ventas_db.* TO 'app_ventas'@'192.168.1.21';

-- Acceso local para mantenimiento (privilegios diferentes)
CREATE USER 'app_ventas'@'localhost' IDENTIFIED BY 'PasswordMant#2026';
GRANT SELECT ON ventas_db.* TO 'app_ventas'@'localhost';
```

Tres usuarios. Mismo nombre. Privilegios calibrados.

El usuario local tiene solo `SELECT` porque sirve para verificaciones, no para escribir datos. Contraseña diferente porque el contexto de uso es diferente.

Principio del {{< glossary term="least-privilege" >}}privilegio mínimo{{< /glossary >}}. Aplicado en el punto correcto.

---

## La trampa del matching: ¿quién gana?

Aquí es donde nacen la mayoría de los errores.

Si existen tanto `'mario'@'%'` como `'mario'@'localhost'`, y Mario se conecta desde localhost, ¿qué usuario se usa?

Respuesta: **`'mario'@'localhost'`**.

MySQL ordena las filas en la tabla `mysql.user` de la más específica a la menos específica:

1. Host literal exacto (`192.168.1.20`)
2. Patrón con comodín (`192.168.1.%`)
3. Comodín total (`%`)

Y usa la **primera coincidencia** en orden de especificidad.

El problema clásico es este: creas `'mario'@'%'` con todos los privilegios. Después alguien crea `'mario'@'localhost'` sin privilegios (o con una contraseña diferente). Desde ese momento, Mario ya no puede entrar desde local y nadie entiende por qué.

He visto este escenario al menos una docena de veces en producción. La solución es siempre la misma: **verifica qué existe antes de crear**.

``` sql
SELECT user, host, authentication_string
FROM mysql.user
WHERE user = 'mario';
```

Si no lo haces antes, lo harás después. Con más urgencia y menos calma.

---

## MySQL vs MariaDB: las diferencias que importan

El modelo `usuario@host` es idéntico entre MySQL y MariaDB. Pero hay diferencias de implementación que vale la pena conocer.

**{{< glossary term="authentication-plugin" >}}Autenticación{{< /glossary >}} por defecto:**

| Versión | Plugin por defecto |
|---|---|
| MySQL 5.7 | `mysql_native_password` |
| MySQL 8.0+ | `caching_sha2_password` |
| MariaDB 10.x | `mysql_native_password` |

Si migras de MariaDB a MySQL 8 (o viceversa), los clientes podrían no conectarse porque el plugin de autenticación es diferente. No es un bug. Es un cambio de configuración por defecto.

**Creación de usuarios:**

En MySQL 8, `GRANT` ya no crea usuarios implícitamente. Debes hacer `CREATE USER` primero y `GRANT` después. Siempre.

``` sql
-- MySQL 8: correcto
CREATE USER 'app'@'10.0.0.5' IDENTIFIED BY 'pwd123';
GRANT SELECT ON mydb.* TO 'app'@'10.0.0.5';

-- MySQL 5.7 / MariaDB: todavía funciona (pero está deprecado)
GRANT SELECT ON mydb.* TO 'app'@'10.0.0.5' IDENTIFIED BY 'pwd123';
```

Si estás escribiendo scripts de provisioning, este detalle puede romper una pipeline CI/CD entera.

**Roles:**

MySQL 8.0 introdujo los roles. MariaDB los soporta desde la 10.0.5, pero con sintaxis ligeramente diferente.

``` sql
-- MySQL 8.0
CREATE ROLE 'role_lectura';
GRANT SELECT ON ventas_db.* TO 'role_lectura';
GRANT 'role_lectura' TO 'app_ventas'@'192.168.1.20';
SET DEFAULT ROLE 'role_lectura' FOR 'app_ventas'@'192.168.1.20';

-- MariaDB 10.x
CREATE ROLE role_lectura;
GRANT SELECT ON ventas_db.* TO role_lectura;
GRANT role_lectura TO 'app_ventas'@'192.168.1.20';
SET DEFAULT ROLE role_lectura FOR 'app_ventas'@'192.168.1.20';
```

La diferencia parece cosmética (comillas o no), pero en scripts automatizados puede generar errores sintácticos.

---

## El usuario anónimo: el fantasma que nadie invitó

MySQL viene instalado con un {{< glossary term="anonymous-user" >}}usuario anónimo{{< /glossary >}}: `''@'localhost'`. Sin nombre, sin contraseña.

Este usuario es un residuo histórico de las instalaciones de desarrollo. En producción es un riesgo de seguridad puro.

El usuario anónimo gana sobre `'mario'@'%'` cuando la conexión llega desde localhost, porque `'localhost'` es más específico que `'%'`.

Resultado: Mario se conecta desde local, MySQL lo autentica como usuario anónimo, y los privilegios de Mario desaparecen.

Lo primero que hay que hacer en cualquier instalación MySQL/MariaDB en producción:

``` sql
SELECT user, host FROM mysql.user WHERE user = '';

-- Si se encuentra:
DROP USER ''@'localhost';
DROP USER ''@'%';  -- si existe
{{< glossary term="flush-privileges" >}}FLUSH PRIVILEGES{{< /glossary >}};
```

No es paranoia. Es higiene.

---

## Checklist operativa

Después de la experiencia con el cliente, formalicé una checklist que uso cada vez que debo crear usuarios en MySQL o MariaDB:

1. **Verifica usuarios existentes** con el mismo nombre en hosts diferentes
2. **Elimina usuarios anónimos** si están presentes
3. **Crea usuarios con hosts específicos**, nunca con `'%'` en producción si no es estrictamente necesario
4. **Asigna solo los privilegios necesarios** — `SELECT` si basta `SELECT`
5. **Usa `CREATE USER` + `GRANT` separados** (obligatorio en MySQL 8)
6. **Verifica el plugin de autenticación** si los clientes tienen problemas de conexión
7. **Documenta los pares usuario/host** — en seis meses nadie recordará por qué existen tres "app_ventas"

---

## Conclusión

En MySQL y MariaDB un usuario no es un nombre. Es un nombre ligado a un punto de origen.

Este modelo es potente porque permite segmentar los accesos sin infraestructura adicional. Pero también es fuente de errores sutiles si no se comprende a fondo.

La próxima vez que alguien te pida "crear un usuario en MySQL", antes de escribir el primer `CREATE USER`, pregúntate: **¿desde dónde se conectará?**

La respuesta a esa pregunta lo cambia todo.

------------------------------------------------------------------------

## Glosario

**[GRANT](/es/glossary/grant/)** — Comando SQL para asignar privilegios a un usuario o rol. En MySQL 8 ya no crea usuarios implícitamente: primero `CREATE USER`, luego `GRANT`.

**[Least Privilege](/es/glossary/least-privilege/)** — Principio de seguridad que prevé asignar solo los permisos estrictamente necesarios. En MySQL se aplica calibrando privilegios por par usuario/host.

**[Authentication Plugin](/es/glossary/authentication-plugin/)** — Módulo que gestiona la verificación de credenciales. El default cambia entre MySQL 5.7 (`mysql_native_password`), MySQL 8 (`caching_sha2_password`) y MariaDB.

**[Anonymous User](/es/glossary/anonymous-user/)** — Usuario MySQL sin nombre (`''@'localhost'`) creado automáticamente durante la instalación. Puede interferir con el matching de usuarios legítimos y debe eliminarse en producción.

**[FLUSH PRIVILEGES](/es/glossary/flush-privileges/)** — Comando que recarga las tablas de grant en memoria, haciendo efectivos los cambios manuales de privilegios. Necesario después de operaciones directas sobre la tabla `mysql.user`.
