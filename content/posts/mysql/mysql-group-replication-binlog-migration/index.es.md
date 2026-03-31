---
title: "Disco lleno en un cluster MySQL: binary logs, Group Replication y una migración que no admite errores"
description: "Filesystem al 92% en un cluster MySQL Group Replication de 3 nodos. ¿La causa? Binary logs acumulados en el volumen principal. Desde la alerta hasta la migración a un volumen dedicado, nodo por nodo, sin perder el quórum."
date: "2025-10-14T08:03:00+01:00"
draft: false
translationKey: "mysql_group_replication_binlog_migration"
tags: ["group-replication", "binary-log", "disk-space", "cluster", "innodb-cluster"]
categories: ["mysql"]
image: "mysql-group-replication-binlog-migration.cover.jpg"
---

La alerta llegó un lunes por la mañana, entre tres reuniones y un café todavía caliente. "Filesystem /mysql al 85% en el nodo primario." En otro nodo estaba al 66%, en el tercero al 25%. En un cluster, cuando los números no cuadran entre los nodos, siempre hay algo debajo.

La primera pregunta que te viene a la cabeza es "¿cuánto espacio necesitamos?". Pero es la pregunta equivocada. La correcta es: "¿por qué se está llenando?"

---

## La causa: binary logs en el volumen equivocado

Verificar fue rápido:

```sql
SHOW VARIABLES LIKE 'log_bin';
```

Resultado: `ON`. Los binary logs estaban activos — como se espera en un cluster. Pero el path era el problema:

```sql
SHOW VARIABLES LIKE 'log_bin_basename';
```

```
/mysql/bin_log/binlog
```

Los binlogs estaban en el mismo volumen que los datos: `/mysql`. Un volumen de unos 3 TB que en un nodo ya iba por el 85%.

También comprobé la retención:

```sql
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';
```

```
604800
```

Siete días. No es un valor disparatado, pero con tres nodos escribiendo binlogs locales en un volumen compartido con los datos, siete días pueden pesar mucho — especialmente si la carga de escritura es alta.

El cuadro era claro: los binary logs estaban devorando el filesystem principal. No un bug, no una tabla descontrolada. Solo una elección arquitectónica hecha en la instalación y nunca revisada.

---

## ¿Qué tipo de cluster es exactamente?

Antes de tocar cualquier cosa en un servidor MySQL — antes siquiera de pensar en mover un archivo — necesitas saber qué tienes delante. "Es un cluster" no basta. MySQL tiene al menos cuatro formas distintas de hacer alta disponibilidad, y cada una tiene sus propias reglas.

Empecé con la replicación clásica:

```sql
SHOW SLAVE STATUS\G
```

Empty set en los dos nodos que comprobé. Ninguna replicación tradicional activa.

Después probé con `SHOW REPLICA STATUS` — pero en MySQL 8.0.20 ese comando todavía no existe. Se introdujo en la 8.0.22. Un detalle que la documentación online suele olvidar mencionar, dejándote persiguiendo un error de sintaxis que no es tal.

Siguiente paso — Group Replication:

```sql
SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
```

Y ahí estaba la respuesta:

| MEMBER_HOST | MEMBER_STATE | MEMBER_ROLE |
|-------------|-------------|-------------|
| dbcluster01 | ONLINE | SECONDARY |
| dbcluster02 | ONLINE | SECONDARY |
| dbcluster03 | ONLINE | PRIMARY |

Tres nodos. Todos ONLINE. Un primary, dos secondary. Group Replication en modo single-primary.

Confirmación final desde los plugins:

```sql
SHOW PLUGINS;
```

En la lista: `group_replication | ACTIVE | GROUP REPLICATION | group_replication.so`. Y desde la configuración:

```sql
SHOW VARIABLES LIKE 'group_replication_single_primary_mode';
```

```
ON
```

Ahora sabía exactamente lo que tenía delante. No una réplica clásica, no un Galera, no un NDB Cluster. Un MySQL Group Replication single-primary con tres nodos, GTID habilitados, formato de binlog ROW. El panorama completo.

La tentación siempre es saltarse esta fase. "Ya sé que es un cluster, vamos." Pero saltarse el diagnóstico en un cluster es como operar sin un TAC: puede que tengas suerte, o puede que provoques un desastre.

---

## La solución: un volumen dedicado para los binary logs

La estrategia era sencilla: los binlogs necesitan su propio volumen. No en el mismo filesystem que los datos, no en un symlink improvisado, no en un directorio compartido. Un volumen dedicado, montado en el mismo path en los tres nodos.

Pedí a los administradores de sistemas que crearan un nuevo volumen de 600 GB con punto de montaje `/mysql/binary_logs` en cada uno de los tres nodos.

Cuando el volumen estuvo listo, verifiqué en los tres:

```bash
df -h /mysql/binary_logs
```

| Nodo | /mysql | /mysql/binary_logs |
|------|--------|--------------------|
| dbcluster03 (PRIMARY) | 85% | 1% |
| dbcluster02 (SECONDARY) | 66% | 1% |
| dbcluster01 (SECONDARY) | 25% | 1% |

Espacio limpio y dedicado. Cada volumen en un disco local de la VM correspondiente — tres discos, tres volúmenes, mismo mountpoint en los tres nodos. Los administradores habían hecho un trabajo limpio.

---

## Los checks antes de tocar MySQL

Antes de parar el primer nodo, ejecuté tres comprobaciones que considero obligatorias.

**Permisos del directorio.** MySQL no arranca si no puede escribir en el directorio de binlogs. Parece obvio, pero es una de las causas más frecuentes de "¿por qué no reinicia después del cambio de configuración?"

```bash
ls -ld /mysql/binary_logs
```

En los tres nodos los permisos eran 755. Funciona, pero no es ideal para la seguridad — los binlogs pueden contener datos sensibles. Los cambié a 750:

```bash
chmod 750 /mysql/binary_logs
```

Resultado: `drwxr-x--- mysql mysql`. Solo el usuario mysql puede leer y escribir.

**Test de escritura real.** Antes de dejar que MySQL escriba ahí, verifiqué que el filesystem respondiera:

```bash
touch /mysql/binary_logs/testfile
ls -l /mysql/binary_logs/testfile
rm -f /mysql/binary_logs/testfile
```

Si el touch falla, el problema es del storage o de los permisos — y mejor descubrirlo ahora que después de un restart de MySQL.

**Estado del cluster.** La última comprobación antes de proceder:

```sql
SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
```

Tres nodos ONLINE. Quórum intacto. Se puede empezar.

---

## La estrategia: un nodo a la vez, el primary de último

En un Group Replication de tres nodos, el quórum es dos. Si paras un nodo, los otros dos mantienen el grupo. Si paras dos — has perdido el cluster.

La regla es simple: **un nodo a la vez, esperando a que el anterior reingrese al grupo antes de tocar el siguiente**. Y el primary se hace de último.

¿Por qué? Porque cuando paras el primary, sucede algo importante: el cluster lanza una elección automática y uno de los secondary se convierte en el nuevo primary. Durante esos segundos — pocos, si todo está sano — las conexiones activas pueden caerse, las transacciones en curso pueden fallar. Es una interrupción breve, pero es una interrupción. Hay que comunicarla.

El orden que seguí:

1. **dbcluster01** (SECONDARY)
2. **dbcluster02** (SECONDARY)
3. **dbcluster03** (PRIMARY)

---

## El procedimiento, nodo por nodo

En cada nodo la secuencia es idéntica:

**A. Verificar el rol del nodo.** Antes de pararlo, confirma que es lo que piensas:

```sql
SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
```

**B. Parar MySQL:**

```bash
systemctl stop mysqld
```

**C. Modificar la configuración.** En el `my.cnf`, cambiar el parámetro `log_bin`:

De:
```ini
log_bin=/mysql/bin_log/binlog
```

A:
```ini
log_bin=/mysql/binary_logs/mysql-bin
```

Una línea. Un solo cambio. No toques los parámetros de Group Replication, no cambies el `server_id`, no reinventes el motor a vapor mientras cambias una rueda.

**D. Arrancar MySQL:**

```bash
systemctl start mysqld
```

**E. Verificar.** Tres cosas que comprobar:

El nuevo path:
```sql
SHOW VARIABLES LIKE 'log_bin_basename';
```
Debe devolver `/mysql/binary_logs/mysql-bin`.

El reingreso al grupo:
```sql
SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
```
El nodo debe volver a ONLINE.

Los nuevos binlogs en el nuevo path:
```bash
ls -lh /mysql/binary_logs/
```
Deben aparecer los nuevos archivos `mysql-bin.000001`.

**Solo cuando el nodo está ONLINE y el cluster muestra de nuevo tres nodos activos se pasa al siguiente.** No antes.

Para el primary — dbcluster03 — el procedimiento es idéntico, pero antes de pararlo verifiqué que ambos secondary estuvieran ONLINE y ya migrados. En el momento del stop, el cluster lanzó la elección. Uno de los secondary se convirtió en primary. Breve interrupción, como estaba previsto.

---

## Qué no hacer

Desde mi experiencia, estas son las trampas más comunes en este tipo de intervención:

**No copies los binlogs viejos al nuevo path.** En Group Replication no hace falta hacer arqueología binaria. Los nuevos binlogs se crearán en el nuevo directorio después del reinicio. Los viejos solo se necesitan si requieres point-in-time recovery — y en ese caso ya sabes dónde encontrarlos.

**No toques dos nodos a la vez.** Con tres nodos el quórum es sagrado. Un nodo a la vez, sin excepciones. Si paras dos juntos, estás jugando a Jenga con los ojos vendados.

**No empieces por el primary.** Siempre los secondary primero, el primary de último. Hacerlo al revés es la forma elegante de invitar al caos a cenar.

**No borres los binlogs viejos inmediatamente.** Después del cambio, el path antiguo `/mysql/bin_log/` ya no se usará para los nuevos archivos. Pero no hagas `rm -rf /mysql/bin_log/*` corriendo. Espera. Verifica que el cluster esté estable, que los nuevos binlogs se escriban en el nuevo mount, que no haya errores en el log de MySQL. Solo después de unos días de observación, piensa en la limpieza.

**No te fíes solo de que "MySQL arrancó".** MySQL puede arrancar pero no reincorporarse al grupo. Necesitas verificar tres cosas: `log_bin_basename` apunta al nuevo path, el nodo está ONLINE en `replication_group_members`, y los archivos binlog se están escribiendo realmente en el nuevo directorio.

---

## Lo que esta operación enseña

Un filesystem al 92% no es una emergencia — es una señal. El problema real no era el espacio en disco, sino una elección arquitectónica hecha en el momento de la instalación y nunca revisada: binlogs y datos en el mismo volumen.

Separar los binary logs en un volumen dedicado no es solo un fix. Es endurecimiento de la infraestructura. Es la diferencia entre un sistema que "funciona" y uno que está diseñado para seguir funcionando cuando las cosas crecen.

Y la parte más importante de toda la intervención no fue el cambio en el `my.cnf` — eso es una línea. La parte importante fue el diagnóstico: entender qué tipo de cluster tenía delante, verificar el estado de cada nodo, preparar el storage, probar los permisos, planificar el orden de ejecución. Todo antes de tocar un solo parámetro.

Un DBA senior y un DBA junior conocen ambos el comando `systemctl stop mysqld`. La diferencia está en todo lo que sucede antes.

------------------------------------------------------------------------

## Glosario

**[Group Replication](/es/glossary/group-replication/)** — Mecanismo nativo de MySQL para la replicación síncrona multi-nodo con failover automático y gestión de quórum. Soporta modos single-primary y multi-primary.

**[Binary log](/es/glossary/binary-log/)** — Registro binario secuencial de MySQL que rastrea todas las modificaciones de datos (INSERT, UPDATE, DELETE, DDL), usado para la replicación y el point-in-time recovery.

**[GTID](/es/glossary/gtid/)** — Global Transaction Identifier — identificador único asignado a cada transacción en MySQL, que simplifica la gestión de la replicación y el seguimiento de transacciones entre los nodos del cluster.

**[Quorum](/es/glossary/quorum/)** — Número mínimo de nodos que deben estar activos y en comunicación para que un cluster pueda seguir operando. En un cluster de 3 nodos, el quórum es 2.

**[Single-primary](/es/glossary/single-primary/)** — Modo de Group Replication en el que solo un nodo acepta escrituras mientras los demás son de solo lectura con failover automático.
