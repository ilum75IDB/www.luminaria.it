---
title: "Galera Cluster con 3 nodos: cómo resolví un problema de disponibilidad en MySQL"
description: "Un cliente con un MySQL standalone que caía cada mes llevándose la aplicación entera. Mi solución: un Galera Cluster de 3 nodos con replicación síncrona. Desde el diagnóstico hasta la puesta en producción, con todos los archivos de configuración y los parámetros críticos."
date: "2026-02-17T10:00:00+01:00"
draft: false
translationKey: "galera_cluster_3_nodi"
tags: ["mariadb", "galera", "cluster", "high-availability", "replication", "wsrep"]
categories: ["mysql"]
image: "galera-cluster-3-nodi.cover.jpg"
---

El ticket era lacónico, como suele pasar cuando el problema es grave: "La base de datos se cayó otra vez. La aplicación está parada. Tercera vez en dos meses."

El cliente tenía un MariaDB en un único servidor Linux — una aplicación de gestión empresarial usada por unos doscientos usuarios internos, con picos de carga durante los cierres contables de fin de mes. Cada vez que el servidor tenía un problema — un disco que se ralentizaba, una actualización del sistema que requería reinicio, un proceso que consumía toda la RAM — la base de datos caía y con ella toda la operatividad empresarial.

La pregunta no era "cómo reparamos el servidor". La pregunta era: **¿cómo hacemos para que la próxima vez que un servidor tenga un problema, la aplicación siga funcionando?**

La respuesta, tras veinte años de experiencia con este tipo de escenarios, era una: **Galera Cluster**.

---

## El diagnóstico: un single point of failure clásico

Lo primero que hice fue analizar la infraestructura. El cuadro era familiar:

- Un único servidor MariaDB, ninguna réplica
- Backup nocturno en disco externo (al menos eso)
- Ningún mecanismo de failover
- La aplicación apuntaba directamente a la IP del servidor de base de datos

Cada caída, aunque fuera de diez minutos, significaba doscientas personas paradas. Durante los cierres contables, significaba retrasos que se propagaban en cascada por los procesos empresariales.

Propuse al cliente una solución basada en Galera Cluster: tres nodos MariaDB con replicación síncrona multi-master. Cualquier nodo acepta lecturas y escrituras, los datos son coherentes en los tres, y si un nodo cae los otros dos siguen sirviendo la aplicación sin interrupción.

El cliente ya tenía disponibles tres VM Linux — el equipo de infraestructura las había provisionado para otro proyecto que se pospuso. Perfecto: ni siquiera hacía falta pedir hardware.

---

## El plan: tres nodos, cero single point of failure

Las máquinas disponibles:

| Nodo | Hostname | IP |
|------|----------|-----|
| Nodo 1 | `db-node1` | `10.0.1.11` |
| Nodo 2 | `db-node2` | `10.0.1.12` |
| Nodo 3 | `db-node3` | `10.0.1.13` |

Una premisa importante: **Galera no es una opción nativa de MySQL Community**. Se usa MariaDB (que lo integra nativamente) o Percona XtraDB Cluster (basado en MySQL). El cliente ya usaba MariaDB, así que la elección era natural y no requería migración de motor.

El objetivo era claro: migrar de una arquitectura de nodo único a un cluster de tres nodos, sin cambiar la aplicación más allá de la dirección de conexión.

---

## Instalación: misma versión en todos los nodos

Primer principio innegociable: **todos los nodos deben tener exactamente la misma versión de MariaDB**. He visto clusters inestables durante meses porque alguien había actualizado un nodo y no los demás.

En los tres servidores:

```bash
# Añadir el repositorio oficial de MariaDB (ejemplo para 11.4 LTS)
curl -LsS https://downloads.mariadb.com/MariaDB/mariadb_repo_setup | \
    sudo bash -s -- --mariadb-server-version="mariadb-11.4"

# Instalar MariaDB Server y el plugin Galera
sudo dnf install MariaDB-server MariaDB-client galera-4 -y

# Habilitar pero NO iniciar aún el servicio
sudo systemctl enable mariadb
```

No iniciar el servicio. Primero se configura. Siempre.

---

## El corazón de la configuración: `/etc/my.cnf.d/galera.cnf`

Este es el archivo que define el comportamiento del cluster. Debe crearse en cada nodo, con las diferencias apropiadas de dirección IP y nombre de nodo.

Aquí la configuración completa para el **Nodo 1**:

```ini
[mysqld]
# === Motor y charset ===
binlog_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
innodb_flush_log_at_trx_commit=2
innodb_buffer_pool_size=1G

# === Configuración WSREP (Galera) ===
wsrep_on=ON
wsrep_provider=/usr/lib64/galera-4/libgalera_smm.so

# Lista de TODOS los nodos del cluster
wsrep_cluster_address="gcomm://10.0.1.11,10.0.1.12,10.0.1.13"

# Nombre del cluster (debe ser idéntico en todos los nodos)
wsrep_cluster_name="galera_produccion"

# Identidad de ESTE nodo (cambia en cada servidor)
wsrep_node_address="10.0.1.11"
wsrep_node_name="db-node1"

# Método SST (State Snapshot Transfer)
wsrep_sst_method=mariabackup
wsrep_sst_auth="sst_user:password_segura_aqui"

# === Red ===
bind-address=0.0.0.0
```

Para el **Nodo 2** y el **Nodo 3**, lo único que cambia es:

```ini
# Nodo 2
wsrep_node_address="10.0.1.12"
wsrep_node_name="db-node2"

# Nodo 3
wsrep_node_address="10.0.1.13"
wsrep_node_name="db-node3"
```

Todo lo demás es idéntico. **Idéntico**. No ceder a la tentación de "personalizar" buffer pool u otros parámetros por nodo: en un cluster Galera, la simetría es una virtud.

---

## Por qué cada parámetro importa

Veamos los parámetros uno por uno, porque cada uno tiene una razón precisa.

### `binlog_format=ROW`

Galera requiere el formato ROW para el binary log. Ni STATEMENT, ni MIXED. **ROW** y punto. Con otros formatos el cluster ni siquiera arranca — y con razón, porque la replicación síncrona basada en certification depende de la precisión a nivel de fila.

### `innodb_autoinc_lock_mode=2`

Este parámetro configura el modo de bloqueo para auto-increment en "interleaved". En un cluster multi-master, dos nodos pueden generar INSERT simultáneamente en la misma tabla. Con el lock mode 1 (el predeterminado) se crearían deadlocks. Con el valor 2, InnoDB genera los auto-increment sin lock global, permitiendo inserciones concurrentes desde nodos diferentes.

La consecuencia: los IDs auto-increment **no serán secuenciales** entre nodos. Si tu aplicación depende de la secuencialidad de los IDs, tienes un problema arquitectónico que resolver antes.

### `innodb_flush_log_at_trx_commit=2`

Aquí se hace un compromiso consciente. El valor 1 (predeterminado) garantiza durabilidad total — cada commit se escribe y se hace fsync al disco. Pero en un cluster Galera, la durabilidad ya está garantizada por la replicación síncrona en tres nodos. El valor 2 escribe en el buffer del sistema operativo en cada commit y hace fsync solo cada segundo, mejorando el rendimiento de escritura en un 30-40% en nuestras pruebas.

Si pierdes un nodo, los datos están en los otros dos. Si pierdes el datacenter entero... bueno, esa es otra conversación.

### `wsrep_sst_method=mariabackup`

SST es el mecanismo por el cual un nodo que se une al cluster recibe una copia completa de los datos. Las opciones son:

| Método | Pro | Contra |
|--------|-----|--------|
| `rsync` | Rápido | El nodo donante se bloquea en lectura |
| `mariabackup` | No bloquea al donante | Requiere instalación separada |
| `mysqldump` | Simple | Lentísimo, bloquea al donante |

**Siempre mariabackup**. En producción, bloquear un nodo donante durante un SST significa degradar el cluster en el momento que más lo necesitas.

```bash
# Instalar mariabackup en todos los nodos
sudo dnf install MariaDB-backup -y
```

---

## Firewall: los puertos que Galera necesita abiertos

Este es el punto donde veo fallar el 50% de las primeras instalaciones. Galera no usa solo el puerto de MySQL.

```bash
# En los tres nodos
sudo firewall-cmd --permanent --add-port=3306/tcp   # MySQL estándar
sudo firewall-cmd --permanent --add-port=4567/tcp   # Comunicación cluster Galera
sudo firewall-cmd --permanent --add-port=4567/udp   # Replicación multicast (opcional)
sudo firewall-cmd --permanent --add-port=4568/tcp   # IST (Incremental State Transfer)
sudo firewall-cmd --permanent --add-port=4444/tcp   # SST (State Snapshot Transfer)
sudo firewall-cmd --reload
```

Si SELinux está activo (y debería estarlo), también necesitas las políticas:

```bash
sudo setsebool -P mysql_connect_any 1
```

Cuatro puertos. Cuatro. Ni uno más, ni uno menos. Si olvidas uno, el cluster se forma pero no sincroniza — y el debug se convierte en un ejercicio de frustración.

---

## La migración de datos y el bootstrap

Antes de iniciar el cluster, migré los datos del servidor standalone al Nodo 1 con un dump completo:

```bash
# En el antiguo servidor standalone
mysqldump --all-databases --single-transaction --routines --triggers \
    --events > /tmp/full_dump.sql

# Transferencia al Nodo 1
scp /tmp/full_dump.sql db-node1:/tmp/
```

Luego el bootstrap — el momento de la verdad. El primer nodo no se inicia con `systemctl start mariadb`. Se inicia con el comando de bootstrap.

**Solo en el Nodo 1:**

```bash
sudo galera_new_cluster
```

Este comando inicia MariaDB con `wsrep_cluster_address=gcomm://` (vacío), que significa: "Yo soy el fundador, no busco otros nodos."

Importación de datos y creación del usuario SST:

```sql
-- Importar el dump del antiguo servidor
SOURCE /tmp/full_dump.sql;

-- Crear el usuario para la transferencia de datos entre nodos
CREATE USER 'sst_user'@'localhost' IDENTIFIED BY 'password_segura_aqui';
GRANT RELOAD, PROCESS, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'sst_user'@'localhost';
FLUSH PRIVILEGES;
```

Verificación inmediata:

```sql
SHOW STATUS LIKE 'wsrep_cluster_size';
-- +--------------------+-------+
-- | Variable_name      | Value |
-- +--------------------+-------+
-- | wsrep_cluster_size | 1     |
-- +--------------------+-------+
```

Si el valor es 1, el bootstrap funcionó. Ahora, **en los otros dos nodos:**

```bash
sudo systemctl start mariadb
```

Estos nodos leen `wsrep_cluster_address`, encuentran al Nodo 1, reciben un SST completo con todos los datos y se unen al cluster.

Después de iniciar los tres:

```sql
SHOW STATUS LIKE 'wsrep_cluster_size';
-- +--------------------+-------+
-- | Variable_name      | Value |
-- +--------------------+-------+
-- | wsrep_cluster_size | 3     |
-- +--------------------+-------+
```

Tres. Ese es el número mágico. El momento en que el cliente dejó de tener un single point of failure.

---

## Verificar el estado de salud del cluster

Esta es la parte que realmente importa a quien gestiona el cluster día a día. No basta con saber que `wsrep_cluster_size` es 3. Hay que saber leer el estado completo.

### La query diagnóstica que uso siempre

```sql
SHOW STATUS WHERE Variable_name IN (
    'wsrep_cluster_size',
    'wsrep_cluster_status',
    'wsrep_connected',
    'wsrep_ready',
    'wsrep_local_state_comment',
    'wsrep_incoming_addresses',
    'wsrep_evs_state',
    'wsrep_flow_control_paused',
    'wsrep_local_recv_queue_avg',
    'wsrep_local_send_queue_avg',
    'wsrep_cert_deps_distance'
);
```

### Cómo interpretar los resultados

| Variable | Valor sano | Significado |
|----------|------------|-------------|
| `wsrep_cluster_size` | `3` | Todos los nodos están en el cluster |
| `wsrep_cluster_status` | `Primary` | El cluster está operativo y tiene quorum |
| `wsrep_connected` | `ON` | Este nodo está conectado al cluster |
| `wsrep_ready` | `ON` | Este nodo acepta queries |
| `wsrep_local_state_comment` | `Synced` | Este nodo está sincronizado |
| `wsrep_flow_control_paused` | `0.0` | Sin pausas por flow control |
| `wsrep_local_recv_queue_avg` | `< 0.5` | La cola de recepción está bajo control |
| `wsrep_local_send_queue_avg` | `< 0.5` | La cola de envío está bajo control |

### Las señales de peligro

**`wsrep_cluster_status = Non-Primary`**: el nodo ha perdido el quorum. Está aislado. No aceptará escrituras (y no debe). Esto sucede cuando un nodo pierde la conexión con la mayoría del cluster.

**`wsrep_flow_control_paused > 0.0`**: flow control activado. Significa que un nodo es demasiado lento aplicando transacciones y está pidiendo a los demás que frenen. Un valor cercano a 1.0 significa que el cluster está esencialmente detenido, esperando al nodo más lento.

**`wsrep_local_recv_queue_avg > 1.0`**: las transacciones entrantes se acumulan. Podría ser un problema de I/O de disco, CPU, o un nodo subdimensionado.

### Script de monitorización

También entregué al cliente un script para su sistema de monitorización (Zabbix, en su caso):

```bash
#!/bin/bash
# galera_health_check.sh — ejecutar en cada nodo

MYSQL="mysql -u monitor -p'monitor_pwd' -Bse"

CLUSTER_SIZE=$($MYSQL "SHOW STATUS LIKE 'wsrep_cluster_size'" | awk '{print $2}')
CLUSTER_STATUS=$($MYSQL "SHOW STATUS LIKE 'wsrep_cluster_status'" | awk '{print $2}')
NODE_STATE=$($MYSQL "SHOW STATUS LIKE 'wsrep_local_state_comment'" | awk '{print $2}')
FLOW_CONTROL=$($MYSQL "SHOW STATUS LIKE 'wsrep_flow_control_paused'" | awk '{print $2}')

if [ "$CLUSTER_SIZE" -lt 3 ] || [ "$CLUSTER_STATUS" != "Primary" ] || [ "$NODE_STATE" != "Synced" ]; then
    echo "CRITICAL: Galera cluster degradado"
    echo "  Size: $CLUSTER_SIZE | Status: $CLUSTER_STATUS | State: $NODE_STATE | FC: $FLOW_CONTROL"
    exit 2
fi

echo "OK: Galera cluster sano (3 nodos, Primary, Synced)"
exit 0
```

---

## El problema del split-brain: por qué tres nodos y no dos

Cuando presenté la solución al cliente, la primera pregunta fue: "¿De verdad necesitamos tres servidores? ¿No bastan dos?"

No. Y no es una cuestión de coste — es una cuestión de matemáticas.

Galera usa un algoritmo de consenso basado en **quorum**. Con tres nodos, el quorum es 2: si un nodo cae, los otros dos reconocen que son la mayoría y siguen operando. Con dos nodos, el quorum es 2: si un nodo cae, el que queda **no tiene quorum** y se bloquea para evitar el split-brain.

Existe el parámetro `pc.ignore_quorum` para forzar a un nodo a operar sin quorum, pero es como desactivar la alarma de incendios porque suena demasiado a menudo.

**Tres nodos es el mínimo para un cluster Galera en producción.** El tercer nodo no es un lujo — es lo que permite al cluster seguir funcionando cuando las cosas van mal.

---

## Cuando un nodo cae y vuelve

Una de las primeras cosas que hice después de la puesta en producción fue simular un fallo — con el cliente mirando.

Apagué el Nodo 3. La aplicación siguió funcionando sin interrupción en los nodos 1 y 2. Sin errores, sin timeouts. Doscientos usuarios que no se enteraron de nada.

Luego reinicié el Nodo 3. Qué pasó:

1. El nodo arrancó y contactó a los demás a través de `wsrep_cluster_address`
2. El gap de transacciones era pequeño, así que recibió un **IST** (Incremental State Transfer) — solo las transacciones faltantes
3. En menos de un minuto estaba en `Synced` de nuevo

Si el nodo hubiera estado caído más tiempo y el gcache se hubiera superado, habría recibido un **SST** completo — el dataset entero. Por eso el parámetro `gcache.size` es importante:

```ini
wsrep_provider_options="gcache.size=512M"
```

Un gcache más grande significa que el cluster puede tolerar tiempos de inactividad más largos de un nodo sin necesitar un SST completo. En el caso del cliente, con unos 80-100 MB de transacciones al día, un gcache de 512 MB cubría casi una semana de ausencia.

El cliente vio al Nodo 3 volver a sincronizarse y dijo: "Entonces, ¿la próxima vez que necesitemos hacer mantenimiento en un servidor no tenemos que parar todo?" Exacto. Ese era el punto.

---

## Checklist de puesta en producción

Antes de declarar el cluster listo al cliente, verifiqué cada punto:

- [ ] Misma versión de MariaDB en todos los nodos
- [ ] `wsrep_cluster_size` = 3
- [ ] `wsrep_cluster_status` = Primary en todos los nodos
- [ ] `wsrep_local_state_comment` = Synced en todos los nodos
- [ ] Test de escritura en Nodo 1, verificación de lectura en Nodo 2 y 3
- [ ] Test de apagado de un nodo: el cluster sigue funcionando
- [ ] Test de rejoin: el nodo vuelve a Synced sin SST completo
- [ ] Usuario SST configurado y funcionando
- [ ] Firewall verificado en todos los puertos (3306, 4567, 4568, 4444)
- [ ] Monitorización activa sobre `wsrep_cluster_status` y `wsrep_flow_control_paused`
- [ ] Backup configurado (en UN nodo, no en los tres)
- [ ] Aplicación reconfigurada para apuntar al load balancer o al VIP

---

## Seis meses después

Contacté con el cliente seis meses después de la puesta en producción. Mientras tanto habían tenido dos reinicios programados por actualizaciones de sistema y un fallo imprevisto de disco en uno de los nodos. En los tres casos, la aplicación nunca dejó de funcionar. Cero tiempo de inactividad no planificado.

Lo que más me impactó fue su frase: "Antes vivíamos con la ansiedad de que la base de datos cayera. Ahora ya no pensamos en eso."

Ese es el verdadero valor de un cluster Galera bien configurado. No es la tecnología en sí — es la tranquilidad que trae. La certeza de que un único fallo ya no para la empresa.

La parte técnica es la más sencilla. Lo que marca la diferencia es entender **por qué** cada parámetro está configurado de cierta manera, qué pasa cuando las cosas van mal, y cómo diagnosticar los problemas antes de que se conviertan en emergencias. Un cluster que funciona en demo y uno que aguanta en producción: la distancia entre los dos está toda en los detalles que he contado aquí.
