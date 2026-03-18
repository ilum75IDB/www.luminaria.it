---
title: "MySQL Multi-Instance: A Ticket, a CSV and the secure-file-priv Wall"
description: "An operation that should have taken five minutes — exporting a CSV from MySQL — turns into an investigation across multiple instances on the same server, Unix sockets, different ports and the secure-file-priv directive that blocks everything. From finding the right instance to exporting from the shell."
date: "2025-11-04T10:00:00+01:00"
draft: false
translationKey: "mysql_multi_istanza_secure_file_priv"
tags: ["multi-instance", "secure-file-priv", "csv-export", "systemd", "socket", "troubleshooting"]
categories: ["mysql"]
image: "mysql-multi-istanza-secure-file-priv.cover.jpg"
---

The ticket said: "We need a CSV export from the orders table in the ERP database. By 2 PM."

It was 11 AM. Three hours for a SELECT with INTO OUTFILE — a five-minute job, I thought. Then I opened the VPN, connected to the server and realized five minutes were not going to cut it.

The server was a CentOS 7 box running four MySQL instances. Four. On the same host, with four different systemd services, four different ports, four different Unix sockets, four different data directories. A setup someone had put together years earlier — probably to save on a second server — and that no one had touched or documented since.

The first problem was not the query. The first problem was: which of the four instances do I need to connect to?

---

## The Environment: Four MySQL Instances, One Server

Multi-instance MySQL environments are not as rare as you might think. I run into them more often than I would like, especially in small and mid-sized companies where servers are scarce and applications are plenty. The logic is simple: instead of buying four servers, you buy one beefy machine and run four MySQL instances on it, each with its own database, its own port, its own configuration file.

It works, until you need to do maintenance. And maintenance on a multi-instance setup with no documentation is an exercise in IT archaeology.

On that server, the situation looked like this:

```bash
systemctl list-units --type=service | grep mysql
    mysqld.service          loaded active running  MySQL Server (porta 3306)
    mysqld-app2.service     loaded active running  MySQL Server (porta 3307)
    mysqld-reporting.service loaded active running  MySQL Server (porta 3308)
    mysqld-legacy.service   loaded active running  MySQL Server (porta 3309)
```

Four services. The names were vaguely descriptive — "app2", "reporting", "legacy" — but the ticket mentioned the "ERP" without specifying which instance hosted that database. No internal wiki, no README file on the server, no comments in the configuration files.

---

## Finding the Right Instance

The first step was figuring out which instance held the orders database. The technique is always the same: start from the systemd service, trace back to the configuration file, read the port and socket from there.

```bash
systemctl cat mysqld-app2.service | grep ExecStart
    ExecStart=/usr/sbin/mysqld --defaults-file=/etc/mysql/app2.cnf
```

Each service pointed to a different `my.cnf`. I checked all four:

```bash
grep -E "^(port|socket|datadir)" /etc/mysql/app2.cnf
    port      = 3307
    socket    = /var/run/mysqld/mysqld-app2.sock
    datadir   = /data/mysql-app2
```

For each instance I noted down the port, socket and datadir. Then I did a quick round:

```bash
mysql --socket=/var/run/mysqld/mysqld.sock -u root -p -e "SHOW DATABASES;" 2>/dev/null
mysql --socket=/var/run/mysqld/mysqld-app2.sock -u root -p -e "SHOW DATABASES;" 2>/dev/null
mysql --socket=/var/run/mysqld/mysqld-reporting.sock -u root -p -e "SHOW DATABASES;" 2>/dev/null
mysql --socket=/var/run/mysqld/mysqld-legacy.sock -u root -p -e "SHOW DATABASES;" 2>/dev/null
```

The `gestionale_prod` database was on the second instance — the one on port 3307 with socket `/var/run/mysqld/mysqld-app2.sock`.

One detail that seems trivial but makes all the difference in a multi-instance environment: when you connect to MySQL specifying only `-h localhost`, the client does not use TCP. It uses the default Unix socket, which almost always belongs to the primary instance on port 3306. If the database you are looking for lives on a different instance, you connect to the wrong one without even realizing it.

---

## Connecting and Verifying

Once I had identified the instance, I connected specifying the socket explicitly:

```bash
mysql --socket=/var/run/mysqld/mysqld-app2.sock -u root -p
```

First thing after login: verify you are on the right instance.

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

Port 3307, database present, orders table right where it should be. The connection was correct.

The port check may look like paranoia, but it is not. In an environment with four instances, mixing up which socket points to which port is easier than you think. And you only discover the mistake when the data you export is not what you expected — or worse, when you make a change thinking you are on the test database and find out you were on production.

---

## The First Attempt: INTO OUTFILE

The query was straightforward. The requester wanted the last quarter's orders with amount, customer and date:

```sql
SELECT o.id_ordine, o.data_ordine, c.ragione_sociale, o.importo_totale
FROM ordini o
JOIN clienti c ON o.id_cliente = c.id_cliente
WHERE o.data_ordine >= '2025-07-01'
ORDER BY o.data_ordine;
```

My first instinct was to use `INTO OUTFILE`, MySQL's native way of writing results to a file:

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

MySQL's response was blunt:

```
ERROR 1290 (HY000): The MySQL server is running with the
--secure-file-priv option so it cannot execute this statement
```

There it was. The wall.

---

## secure-file-priv: The Directive That Blocks Everything (And Rightly So)

The `secure_file_priv` variable is how MySQL restricts file read and write operations. It controls where `LOAD DATA INFILE`, `SELECT INTO OUTFILE` and the `LOAD_FILE()` function are allowed to operate.

```sql
SHOW VARIABLES LIKE 'secure_file_priv';
+------------------+------------------------+
| Variable_name    | Value                  |
+------------------+------------------------+
| secure_file_priv | /var/lib/mysql-files/  |
+------------------+------------------------+
```

This variable has three modes:

1. **A specific path** (e.g. `/var/lib/mysql-files/`): file operations work, but only within that directory
2. **Empty string** (`""`): no restrictions — MySQL can read and write anywhere the system user has permissions
3. **NULL**: file operations are completely disabled

My instance was configured with a specific path. The attempt to write to `/tmp/` was blocked because `/tmp/` is not `/var/lib/mysql-files/`.

The first reaction — one I see many people have — would be: "let's change secure-file-priv to an empty string in my.cnf and restart." No. Absolutely not. On a production server with four MySQL instances, restarting an instance at 11:30 in the morning for a CSV export is not an option. And disabling a security protection is never the right answer, not even in an emergency.

The obvious alternative was to write the file to the authorized directory:

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

But there was another problem. The `/var/lib/mysql-files/` directory belonged to the primary instance (port 3306). The instance on port 3307 had its own separate datadir under `/data/mysql-app2/`, and its `secure_file_priv` pointed to `/data/mysql-app2/files/` — a directory that did not exist and that nobody had ever created.

I could have created the directory, assigned the correct permissions to the `mysql` user and written there. But at that point I was already losing time. And there is a cleaner way.

---

## The Solution: Shell Export with the mysql Client

When `INTO OUTFILE` is blocked or inconvenient, the most practical solution is to bypass MySQL's file-writing mechanism entirely and use the command-line client to redirect the output.

The trick is in the `-B` (batch mode) and `-e` (execute) options:

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

The `-B` option produces tab-separated output without the ASCII table borders. The result is a clean TSV file that opens without issues in any spreadsheet application.

If you need an actual CSV with commas as separators, just pipe through `sed`:

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

The `-N` option removes the header row with column names. If you want it, drop the flag.

The file was ready in under a minute. 12,400 rows, 1.2 MB. I copied it to my machine with `scp`, checked it opened correctly in LibreOffice Calc and sent it to the requester. It was 11:45. The ticket that was supposed to take five minutes had taken forty-five — but at least I had not restarted any instances.

---

## Why You Should Not Disable secure-file-priv

The temptation to set `secure_file_priv = ""` is strong, especially on development servers or on machines where "it's just us anyway." The problem is that protection exists for a very specific reason.

Without `secure_file_priv`, a MySQL user with the `FILE` privilege can:

- Read any file readable by the `mysql` system user — including `/etc/passwd`, configuration files, SSH keys if permissions are not locked down
- Write files anywhere the `mysql` user has write access — including the webroot of an Apache or Nginx running on the same server

In a SQL injection scenario, the `FILE` privilege combined with an empty `secure_file_priv` is an open door. The attacker can read system files, write web shells, escalate privileges. This is not theory — it is one of the most well-documented attack vectors in penetration tests against web applications backed by MySQL.

The rule is simple: configure `secure_file_priv` with a specific path, create the necessary directories for each instance at setup time, and leave them there. If you need to do occasional exports, the mysql command-line client does the same job without touching the security configuration.

---

## Lessons from a Five-Minute Ticket

That ticket reminded me of three things that in thirty years of working with databases I have seen confirmed hundreds of times.

The first: **in a multi-instance environment, the first step is always identifying the instance**. It sounds obvious, but the number of mistakes that come from connecting to the wrong instance — thinking you are somewhere else — is staggering. A `SHOW VARIABLES LIKE 'port'` after every connection is not paranoia, it is operational hygiene.

The second: **secure-file-priv is not an obstacle, it is a safeguard**. When it blocks you, that is not the moment to disable it. That is the moment to use an alternative path or an alternative method. The directive exists because MySQL in the hands of a user with the FILE privilege and no filesystem restrictions is a real risk.

The third: **the mysql command-line client is more powerful than most DBAs give it credit for**. With `-B`, `-N`, `-e` and a pipe to `sed` or `awk`, you can do exports, transformations and automations without ever touching `INTO OUTFILE`. Less elegant, maybe. But it always works, requires no special permissions and does not need someone to have created the right directory six months earlier.

The CSV arrived at 11:45. The requester never knew that behind five columns and 12,400 rows there were forty-five minutes of system archaeology. But that is how tickets work: the person who opens them sees the result, the person who resolves them sees the journey.
