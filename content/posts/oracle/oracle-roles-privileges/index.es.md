---
title: "Usuarios, roles y privilegios en Oracle: por qué GRANT ALL nunca es la respuesta"
description: "Un cliente donde todos los usuarios aplicativos se conectaban como schema owner con el rol DBA. Cómo reestructuré el modelo de seguridad Oracle aplicando el principio del mínimo privilegio — con SQL real, roles personalizados y Unified Audit."
date: "2026-01-27T10:00:00+01:00"
draft: false
translationKey: "oracle_roles_privileges"
tags: ["security", "roles", "privileges", "grant", "revoke", "audit"]
categories: ["oracle"]
image: "oracle-roles-privileges.cover.jpg"
---

Me ha pasado más de una vez: entro en un entorno Oracle y encuentro la misma situación. Todos los usuarios aplicativos conectados como schema owner, con el rol DBA asignado. Desarrolladores, procesos batch, herramientas de reporting — todos con los mismos privilegios que el usuario propietario de las tablas.

Cuando preguntas por qué, la respuesta siempre es alguna variante de: "Así todo funciona sin problemas de permisos."

Claro. Todo funciona. Hasta el día en que un desarrollador ejecuta un `DROP TABLE` sobre la tabla equivocada. O un batch de importación hace un `TRUNCATE` sobre una tabla de producción pensando que está en el entorno de pruebas. O alguien ejecuta un `DELETE FROM clientes` sin la cláusula `WHERE`.

Ese día el problema ya no son los permisos. Es que no tienes idea de quién hizo qué, y no tienes ninguna herramienta para impedir que vuelva a ocurrir.

---

## El contexto: un patrón que se repite

El cliente era una empresa mediana con una aplicación de gestión sobre Oracle 19c. Unos veinte usuarios entre desarrolladores, cuentas aplicativas y operadores. El schema aplicativo — llamémoslo `APP_OWNER` — contenía unas 300 tablas, unas sesenta vistas y unas cuantas docenas de procedimientos PL/SQL.

El problema era fácil de describir:

- Todos se conectaban como `APP_OWNER`
- `APP_OWNER` tenía el rol `DBA`
- Ningún audit configurado
- Ninguna separación entre quien lee y quien escribe
- Las contraseñas se compartían por email

No era negligencia. Era inercia. El sistema había crecido así a lo largo de los años, y nadie se había detenido a repensar el modelo. Funcionaba, y eso bastaba.

Hasta que un operador borró por error los datos de facturación de un trimestre entero. Ningún log, ningún rastro, ningún culpable identificable. Solo un backup de dos días antes y un vacío en los datos que llevó semanas rellenar.

---

## Cómo funciona la seguridad en Oracle: el modelo

Antes de contar lo que hice, conviene entender cómo Oracle estructura la seguridad. El modelo es diferente de PostgreSQL y de MySQL, y las diferencias no son cosméticas.

### Usuario y schema: la misma cosa (casi)

En Oracle, **crear un usuario significa crear un schema**. No son dos conceptos separados: el usuario `APP_OWNER` es también el schema `APP_OWNER`, y los objetos creados por ese usuario viven en ese schema.

``` sql
CREATE USER app_read IDENTIFIED BY "PasswordSegura#2026"
DEFAULT TABLESPACE users
TEMPORARY TABLESPACE temp
QUOTA 0 ON users;
```

La `QUOTA 0` es intencional: este usuario no debe crear objetos. Es un consumidor, no un propietario.

### Privilegios de sistema vs privilegios de objeto

Oracle distingue claramente entre:

- **System privileges**: operaciones globales como `CREATE TABLE`, `CREATE SESSION`, `ALTER SYSTEM`
- **Object privileges**: operaciones sobre objetos específicos como `SELECT ON app_owner.clientes`, `EXECUTE ON app_owner.pkg_facturas`

El rol `DBA` incluye más de 200 system privileges. Asignarlo a un usuario aplicativo es como entregar las llaves de todo el edificio a quien solo necesita entrar en una habitación.

### Los roles: predefinidos y personalizados

Oracle ofrece roles predefinidos (`CONNECT`, `RESOURCE`, `DBA`) y permite crear roles personalizados. Los roles predefinidos tienen un problema histórico: `CONNECT` y `RESOURCE` incluían privilegios excesivos en versiones anteriores. Desde Oracle 12c se han reducido, pero la costumbre de asignarlos sin pensarlo es difícil de erradicar.

El camino correcto es crear roles personalizados calibrados a las necesidades reales.

---

## La implementación: tres roles, cero ambigüedad

Diseñé tres roles: lectura, escritura y administración aplicativa.

### 1. Rol de solo lectura

``` sql
CREATE ROLE app_read_role;

-- Privilegios sobre las tablas
GRANT SELECT ON app_owner.clientes      TO app_read_role;
GRANT SELECT ON app_owner.pedidos       TO app_read_role;
GRANT SELECT ON app_owner.facturas      TO app_read_role;
GRANT SELECT ON app_owner.productos     TO app_read_role;
GRANT SELECT ON app_owner.movimientos   TO app_read_role;

-- Privilegios sobre las vistas
GRANT SELECT ON app_owner.v_informe_ventas   TO app_read_role;
GRANT SELECT ON app_owner.v_estado_pedidos   TO app_read_role;
```

En un entorno con 300 tablas no las enumeras una por una a mano. Usé un bloque PL/SQL para generar los grants:

``` sql
BEGIN
  FOR t IN (SELECT table_name FROM dba_tables
            WHERE owner = 'APP_OWNER') LOOP
    EXECUTE IMMEDIATE 'GRANT SELECT ON app_owner.'
      || t.table_name || ' TO app_read_role';
  END LOOP;
END;
/
```

Simple, repetible y sobre todo: documentado. Porque dentro de seis meses alguien tendrá que entender qué se hizo y por qué.

### 2. Rol de lectura y escritura

``` sql
CREATE ROLE app_write_role;

-- Hereda todo del rol de lectura
GRANT app_read_role TO app_write_role;

-- Agrega DML sobre las tablas operativas
GRANT INSERT, UPDATE, DELETE ON app_owner.pedidos       TO app_write_role;
GRANT INSERT, UPDATE, DELETE ON app_owner.movimientos   TO app_write_role;
GRANT INSERT, UPDATE ON app_owner.clientes              TO app_write_role;

-- Permiso de ejecución sobre los procedimientos aplicativos
GRANT EXECUTE ON app_owner.pkg_pedidos   TO app_write_role;
GRANT EXECUTE ON app_owner.pkg_facturas  TO app_write_role;
```

Nota: nada de `DELETE` sobre la tabla `clientes`. No porque sea técnicamente imposible, sino porque el proceso aplicativo prevé una desactivación, no un borrado. El privilegio refleja el proceso, no la comodidad.

### 3. Rol de administración aplicativa

``` sql
CREATE ROLE app_admin_role;

-- Hereda el rol de escritura
GRANT app_write_role TO app_admin_role;

-- Agrega DDL controlado
GRANT CREATE VIEW TO app_admin_role;
GRANT CREATE PROCEDURE TO app_admin_role;
GRANT CREATE SYNONYM TO app_admin_role;

-- Puede gestionar las tablas de configuración
GRANT INSERT, UPDATE, DELETE ON app_owner.parametros    TO app_admin_role;
GRANT INSERT, UPDATE, DELETE ON app_owner.lookup_tipos  TO app_admin_role;
```

Nada de `CREATE TABLE`, nada de `DROP ANY`, nada de `ALTER SYSTEM`. El admin aplicativo gestiona la lógica, no la estructura física.

---

## Creación de usuarios y asignación

``` sql
-- Usuario para reportes (solo lectura)
CREATE USER srv_report IDENTIFIED BY "RptSecure#2026"
DEFAULT TABLESPACE users TEMPORARY TABLESPACE temp
QUOTA 0 ON users;
GRANT CREATE SESSION TO srv_report;
GRANT app_read_role TO srv_report;

-- Usuario aplicativo (lectura y escritura)
CREATE USER srv_app IDENTIFIED BY "AppSecure#2026"
DEFAULT TABLESPACE users TEMPORARY TABLESPACE temp
QUOTA 0 ON users;
GRANT CREATE SESSION TO srv_app;
GRANT app_write_role TO srv_app;

-- DBA aplicativo (administración)
CREATE USER dba_app IDENTIFIED BY "DbaSecure#2026"
DEFAULT TABLESPACE users TEMPORARY TABLESPACE temp
QUOTA 10M ON users;
GRANT CREATE SESSION TO dba_app;
GRANT app_admin_role TO dba_app;
```

Cada usuario tiene su propia contraseña, un rol específico y una cuota de disco coherente con su propósito. `srv_report` no tiene cuota porque no debe crear nada. `dba_app` tiene 10 MB porque necesita crear vistas y procedimientos.

---

## Revocación del rol DBA

El paso más delicado: quitar el `DBA` a `APP_OWNER`.

``` sql
REVOKE DBA FROM app_owner;
```

Una línea. Pero antes de ejecutarla, verifiqué que `APP_OWNER` aún tuviera los privilegios necesarios para ser propietario de sus objetos:

``` sql
SELECT privilege FROM dba_sys_privs WHERE grantee = 'APP_OWNER';
SELECT granted_role FROM dba_role_privs WHERE grantee = 'APP_OWNER';
```

Y asigné solo los privilegios estrictamente necesarios:

``` sql
GRANT CREATE SESSION TO app_owner;
GRANT CREATE TABLE TO app_owner;
GRANT CREATE VIEW TO app_owner;
GRANT CREATE PROCEDURE TO app_owner;
GRANT CREATE SEQUENCE TO app_owner;
GRANT UNLIMITED TABLESPACE TO app_owner;
```

`APP_OWNER` sigue siendo el propietario de los objetos pero ya no tiene el poder de hacer cualquier cosa en la base de datos. Es un propietario, no un dios.

---

## Audit: saber quién hizo qué

Tener los roles correctos no basta. Hay que saber quién hizo qué, especialmente en las operaciones críticas.

Oracle desde la versión 12c ofrece **Unified Audit**, que sustituye el viejo audit tradicional con un sistema centralizado.

``` sql
-- Audit sobre operaciones DDL críticas
CREATE AUDIT POLICY pol_ddl_critico
ACTIONS CREATE TABLE, DROP TABLE, ALTER TABLE,
        TRUNCATE TABLE, CREATE USER, DROP USER,
        ALTER USER, GRANT, REVOKE;

ALTER AUDIT POLICY pol_ddl_critico ENABLE;

-- Audit sobre accesos a datos sensibles
CREATE AUDIT POLICY pol_acceso_datos
ACTIONS SELECT ON app_owner.clientes,
        DELETE ON app_owner.facturas,
        UPDATE ON app_owner.facturas;

ALTER AUDIT POLICY pol_acceso_datos ENABLE;

-- Audit sobre logins fallidos
CREATE AUDIT POLICY pol_login_fallidos
ACTIONS LOGON;
ALTER AUDIT POLICY pol_login_fallidos
ENABLE WHENEVER NOT SUCCESSFUL;
```

Para verificar qué se está registrando:

``` sql
SELECT * FROM unified_audit_trail
WHERE event_timestamp > SYSDATE - 7
ORDER BY event_timestamp DESC;
```

El audit no es paranoia. Es la única forma de responder a la pregunta "¿quién hizo qué?" sin tener que recurrir a la intuición.

---

## La comparación con PostgreSQL y MySQL

Este artículo es el tercero de una serie sobre gestión de seguridad en bases de datos relacionales. Los dos primeros cubren [PostgreSQL](/es/posts/postgresql/security/postgresql_roles_and_users/) y [MySQL](/es/posts/mysql/security/mysql-users-and-hosts/).

Las diferencias entre los tres sistemas son sustanciales:

| Aspecto | Oracle | PostgreSQL | MySQL |
|---|---|---|---|
| ¿Usuario = schema? | Sí | No (independientes) | Sí (bases de datos separadas) |
| Modelo de roles | Predefinidos + custom | Todo es un ROLE | Roles desde MySQL 8.0 |
| Identidad | Nombre de usuario | Nombre de usuario | Par usuario@host |
| Audit nativo | Unified Audit (12c+) | pgAudit (extensión) | Audit plugin |
| Privilegios granulares | System + Object | Database/Schema/Object | Global/DB/Table/Column |
| GRANT ALL | Existe pero peligroso | Existe, desaconsejado | Existe, desaconsejado |

En PostgreSQL todo es un ROLE, y la simplicidad del modelo es su punto fuerte. En MySQL la identidad está vinculada al host de origen, lo que agrega una capa de complejidad (y seguridad) que los otros no tienen. En Oracle el modelo es el más rico y el más granular, pero también el más fácil de configurar mal por exceso de opciones.

El principio sigue siendo el mismo en todas partes: **dale a cada uno solo lo que necesita, ni un privilegio más**.

---

## Qué cambió después

La transición fue gradual — dos semanas para el despliegue completo, con pruebas en cada aplicación y procedimiento. Algunos scripts dejaron de funcionar porque daban por sentados privilegios que no les correspondían. Cada error era en realidad un problema oculto que antes era invisible.

El resultado:

- **20 usuarios nominales** en lugar de un único schema compartido
- **3 roles personalizados** en lugar del rol DBA
- **Audit activo** sobre DDL y operaciones sensibles
- **Cero incidentes** de borrado accidental en los meses siguientes

El cliente no notó mejoras de rendimiento. No era ese el objetivo. Lo que notó fue que cuando alguien cometía un error, el daño era contenido y rastreable. Y eso, en un entorno de producción, vale más que cualquier optimización.

---

## Conclusión

`GRANT ALL PRIVILEGES` y el rol `DBA` son atajos. Funcionan en el sentido de que eliminan los errores de permisos. Pero también eliminan cualquier capa de protección.

La seguridad en Oracle no es un problema de herramientas — las herramientas están ahí, y son potentes. Es un problema de diseño: decidir quién puede hacer qué, documentarlo, implementarlo y luego verificar que funciona.

No es el trabajo más glamuroso del mundo. Pero es el que marca la diferencia entre una base de datos que simplemente sobrevive y una que está verdaderamente bajo control.

------------------------------------------------------------------------

## Glosario

**[System Privilege](/es/glossary/system-privilege/)** — Privilegio Oracle que autoriza operaciones globales en la base de datos como CREATE TABLE, CREATE SESSION o ALTER SYSTEM, independientes de cualquier objeto específico.

**[Object Privilege](/es/glossary/object-privilege/)** — Privilegio Oracle que autoriza operaciones sobre un objeto específico de la base de datos como SELECT, INSERT o EXECUTE sobre una tabla, vista o procedimiento.

**[REVOKE](/es/glossary/revoke/)** — Comando SQL para eliminar privilegios o roles previamente asignados a un usuario o rol, complementario al comando GRANT.

**[Unified Audit](/es/glossary/unified-audit/)** — Sistema de auditoría centralizado introducido en Oracle 12c que unifica todos los tipos de auditoría en una única infraestructura, sustituyendo el antiguo audit tradicional.

**[Least Privilege](/es/glossary/least-privilege/)** — Principio de seguridad que prevé asignar a cada usuario solo los permisos estrictamente necesarios para desempeñar su función.
