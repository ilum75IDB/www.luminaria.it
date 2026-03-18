---
title: "MySQL multi-instancia: un ticket, un CSV y el muro de secure-file-priv"
description: "Una operación que debía durar cinco minutos — extraer un CSV de MySQL — se convierte en una investigación entre instancias múltiples en el mismo servidor, sockets Unix, puertos diferentes y la directiva secure-file-priv que bloquea todo. Desde la conexión a la instancia correcta hasta el export desde shell."
date: "2025-11-04T10:00:00+01:00"
draft: false
translationKey: "mysql_multi_istanza_secure_file_priv"
tags: ["multi-instance", "secure-file-priv", "csv-export", "systemd", "socket", "troubleshooting"]
categories: ["mysql"]
image: "mysql-multi-istanza-secure-file-priv.cover.jpg"
---

El ticket decía: "Necesitamos un export CSV de la tabla de pedidos del gestional. Antes de las 14:00."

Eran las 11. Tres horas para un SELECT con INTO OUTFILE — cosa de cinco minutos, pensé. Después abrí la VPN, me conecté al servidor y entendí que cinco minutos no iban a alcanzar.

El servidor era una máquina CentOS 7 con cuatro instancias MySQL. Cuatro. En el mismo host, con cuatro servicios {{< glossary term="systemd" >}}systemd{{< /glossary >}} distintos, cuatro puertos distintos, cuatro sockets Unix distintos, cuatro directorios de datos distintos. Un setup que alguien había montado años atrás — probablemente para ahorrarse un segundo servidor — y que desde entonces nadie había tocado ni documentado.

El primer problema no era la query. El primer problema era: ¿a cuál de las cuatro instancias tengo que conectarme?

---

## El entorno: cuatro MySQL, un solo servidor

Los entornos multi-instancia en MySQL no son tan raros como se podría pensar. Los encuentro con más frecuencia de la que quisiera, sobre todo en empresas medianas y pequeñas donde los servidores son pocos y las aplicaciones son muchas. La lógica es simple: en vez de comprar cuatro servidores, compras uno potente y haces correr cuatro instancias MySQL, cada una con su base de datos, su puerto, su archivo de configuración.

El resultado funciona, hasta que necesitas hacer mantenimiento. Y el mantenimiento en un multi-instancia, sin documentación, es un ejercicio de arqueología informática.

En ese servidor, la situación era esta:

```bash
systemctl list-units --type=service | grep mysql
    mysqld.service          loaded active running  MySQL Server (porta 3306)
    mysqld-app2.service     loaded active running  MySQL Server (porta 3307)
    mysqld-reporting.service loaded active running  MySQL Server (porta 3308)
    mysqld-legacy.service   loaded active running  MySQL Server (porta 3309)
```

Cuatro servicios. Los nombres eran vagamente indicativos — "app2", "reporting", "legacy" — pero el ticket hablaba del "gestional" sin especificar qué instancia alojaba esa base de datos. Ninguna wiki interna, ningún README en el servidor, ningún comentario en los archivos de configuración.

---

## Encontrar la instancia correcta

El primer paso fue entender qué instancia contenía la base de datos de pedidos. La técnica es siempre la misma: partes del servicio systemd, subes al archivo de configuración, de ahí lees puerto y socket.

```bash
systemctl cat mysqld-app2.service | grep ExecStart
    ExecStart=/usr/sbin/mysqld --defaults-file=/etc/mysql/app2.cnf
```

Cada servicio apuntaba a un `my.cnf` diferente. Revisé los cuatro:

```bash
grep -E "^(port|socket|datadir)" /etc/mysql/app2.cnf
    port      = 3307
    socket    = /var/run/mysqld/mysqld-app2.sock
    datadir   = /data/mysql-app2
```

Para cada instancia, anoté puerto, socket y datadir. Después hice la ronda rápida:

```bash
mysql --socket=/var/run/mysqld/mysqld.sock -u root -p -e "SHOW DATABASES;" 2>/dev/null
mysql --socket=/var/run/mysqld/mysqld-app2.sock -u root -p -e "SHOW DATABASES;" 2>/dev/null
mysql --socket=/var/run/mysqld/mysqld-reporting.sock -u root -p -e "SHOW DATABASES;" 2>/dev/null
mysql --socket=/var/run/mysqld/mysqld-legacy.sock -u root -p -e "SHOW DATABASES;" 2>/dev/null
```

La base de datos `gestionale_prod` estaba en la segunda instancia — la del puerto 3307 con socket `/var/run/mysqld/mysqld-app2.sock`.

Un detalle que parece trivial pero que en un entorno multi-instancia marca la diferencia: cuando te conectas a MySQL especificando solo `-h localhost`, el cliente no usa TCP. Usa el {{< glossary term="unix-socket" >}}socket Unix{{< /glossary >}} por defecto, que casi siempre es el de la instancia primaria en el puerto 3306. Si la base de datos que buscas está en otra instancia, te conectas a la equivocada sin siquiera darte cuenta.

---

## La conexión y la verificación

Una vez identificada la instancia, me conecté especificando explícitamente el socket:

```bash
mysql --socket=/var/run/mysqld/mysqld-app2.sock -u root -p
```

Primera cosa después del login: verificar que estaba en la instancia correcta.

```sql
SHOW VARIABLES LIKE 'port';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| port          | 3307  |
+---------------+-------+

SELECT DATABASE();

USE gestionale_prod;

SHOW TABLES LIKE '%ordini%';
+----------------------------------+
| Tables_in_gestionale_prod        |
+----------------------------------+
| ordini                           |
| ordini_dettaglio                 |
| ordini_storico                   |
+----------------------------------+
```

Puerto 3307, base de datos presente, tabla de pedidos en su lugar. La conexión era la correcta.

El check del puerto parece paranoia, pero no lo es. En un entorno con cuatro instancias, confundir qué socket apunta a qué puerto es más fácil de lo que parece. Y el error lo descubres solo cuando los datos que exportas no son los que esperabas — o peor, cuando haces una modificación pensando que estás en la base de datos de pruebas y descubres que estabas en producción.

---

## El primer intento: INTO OUTFILE

La query era sencilla. El solicitante quería los pedidos del último trimestre con importe, cliente y fecha:

```sql
SELECT o.id_ordine, o.data_ordine, c.ragione_sociale, o.importo_totale
FROM ordini o
JOIN clienti c ON o.id_cliente = c.id_cliente
WHERE o.data_ordine >= '2025-07-01'
ORDER BY o.data_ordine;
```

El primer instinto fue usar {{< glossary term="into-outfile" >}}`INTO OUTFILE`{{< /glossary >}}, la manera nativa de MySQL para escribir resultados en archivo:

```sql
SELECT o.id_ordine, o.data_ordine, c.ragione_sociale, o.importo_totale
FROM ordini o
JOIN clienti c ON o.id_cliente = c.id_cliente
WHERE o.data_ordine >= '2025-07-01'
ORDER BY o.data_ordine
INTO OUTFILE '/tmp/export_ordini.csv'
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n';
```

La respuesta de MySQL fue seca:

```
ERROR 1290 (HY000): The MySQL server is running with the
--secure-file-priv option so it cannot execute this statement
```

Ahí estaba el muro.

---

## {{< glossary term="secure-file-priv" >}}secure-file-priv{{< /glossary >}}: la directiva que bloquea todo (y hace bien)

La variable `secure_file_priv` es la forma en que MySQL limita las operaciones de lectura y escritura sobre archivos. Controla dónde `LOAD DATA INFILE`, `SELECT INTO OUTFILE` y la función `LOAD_FILE()` pueden operar.

```sql
SHOW VARIABLES LIKE 'secure_file_priv';
+------------------+------------------------+
| Variable_name    | Value                  |
+------------------+------------------------+
| secure_file_priv | /var/lib/mysql-files/  |
+------------------+------------------------+
```

Esta variable tiene tres modalidades:

1. **Una ruta específica** (ej. `/var/lib/mysql-files/`): las operaciones sobre archivos funcionan, pero solo dentro de ese directorio
2. **Cadena vacía** (`""`): sin restricciones — MySQL puede leer y escribir donde sea que su usuario de sistema tenga permisos
3. **NULL**: las operaciones sobre archivos están completamente deshabilitadas

Mi instancia estaba configurada con una ruta específica. El intento de escribir en `/tmp/` fue bloqueado porque `/tmp/` no es `/var/lib/mysql-files/`.

La primera reacción — la que veo hacer a muchos — habría sido: "cambiamos secure-file-priv a cadena vacía en el my.cnf y reiniciamos". No. Absolutamente no. En un servidor de producción con cuatro instancias MySQL, reiniciar una instancia a las 11:30 de la mañana por un export CSV no es una opción. Y deshabilitar una protección de seguridad nunca es la respuesta correcta, ni siquiera en emergencia.

La alternativa obvia era escribir el archivo en el directorio autorizado:

```sql
SELECT o.id_ordine, o.data_ordine, c.ragione_sociale, o.importo_totale
FROM ordini o
JOIN clienti c ON o.id_cliente = c.id_cliente
WHERE o.data_ordine >= '2025-07-01'
ORDER BY o.data_ordine
INTO OUTFILE '/var/lib/mysql-files/export_ordini.csv'
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n';
```

Pero había otro problema. El directorio `/var/lib/mysql-files/` era el de la instancia primaria (puerto 3306). La instancia en el puerto 3307 tenía su datadir separado en `/data/mysql-app2/`, y su `secure_file_priv` apuntaba a `/data/mysql-app2/files/` — un directorio que no existía y que nadie había creado jamás.

Podría haber creado el directorio, asignado los permisos correctos al usuario `mysql` y escrito ahí. Pero a esas alturas ya estaba perdiendo tiempo. Y hay una forma más limpia.

---

## La solución: export desde shell con el cliente mysql

Cuando `INTO OUTFILE` está bloqueado o resulta incómodo, la solución más práctica es saltarse completamente el mecanismo de escritura de archivos de MySQL y usar el cliente desde línea de comandos para redirigir la salida.

El truco está en las opciones `-B` (batch mode) y `-e` (execute):

```bash
mysql --socket=/var/run/mysqld/mysqld-app2.sock \
      -u root -p \
      -B -e "
SELECT o.id_ordine, o.data_ordine, c.ragione_sociale, o.importo_totale
FROM ordini o
JOIN clienti c ON o.id_cliente = c.id_cliente
WHERE o.data_ordine >= '2025-07-01'
ORDER BY o.data_ordine
" gestionale_prod > /tmp/export_ordini.tsv
```

La opción `-B` produce una salida separada por tabulaciones sin los bordes ASCII de las tablas. El resultado es un archivo TSV limpio que se abre sin problemas en cualquier hoja de cálculo.

Si necesitas un verdadero CSV con comas como separador, basta un paso con `sed`:

```bash
mysql --socket=/var/run/mysqld/mysqld-app2.sock \
      -u root -p \
      -B -N -e "
SELECT o.id_ordine, o.data_ordine, c.ragione_sociale, o.importo_totale
FROM ordini o
JOIN clienti c ON o.id_cliente = c.id_cliente
WHERE o.data_ordine >= '2025-07-01'
ORDER BY o.data_ordine
" gestionale_prod | sed 's/\t/,/g' > /tmp/export_ordini.csv
```

La opción `-N` elimina la fila de encabezado con los nombres de las columnas. Si la quieres, quita el flag.

El archivo estaba listo en menos de un minuto. 12.400 filas, 1,2 MB. Lo copié a mi máquina con `scp`, verifiqué que abriera bien en LibreOffice Calc y lo envié al solicitante. Eran las 11:45. El ticket que debía durar cinco minutos había requerido cuarenta y cinco — pero al menos no había reiniciado ninguna instancia.

---

## Por qué no deshabilitar secure-file-priv

La tentación de configurar `secure_file_priv = ""` es fuerte, sobre todo en servidores de desarrollo o en máquinas donde "total, solo somos nosotros". El problema es que esa protección existe por una razón concreta.

Sin `secure_file_priv`, un usuario MySQL con el privilegio `FILE` puede:

- Leer cualquier archivo legible por el usuario de sistema `mysql` — incluyendo `/etc/passwd`, archivos de configuración, llaves SSH si los permisos no están blindados
- Escribir archivos donde sea que el usuario `mysql` tenga permisos de escritura — incluyendo el webroot de un eventual Apache o Nginx en el mismo servidor

En un contexto de {{< glossary term="sql-injection" >}}SQL injection{{< /glossary >}}, el privilegio `FILE` combinado con un `secure_file_priv` vacío es una puerta abierta. El atacante puede leer archivos del sistema, escribir web shells, hacer escalación de privilegios. No es teoría — es uno de los vectores de ataque más documentados en penetration tests sobre aplicaciones web con MySQL detrás.

La regla es simple: `secure_file_priv` se configura con una ruta específica, se crean los directorios necesarios para cada instancia al momento del setup, y se dejan ahí. Si necesitas hacer exports ocasionales, el cliente mysql desde shell hace el mismo trabajo sin tocar la configuración de seguridad.

---

## Lecciones de un ticket de cinco minutos

Ese ticket me recordó tres cosas que en treinta años de trabajo con bases de datos he visto confirmadas cientos de veces.

La primera: **en un entorno multi-instancia, el primer paso es siempre identificar la instancia**. Parece obvio, pero la cantidad de errores que nacen de conectarse a la instancia equivocada — pensando que estás en otra — es impresionante. Un `SHOW VARIABLES LIKE 'port'` después de cada conexión no es paranoia, es higiene operativa.

La segunda: **secure-file-priv no es un obstáculo, es una protección**. Cuando te bloquea, no es el momento de deshabilitarla. Es el momento de usar una ruta alternativa o un método alternativo. La directiva existe porque MySQL en manos de un usuario con el privilegio FILE y sin restricciones sobre el filesystem es un riesgo concreto.

La tercera: **el cliente mysql desde línea de comandos es más poderoso de lo que la mayoría de los DBA le reconocen**. Con `-B`, `-N`, `-e` y un pipe hacia `sed` o `awk`, puedes hacer exports, transformaciones y automatizaciones sin tocar jamás `INTO OUTFILE`. Es menos elegante, quizás. Pero funciona siempre, no requiere permisos especiales y no necesita que alguien haya creado el directorio correcto seis meses antes.

El CSV llegó a las 11:45. El solicitante nunca supo que detrás de cinco columnas y 12.400 filas había cuarenta y cinco minutos de arqueología de sistemas. Pero así funcionan los tickets: quien los abre ve el resultado, quien los resuelve ve el camino.

------------------------------------------------------------------------

## Glosario

**[secure-file-priv](/es/glossary/secure-file-priv/)** — Directiva de seguridad MySQL que limita los directorios donde el servidor puede leer y escribir archivos mediante `INTO OUTFILE`, `LOAD DATA INFILE` y `LOAD_FILE()`.

**[Unix Socket](/es/glossary/unix-socket/)** — Mecanismo de comunicación local entre procesos en sistemas Linux, usado por MySQL como método de conexión predeterminado al conectarse a `localhost`.

**[INTO OUTFILE](/es/glossary/into-outfile/)** — Cláusula SQL de MySQL para exportar resultados de queries directamente a un archivo en el filesystem del servidor. Sujeta a las restricciones de `secure-file-priv`.

**[systemd](/es/glossary/systemd/)** — Gestor de servicios en Linux moderno, usado para gestionar múltiples instancias MySQL en el mismo servidor mediante unit files separados.

**[SQL Injection](/es/glossary/sql-injection/)** — Técnica de ataque que inserta código SQL malicioso en los inputs de una aplicación. La directiva `secure-file-priv` contribuye a mitigar su impacto.
