---
title: "Roles y usuarios en PostgreSQL: por qué todo es (solo) un ROLE"
description: "PostgreSQL no distingue entre usuarios y roles: todo es un ROLE. El modelo mental correcto, un caso real y un ejemplo completo para construir un usuario read-only realmente mantenible."
date: "2026-02-23T09:34:00+01:00"
lastmod: "2026-02-26T09:34:00+01:00"
draft: false
translationKey: "postgresql_roles_and_users"
tags: ["postgresql", "security", "roles", "privileges", "grant", "revoke"]
categories: ["postgresql", "security"]
image: "postgresql_roles_and_users.cover.jpg"
---

La primera vez que trabajé seriamente con PostgreSQL venía de
años utilizando otros motores de base de datos. Buscaba el comando `CREATE USER`. Lo encontraba.
Luego veía `CREATE ROLE`. Luego `ALTER USER`. Luego `ALTER ROLE`.\
Durante unos minutos pensé: "Vale, aquí alguien se divierte confundiendo
a la gente".

En realidad no. PostgreSQL es mucho más coherente de lo que parece.
Solo que lo es a su manera.

## En PostgreSQL no existen usuarios. Existen roles.

La clave es esta: **en PostgreSQL todo es un ROLE**.

Un ROLE puede:

-   tener derecho de login\
-   no tener derecho de login\
-   poseer objetos\
-   heredar permisos de otros roles\
-   ser utilizado como contenedor de privilegios

Lo que en otros motores llamas "usuario" en PostgreSQL es
simplemente un rol con el atributo `LOGIN`.

De hecho:

``` sql
CREATE USER mario;
```

no es más que un atajo para:

``` sql
CREATE ROLE mario WITH LOGIN;
```

Lo mismo ocurre con `ALTER USER`: es solo un alias de `ALTER ROLE`.

¿Por qué solo existen realmente `CREATE ROLE` y `ALTER ROLE`?\
Porque PostgreSQL no distingue conceptualmente entre usuario y rol.
Es el mismo objeto con atributos distintos. Minimalista. Elegante. Coherente.

Si un rol tiene `LOGIN`, se comporta como un usuario.\
Si no tiene `LOGIN`, es un contenedor de permisos.

Cuando lo entiendes de verdad, cambia la forma en que diseñas la seguridad.

------------------------------------------------------------------------

## El modelo mental correcto

Hoy razono así:

-   Creo roles "funcionales" que representan conjuntos de privilegios\
-   Asigno esos roles a los usuarios reales\
-   Evito conceder permisos directamente a los usuarios

¿Por qué? Porque los usuarios cambian. Los roles no.

Si mañana llega un nuevo compañero, no reescribo medio sistema de grants.\
Le asigno el rol adecuado y listo.

Arquitectura limpia. Cero magia. Cero caos.

------------------------------------------------------------------------

## Una historia real (sin nombres incómodos)

Hace un tiempo me pidieron crear un usuario de solo lectura para un
sistema de monitorización.\
Solicitud aparentemente simple: "Debe leer algunas tablas. Nada de escritura."

El clásico "si total es solo read-only".

La trampa es siempre la misma: si solo ejecutas un `GRANT SELECT` sobre
las tablas existentes, funciona hoy.\
Dentro de tres meses alguien crea una nueva tabla y el sistema de monitorización empieza
a lanzar errores.\
Y adivina a quién llaman.

La solución correcta requiere atención en cuatro niveles:

1.  Permiso de conexión al database\
2.  Permiso de uso del esquema (`USAGE`)\
3.  Permisos `SELECT` sobre tablas y secuencias existentes\
4.  Default privileges para los objetos futuros

Si te saltas una pieza, tarde o temprano pagas el precio.

------------------------------------------------------------------------

## Ejemplo: creación correcta de un usuario read-only

Supongamos que queremos crear un usuario de solo lectura sobre dos esquemas.

Primero creo el rol con login:

``` sql
CREATE ROLE srv_monitorizacion 
WITH LOGIN 
PASSWORD 'PasswordSegura123#';
```

Lo aseguro:

``` sql
ALTER ROLE srv_monitorizacion 
NOSUPERUSER NOCREATEDB NOCREATEROLE NOINHERIT;
```

Permito la conexión al database:

``` sql
GRANT CONNECT ON DATABASE mydb TO srv_monitorizacion;
```

Permiso de uso de los esquemas:

``` sql
GRANT USAGE ON SCHEMA schema1 TO srv_monitorizacion;
GRANT USAGE ON SCHEMA schema2 TO srv_monitorizacion;
```

Permisos de lectura sobre los objetos existentes:

``` sql
GRANT SELECT ON ALL TABLES IN SCHEMA schema1 TO srv_monitorizacion;
GRANT SELECT ON ALL TABLES IN SCHEMA schema2 TO srv_monitorizacion;

GRANT SELECT ON ALL SEQUENCES IN SCHEMA schema1 TO srv_monitorizacion;
GRANT SELECT ON ALL SEQUENCES IN SCHEMA schema2 TO srv_monitorizacion;
```

Y ahora la parte que muchos olvidan:

``` sql
ALTER DEFAULT PRIVILEGES IN SCHEMA schema1
GRANT SELECT ON TABLES TO srv_monitorizacion;

ALTER DEFAULT PRIVILEGES IN SCHEMA schema2
GRANT SELECT ON TABLES TO srv_monitorizacion;
```

Así también las tablas futuras serán legibles.

Nota importante: los `ALTER DEFAULT PRIVILEGES` se aplican al rol que
crea los objetos. Si varios owners crean tablas en los mismos
esquemas, la configuración debe replicarse para cada uno.

------------------------------------------------------------------------

## Por qué este modelo es potente

El hecho de que todo sea un ROLE permite construir jerarquías limpias.

Ejemplo avanzado:

``` sql
CREATE ROLE role_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA schema1 TO role_readonly;

CREATE ROLE srv_monitorizacion WITH LOGIN PASSWORD '...';
GRANT role_readonly TO srv_monitorizacion;
```

Ahora puedo asignar `role_readonly` a diez usuarios distintos sin
duplicar grants.

Esto es diseño. No solo sintaxis.

------------------------------------------------------------------------

## Conclusión

PostgreSQL no complica el concepto de usuario. Lo simplifica.\
Existe un único tipo de objeto: el ROLE. Depende de nosotros usarlo bien.

Si lo tratas como un simple "usuario con contraseña", funciona.\
Si lo usas como un bloque arquitectónico, se convierte en una herramienta poderosa
para diseñar una seguridad limpia, escalable y mantenible.

La diferencia no está en los comandos.\
Está en el modelo mental con el que los utilizas.
