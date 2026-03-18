---
title: "Binary log en MySQL: qué son, cómo gestionarlos y cuándo puedes borrarlos"
description: "Un servidor MySQL con el disco al 95%, 180 GB de binary log acumulados en seis meses. De ahí parte un viaje por los binlog: qué contienen, por qué existen, cómo funcionan con la replicación y el point-in-time recovery, y sobre todo cómo gestionarlos sin causar daños."
date: "2026-03-31T10:00:00+01:00"
draft: false
translationKey: "binary_log_mysql"
tags: ["binlog", "replication", "disk-space", "recovery", "mariadb"]
categories: ["mysql"]
image: "binary-log-mysql.cover.jpg"
---

El mensaje en el canal de Slack del equipo de infraestructura era de esos que te hacen levantar la cabeza de la pantalla: "Disco al 95% en el db de producción. ¿Alguien puede mirar?"

El servidor era un MySQL 8.0 sobre Rocky Linux, un sistema de gestión usado por un centenar de usuarios. La base de datos en sí ocupaba unos 40 GB — nada extraordinario. Pero en el directorio de datos había 180 GB de binary logs. Seis meses de binlog que nadie había pensado en gestionar.

No es la primera vez que veo este escenario. De hecho, diría que es uno de los patrones más recurrentes en los tickets que me llegan. El binary log es una de esas funcionalidades de MySQL que trabajan en silencio, sin pedir nada — hasta que el disco se llena.

---

## Qué son los binary log, en la práctica

El {{< glossary term="binary-log" >}}binary log{{< /glossary >}} es un registro secuencial de todos los eventos que modifican datos en la base de datos. Cada INSERT, UPDATE, DELETE, cada DDL — todo se escribe en archivos binarios numerados progresivamente: `mysql-bin.000001`, `mysql-bin.000002` y así sucesivamente.

El nombre engaña un poco. No es un "log" en el sentido del syslog o del error log — no está hecho para ser leído por un humano. Es un flujo binario estructurado que MySQL usa internamente para dos propósitos fundamentales:

1. **Replicación**: el slave lee los binlog del master para replicar las mismas operaciones
2. **{{< glossary term="pitr" >}}Point-in-time recovery (PITR){{< /glossary >}}**: después de restaurar un backup, puedes "reaplicar" los binlog para llevar los datos hasta un momento preciso

Sin el binary log, no puedes hacer ni una cosa ni la otra. Esta es la razón por la que el primer instinto — "desactivemos los binlog para que no llenen el disco" — es casi siempre equivocado.

---

## Cómo genera MySQL los binlog

El binary logging se activa con el parámetro `log_bin`. Desde MySQL 8.0 está habilitado por defecto — un cambio importante respecto a versiones anteriores donde había que activarlo explícitamente.

```ini
[mysqld]
log_bin = /var/lib/mysql/mysql-bin
server-id = 1
```

MySQL crea un nuevo archivo binlog en varias circunstancias:

- Cuando el servidor se inicia o reinicia
- Cuando el archivo actual alcanza el tamaño definido por `max_binlog_size` (por defecto: 1 GB)
- Cuando ejecutas `FLUSH BINARY LOGS`
- Cuando ocurre una rotación manual

Cada archivo binlog tiene un archivo índice asociado (`mysql-bin.index`) que registra todos los archivos binlog activos. Este archivo es crítico: si lo corrompes o lo modificas a mano, MySQL ya no sabe qué binlogs existen.

```sql
SHOW BINARY LOGS;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000147 | 1073741824|
| mysql-bin.000148 | 1073741824|
| mysql-bin.000149 | 1073741824|
| ...              |           |
| mysql-bin.000318 |  524288000|
+------------------+-----------+
172 rows in set
```

Ciento setenta y dos archivos. Cada uno de aproximadamente un gigabyte. La cuenta cuadra: 180 GB de binlog nunca purgados.

---

## El papel en la replicación

En una arquitectura master-slave, el binary log es el mecanismo de transporte de datos. El flujo es este:

1. El master escribe cada transacción en el binlog
2. El slave tiene un hilo (I/O thread) que se conecta al master y lee los binlog
3. El slave escribe lo que recibe en su propio {{< glossary term="relay-log" >}}relay log{{< /glossary >}}
4. Un segundo hilo (SQL thread) en el slave ejecuta los eventos del {{< glossary term="relay-log" >}}relay log{{< /glossary >}}

Esto significa que los binlog en el master **deben permanecer disponibles hasta que todos los slaves los hayan leído**. Si borras un binlog que el slave no ha consumido todavía, la replicación se rompe.

Antes de tocar cualquier binlog en un master, el comando a ejecutar es:

```sql
SHOW REPLICA STATUS\G
-- o, en versiones más antiguas:
SHOW SLAVE STATUS\G
```

El campo que interesa es `Relay_Master_Log_File` (o `Source_Log_File` en versiones recientes): te dice qué binlog está leyendo el slave en ese momento. Todos los archivos anteriores a ese son seguros para eliminar.

---

## Point-in-time recovery: la otra razón por la que los binlog existen

El segundo uso — a menudo subestimado — es el point-in-time recovery. El escenario es este: tienes un backup hecho a las 3 de la madrugada. A las 14:30 alguien ejecuta un `DROP TABLE` equivocado. Sin binlog, puedes restaurar el backup y pierdes todo lo que pasó entre las 3:00 y las 14:30. Con los binlog, haces el restore y luego reaplicas los binlog hasta las 14:29.

```bash
# Encontrar el evento del DROP TABLE
mysqlbinlog --start-datetime="2026-03-30 14:00:00" \
            --stop-datetime="2026-03-30 15:00:00" \
            /var/lib/mysql/mysql-bin.000318 | grep -i "DROP"

# Reaplicar los binlog hasta el momento antes del desastre
mysqlbinlog --stop-datetime="2026-03-30 14:29:00" \
            /var/lib/mysql/mysql-bin.000310 \
            /var/lib/mysql/mysql-bin.000311 \
            ... \
            /var/lib/mysql/mysql-bin.000318 | mysql -u root -p
```

En la práctica, los binlog son tu póliza de seguros. El backup es la base, los binlog cubren el delta. Borrar los binlog sin un backup reciente es como cancelar el seguro el día antes de una tormenta.

---

## PURGE BINARY LOGS: la forma correcta de limpiar

Volvamos a nuestro servidor con el disco al 95%. La tentación de hacer un `rm -f mysql-bin.*` es fuerte. Pero es equivocada, por dos razones:

1. MySQL no sabe que has borrado los archivos — el archivo índice sigue apuntando a binlogs que ya no existen
2. Si hay una réplica activa, arriesgas romper la sincronización

La forma correcta es el comando `PURGE`:

```sql
-- Eliminar todos los binlog anteriores a un archivo específico
PURGE BINARY LOGS TO 'mysql-bin.000300';

-- O eliminar todos los binlog más antiguos que una fecha determinada
PURGE BINARY LOGS BEFORE '2026-03-01 00:00:00';
```

`PURGE` hace tres cosas que `rm` no hace:

- Actualiza el archivo índice
- Verifica que los archivos no sean necesarios para la replicación (en teoría — pero compruébalo tú antes)
- Elimina los archivos de forma ordenada

En el caso de nuestro servidor, primero verifiqué que no hubiera slaves:

```sql
SHOW REPLICAS;
-- Empty set
```

Ninguna réplica. Luego comprobé cuál era el binlog actual:

```sql
SHOW MASTER STATUS;
+------------------+----------+
| File             | Position |
+------------------+----------+
| mysql-bin.000318 | 52428800 |
+------------------+----------+
```

Manteniendo los últimos 3 archivos por seguridad:

```sql
PURGE BINARY LOGS TO 'mysql-bin.000316';
```

Resultado: 175 GB liberados en pocos segundos. El disco bajó del 95% al 28%.

---

## Configurar la retención automática

Resolver la emergencia es una cosa. Hacer que no vuelva a ocurrir es otra. MySQL ofrece dos parámetros para la gestión automática de la retención:

### `expire_logs_days` (legacy)

```ini
[mysqld]
expire_logs_days = 14
```

Elimina automáticamente los binlog más antiguos de 14 días. Simple pero tosco — la granularidad es solo en días.

### `binlog_expire_logs_seconds` (MySQL 8.0+)

```ini
[mysqld]
binlog_expire_logs_seconds = 1209600   # 14 días en segundos
```

Misma lógica, pero con granularidad al segundo. Desde MySQL 8.0, este parámetro tiene prioridad sobre `expire_logs_days`. Si los configuras ambos, gana `binlog_expire_logs_seconds`.

La pregunta que siempre me hacen es: "¿Cuántos días de retención?"

Depende. Pero aquí van mis reglas prácticas:

| Escenario | Retención recomendada |
|----------|----------------------|
| Servidor standalone, backup diario | 7 días |
| Master con réplica, backup diario | 7-14 días |
| Master con réplica lenta o en zonas diferentes | 14-30 días |
| Entornos regulados (finanzas, sanidad) | 30-90 días, con archivo |

El principio es: **la retención de los binlog debe cubrir al menos el doble del intervalo entre dos backups**. Si haces backup cada noche, mantén al menos 2-3 días de binlog. Si haces backups semanales, al menos 14 días.

En el caso de nuestro servidor, no se había configurado ninguna retención. El valor por defecto de MySQL 8.0 es 30 días — pero ese valor había sido sobrescrito a 0 (sin expiración) en un `my.cnf` personalizado por alguien que "quería guardar todo por seguridad". La ironía: la seguridad que quería garantizar estaba a punto de tumbar el servidor llenando el disco.

---

## Los tres formatos del binlog: STATEMENT, ROW, MIXED

No todos los binlog son iguales. MySQL soporta tres formatos de registro, y la elección tiene implicaciones concretas.

### STATEMENT

Registra la instrucción SQL tal como fue ejecutada. Compacto, legible, pero problemático: funciones como `NOW()`, `UUID()`, `RAND()` producen resultados diferentes en el master y en el slave. Las consultas con `LIMIT` sin `ORDER BY` pueden producir resultados no deterministas.

```sql
SET binlog_format = 'STATEMENT';
```

### ROW

Registra el cambio a nivel de fila — antes y después. Más pesado en términos de espacio, pero determinista al 100%. Si actualizas 10.000 filas, el binlog contiene 10.000 imágenes before/after. Grande, pero seguro.

```sql
SET binlog_format = 'ROW';
```

### MIXED

MySQL decide caso por caso: usa STATEMENT cuando es seguro, cambia automáticamente a ROW cuando detecta operaciones no deterministas.

```sql
SET binlog_format = 'MIXED';
```

Mi consejo: **usa ROW**. Es el valor por defecto desde MySQL 5.7.7, es lo que Galera Cluster requiere, es lo que todas las herramientas de replicación modernas esperan. STATEMENT es un legado del pasado, MIXED es un compromiso que añade complejidad sin un beneficio real.

El único caso en que ROW se convierte en un problema es cuando haces operaciones masivas — un `UPDATE` sobre millones de filas genera un binlog enorme porque contiene el before y el after de cada fila. En esos casos, la solución no es cambiar el formato, sino dividir la operación en lotes:

```sql
-- En vez de esto (genera binlog gigantesco):
UPDATE orders SET status = 'archived' WHERE order_date < '2025-01-01';

-- Mejor así (lotes de 10.000):
UPDATE orders SET status = 'archived'
WHERE order_date < '2025-01-01' AND status != 'archived'
LIMIT 10000;
-- Repetir hasta 0 rows affected
```

---

## `mysqlbinlog`: leer los binlog cuando hace falta

La herramienta de línea de comandos {{< glossary term="mysqlbinlog" >}}`mysqlbinlog`{{< /glossary >}} es el único modo de inspeccionar el contenido de los archivos binlog. Se usa en dos escenarios: debug de problemas de replicación y point-in-time recovery.

```bash
# Leer un binlog en formato legible
mysqlbinlog /var/lib/mysql/mysql-bin.000318

# Filtrar por intervalo temporal
mysqlbinlog --start-datetime="2026-03-30 10:00:00" \
            --stop-datetime="2026-03-30 11:00:00" \
            /var/lib/mysql/mysql-bin.000318

# Filtrar por base de datos específica
mysqlbinlog --database=gestionale /var/lib/mysql/mysql-bin.000318

# Si el formato es ROW, decodificar en SQL legible
mysqlbinlog --verbose /var/lib/mysql/mysql-bin.000318
```

Con formato ROW, sin `--verbose` solo ves blobs binarios. Con `--verbose` obtienes las filas en formato pseudo-SQL comentado — no es bonito, pero se puede leer.

---

## El principio: gestionar los binlog, no desactivarlos

De vez en cuando alguien sugiere resolver el problema "de raíz" desactivando los binlog:

```ini
# NO HAGAS ESTO en producción
skip-log-bin
```

Sí, resuelve el problema de disco. Pero elimina:

- La posibilidad de configurar una réplica en el futuro
- El point-in-time recovery
- La capacidad de analizar qué pasó en la base de datos después de un incidente
- La compatibilidad con herramientas de {{< glossary term="cdc" >}}CDC (Change Data Capture){{< /glossary >}} como Debezium

Los binlog no son un problema. Los binlog **no gestionados** son un problema. La diferencia es un parámetro de configuración y un chequeo semanal. En el servidor que arreglé, la configuración final fue:

```ini
[mysqld]
log_bin = /var/lib/mysql/mysql-bin
server-id = 1
binlog_format = ROW
binlog_expire_logs_seconds = 604800    # 7 días
max_binlog_size = 512M
```

Un `max_binlog_size` de 512 MB en lugar del 1 GB por defecto — archivos más pequeños son más fáciles de gestionar, transferir y purgar. La retención a 7 días, con backup diario, garantiza cobertura PITR completa con ocupación de disco predecible.

---

## Revisión post intervención

Antes de cerrar el ticket, añadí un par de consultas al sistema de monitorización del cliente:

```sql
-- Espacio ocupado por los binlog
SELECT
    COUNT(*) AS num_files,
    ROUND(SUM(file_size) / 1024 / 1024 / 1024, 2) AS total_gb
FROM information_schema.BINARY_LOGS;   -- MySQL 8.0+ / Performance Schema

-- O, para todas las versiones:
SHOW BINARY LOGS;
-- y sumar manualmente o con script
```

```bash
# Alerta si los binlog superan los 20 GB
#!/bin/bash
BINLOG_SIZE=$(mysql -u monitor -p'pwd' -Bse \
  "SELECT ROUND(SUM(file_size)/1024/1024/1024,2) FROM performance_schema.binary_log_status" 2>/dev/null)

# Fallback para versiones sin performance_schema.binary_log_status
if [ -z "$BINLOG_SIZE" ]; then
    BINLOG_SIZE=$(du -sh /var/lib/mysql/mysql-bin.* 2>/dev/null | \
      awk '{sum+=$1} END {printf "%.2f", sum/1024}')
fi

if (( $(echo "$BINLOG_SIZE > 20" | bc -l) )); then
    echo "WARNING: binlog size ${BINLOG_SIZE} GB"
fi
```

Tres semanas después de la intervención, los binlog ocupaban 8 GB — exactamente dentro de la ventana prevista. El disco no ha vuelto a superar el 45%.

El binlog es como el aceite del motor: nunca piensas en él hasta que se enciende el testigo. La diferencia es que el motor te avisa. MySQL no — sigue escribiendo binlog mientras el filesystem responda. Cuando deja de responder, ya es tarde para preguntarse por qué no habías configurado la retención.

------------------------------------------------------------------------

## Glosario

**[Binary log](/es/glossary/binary-log/)** — Registro binario secuencial de MySQL que rastrea todas las modificaciones de datos (INSERT, UPDATE, DELETE, DDL), usado para la replicación y el point-in-time recovery. Habilitado por defecto desde MySQL 8.0.

**[PITR](/es/glossary/pitr/)** — Point-in-Time Recovery: técnica de restauración que combina un backup completo con los binary log para llevar la base de datos a cualquier momento en el tiempo, no solo al momento del backup.

**[Relay log](/es/glossary/relay-log/)** — Archivo de log intermedio en el slave MySQL que recibe los eventos del binary log del master antes de ser ejecutados localmente por el hilo SQL.

**[CDC](/es/glossary/cdc/)** — Change Data Capture: técnica para interceptar los cambios en los datos en tiempo real leyendo los logs de transacciones. Herramientas como Debezium leen los binary log de MySQL para propagar los cambios hacia sistemas externos.

**[mysqlbinlog](/es/glossary/mysqlbinlog/)** — Utilidad de línea de comandos de MySQL para leer, filtrar y reaplicar el contenido de los archivos binary log. Indispensable para el point-in-time recovery y el debug de replicación.
