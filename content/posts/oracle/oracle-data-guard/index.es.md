---
title: "De single instance a Data Guard: el día en que el CEO entendió el DR"
description: "Una base de datos Oracle en producción sin ninguna redundancia. Un fallo de disco que lo paró todo durante seis horas. Y la decisión del CEO de invertir en una arquitectura Active Data Guard con switchover automático."
date: "2025-12-16T10:00:00+01:00"
draft: false
translationKey: "oracle_data_guard"
tags: ["data-guard", "disaster-recovery", "high-availability", "switchover", "architecture"]
categories: ["oracle"]
image: "oracle-data-guard.cover.jpg"
---

El cliente era una empresa mediana del sector asegurador. Trescientos empleados, un aplicativo de gestión interno sobre Oracle 19c, un solo servidor físico en la sala de máquinas de la planta baja. Sin réplica. Sin standby. Sin plan de disaster recovery.

Durante cinco años todo había funcionado. Y cuando las cosas funcionan, nadie quiere gastar dinero protegiéndose de problemas que nunca han ocurrido.

## El día en que todo se paró

Un miércoles de noviembre por la mañana, a las 8:47, el disco del grupo de datos principal sufrió un fallo físico. No un error lógico, no una corrupción recuperable. Un fallo hardware. La controladora RAID perdió dos discos simultáneamente — uno llevaba semanas degradado sin que nadie se diera cuenta, el otro cedió de golpe.

La base de datos se detuvo. Las pólizas no se emitían. Los siniestros no se tramitaban. El call center respondía a los clientes diciendo "problemas técnicos, llame más tarde."

Recibí la llamada a las 9:15. Cuando llegué a la sede, el administrador de sistemas ya estaba buscando discos compatibles. Los encontró a primera hora de la tarde. Entre la sustitución, la reconstrucción del RAID y la recuperación de la base de datos desde el backup de la noche anterior, el sistema volvió a estar operativo a las 15:20.

Seis horas y media de parada total. Y la pérdida de todas las transacciones desde las 23:00 de la noche anterior hasta las 8:47 de la mañana — unas diez horas de datos, porque el backup era solo nocturno y los archived log no se copiaban a otra máquina.

Esa noche el CEO envió un email a toda la empresa. Al día siguiente me llamó: "¿Qué tenemos que hacer para que esto no vuelva a pasar?"

## El diseño

La respuesta era simple en concepto, menos en la ejecución: necesitaban una segunda base de datos, sincronizada en tiempo real, lista para asumir el rol del primario en caso de fallo.

Oracle Active {{< glossary term="data-guard" >}}Data Guard{{< /glossary >}} hace exactamente eso. Una base de datos primaria genera {{< glossary term="redo-log" >}}redo logs{{< /glossary >}}, y un standby los recibe y aplica continuamente. Si el primario muere, el standby se convierte en primario. Si todo va bien, el standby también se puede usar en modo solo lectura — para informes, para backups, para aligerar la carga.

Diseñé una arquitectura de dos nodos:

- **Primario** (`oraprod1`): el servidor existente, con los discos nuevos, en la sede principal
- **Standby** (`oraprod2`): un nuevo servidor idéntico, en el centro de datos del proveedor de hosting, a 12 km de distancia

La distancia no era casual. Suficientemente lejos para sobrevivir a un evento localizado (incendio, inundación, corte de suministro prolongado), suficientemente cerca para permitir la réplica síncrona sin latencia perceptible.

## La configuración

### Preparación del primario

El primer paso fue verificar que el primario estuviera en modo `ARCHIVELOG` con `FORCE LOGGING` activo. Sin estos dos prerrequisitos, Data Guard no tiene nada que replicar.

```sql
-- Verificar modo archivelog
SELECT log_mode FROM v$database;

-- Si es necesario, activar
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;

-- Force logging: impide operaciones NOLOGGING
ALTER DATABASE FORCE LOGGING;
```

El `FORCE LOGGING` es fundamental. Sin él, cualquier operación con cláusula `NOLOGGING` — un `CREATE TABLE AS SELECT`, un `ALTER INDEX REBUILD` — no genera redo y crea huecos en la réplica. Lo he visto ocurrir tres veces en mi carrera. La tercera vez decidí que `FORCE LOGGING` se activa siempre, sin excepciones.

### Standby redo logs

En el primario creé los standby redo logs — grupos dedicados que se usarán cuando (y si) este servidor se convierta en standby tras un switchover.

```sql
-- Standby redo logs: n+1 respecto a los redo logs online
-- Si tienes 3 grupos online, creas 4 standby
ALTER DATABASE ADD STANDBY LOGFILE GROUP 4 SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 5 SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 6 SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 7 SIZE 200M;
```

La regla es n+1: si el primario tiene tres grupos de redo log, el standby necesita cuatro. No está documentada con total claridad, pero la aprendí por las malas — con tres grupos iguales, bajo carga pesada el standby puede quedarse bloqueado esperando un grupo libre.

### Configuración de red

El `tnsnames.ora` en ambos nodos debe conocer tanto al primario como al standby. La configuración es simétrica:

```
ORAPROD1 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.10.1)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = oraprod)
    )
  )

ORAPROD2 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 10.0.5.1)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = oraprod)
    )
  )
```

El `listener.ora` en el standby debe incluir una entrada estática para la base de datos, porque durante el restore el standby aún no está abierto y el listener no puede registrarlo dinámicamente:

```
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = oraprod_DGMGRL)
      (ORACLE_HOME = /u01/app/oracle/product/19c)
      (SID_NAME = oraprod)
    )
  )
```

El sufijo `_DGMGRL` lo usa el Data Guard Broker para identificar la instancia. Sin esta entrada estática, el broker no puede conectarse al standby y las operaciones de switchover fallan con errores crípticos que te hacen perder media jornada.

### Creación del standby

Para la copia inicial de la base de datos usé un `DUPLICATE` vía {{< glossary term="rman" >}}RMAN{{< /glossary >}} a través de la red. Sin backup en cinta, sin transferencia manual de archivos. Directo, del primario al standby:

```
-- En el servidor standby, iniciar la instancia en NOMOUNT
STARTUP NOMOUNT PFILE='/u01/app/oracle/product/19c/dbs/initoraprod.ora';

-- Desde RMAN, conectado a ambos
RMAN TARGET sys/password@ORAPROD1 AUXILIARY sys/password@ORAPROD2

DUPLICATE TARGET DATABASE
  FOR STANDBY
  FROM ACTIVE DATABASE
  DORECOVER
  SPFILE
    SET db_unique_name='oraprod_stby'
    SET log_archive_dest_2=''
    SET fal_server='ORAPROD1'
  NOFILENAMECHECK;
```

El `NOFILENAMECHECK` se usa cuando las rutas de los archivos son idénticas en ambas máquinas — misma estructura de directorios, misma convención de nombres. Si las rutas difieren, se necesitan los parámetros `DB_FILE_NAME_CONVERT` y `LOG_FILE_NAME_CONVERT`.

La copia tardó unas tres horas para 400 GB a través de una línea dedicada de 1 Gbps. No la más rápida, pero es una operación que se hace una sola vez.

### Data Guard Broker

El Broker es el componente que gestiona la configuración Data Guard de forma centralizada y permite el switchover con un solo comando. Sin el Broker puedes hacer todo a mano, pero no quieres hacerlo a mano cuando el primario acaba de caer y el CEO te llama cada cinco minutos.

```sql
-- En el primario
ALTER SYSTEM SET dg_broker_start=TRUE;

-- En el standby
ALTER SYSTEM SET dg_broker_start=TRUE;
```

Luego, desde `DGMGRL` en el primario:

```
DGMGRL> CREATE CONFIGURATION dg_config AS
         PRIMARY DATABASE IS oraprod
         CONNECT IDENTIFIER IS ORAPROD1;

DGMGRL> ADD DATABASE oraprod_stby AS
         CONNECT IDENTIFIER IS ORAPROD2
         MAINTAINED AS PHYSICAL;

DGMGRL> ENABLE CONFIGURATION;
```

En ese punto, `SHOW CONFIGURATION` debe devolver:

```
Configuration - dg_config

  Protection Mode: MaxPerformance
  Members:
    oraprod   - Primary database
    oraprod_stby - Physical standby database

Fast-Start Failover: DISABLED

Configuration Status:
SUCCESS
```

La palabra que quieres ver es `SUCCESS`. Cualquier otra cosa significa que hay un problema de red, configuración o permisos que resolver antes de continuar.

## El primer switchover

Dos semanas después de la puesta en producción de la arquitectura, hice la primera prueba de switchover. Un sábado por la mañana, con el aplicativo cerrado, pero con el CEO presente — quería verlo con sus propios ojos.

```
DGMGRL> SWITCHOVER TO oraprod_stby;
```

Un solo comando. Cuarenta y dos segundos. El primario se convirtió en standby, el standby se convirtió en primario. Las aplicaciones, configuradas con el servicio correcto, se reconectaron automáticamente.

```
DGMGRL> SHOW CONFIGURATION;

Configuration - dg_config

  Protection Mode: MaxPerformance
  Members:
    oraprod_stby - Primary database
    oraprod      - Physical standby database

Fast-Start Failover: DISABLED

Configuration Status:
SUCCESS
```

Después hicimos el switchback — vuelta al primario original. Otros treinta y ocho segundos. Limpio.

El CEO miró la pantalla, me miró a mí, y dijo: "Cuarenta y dos segundos contra seis horas. ¿Por qué no lo hicimos antes?"

No le di la respuesta. La sabíamos los dos.

## Lo que no te cuentan

La configuración que he descrito funciona. Pero hay cosas que la documentación de Oracle no enfatiza lo suficiente.

**El gap de red.** La réplica síncrona (`SYNC`) garantiza cero pérdida de datos pero introduce latencia en cada commit. Con 12 km y una buena fibra, la latencia añadida era de 1-2 milisegundos — aceptable. Pero a 100 km habría sido 5-8 ms, y en una aplicación con miles de commits por segundo, la ralentización se habría notado. Por eso elegí el modo `MaxPerformance` (asíncrono) como predeterminado, aceptando la posibilidad teórica de perder unos segundos de transacciones en caso de desastre total. Para ese cliente, perder cinco segundos de datos era infinitamente mejor que perder diez horas.

**El password file.** El archivo de contraseñas del usuario `SYS` debe ser idéntico en primario y standby. Si lo cambias en uno y no en el otro, el redo transport se detiene silenciosamente. Ningún error evidente, solo un gap que crece. Lo descubrí después de una hora de debugging un domingo por la noche.

**Los temp tablespace.** El standby no replica los tablespaces temporales. Si abres el standby en lectura para informes (Active Data Guard), debes crear manualmente los temp tablespaces, de lo contrario las consultas con sort o hash join fallan con errores que no tienen nada que ver con el problema real.

```sql
-- En el standby abierto en modo solo lectura
ALTER TABLESPACE TEMP ADD TEMPFILE SIZE 2G AUTOEXTEND ON;
```

**Los parches.** Primario y standby deben estar al mismo nivel de parches. Si aplicas una PSU al primario sin aplicarla al standby, el redo podría contener estructuras que el standby no sabe interpretar. El switchover funcionará, pero después podrías tener corrupciones silenciosas. El procedimiento correcto es: parchear el standby primero, switchover, parchear el antiguo primario (ahora standby), switchback.

## Los números

A seis meses de la implementación, el balance era claro:

| Métrica | Antes | Después |
|---------|-------|---------|
| {{< glossary term="rpo" >}}RPO{{< /glossary >}} (Recovery Point Objective) | ~10 horas (backup nocturno) | < 5 segundos |
| {{< glossary term="rto" >}}RTO{{< /glossary >}} (Recovery Time Objective) | 6+ horas (restore desde backup) | < 1 minuto (switchover) |
| Disponibilidad de informes en paralelo | No | Sí (Active Data Guard) |
| Coste de infraestructura adicional | — | 1 servidor + línea dedicada |
| Pruebas de switchover realizadas | 0 | 6 (una al mes) |

El coste total del proyecto — servidor, licencias, línea dedicada, implementación — era aproximadamente una cuarta parte de lo que había costado aquel único día de parada. No en términos técnicos. En términos de pólizas no emitidas, siniestros no tramitados, clientes no atendidos.

## Lo que aprendí

El disaster recovery no es un problema técnico. Es un problema de percepción del riesgo. Mientras la base de datos funciona, el DR es un gasto. Cuando la base de datos se para, el DR es una inversión que se debería haber hecho seis meses antes.

No puedes convencer a un CEO con un diagrama arquitectónico. Solo puedes esperar a que ocurra el desastre y estar preparado con la solución. Es cínico, pero así funciona en el noventa por ciento de los casos.

Lo único que puedes hacer antes es documentar el riesgo, dejar por escrito que lo señalaste, y tener el proyecto listo en el cajón. Yo había propuesto ese proyecto dieciocho meses antes. Lo habían archivado con un "lo revisamos el año que viene."

El año que viene llegó un miércoles de noviembre por la mañana, a las 8:47.

------------------------------------------------------------------------

## Glosario

**[Data Guard](/es/glossary/data-guard/)** — Tecnologia Oracle para la replica en tiempo real de una base de datos en uno o mas servidores standby. El standby recibe y aplica continuamente los redo logs del primario, permitiendo switchover en segundos.

**[Redo Log](/es/glossary/redo-log/)** — Archivos de log donde Oracle registra cada modificacion de datos antes de escribirla en los datafiles. Son la base de la recuperacion y la replica Data Guard: sin redo, ninguna de estas operaciones es posible.

**[RPO](/es/glossary/rpo/)** — Recovery Point Objective. La cantidad maxima de datos que una organizacion puede permitirse perder en caso de desastre, medida en tiempo. Con Data Guard asincrono se reducen a pocos segundos.

**[RTO](/es/glossary/rto/)** — Recovery Time Objective. El tiempo maximo aceptable para restaurar el servicio tras un fallo. Con Data Guard y switchover automatico, se pasa de horas a menos de un minuto.

**[RMAN](/es/glossary/rman/)** — Recovery Manager. Herramienta nativa Oracle para backup, restore y recovery, incluyendo la creacion de bases de datos standby via `DUPLICATE ... FOR STANDBY FROM ACTIVE DATABASE`.
