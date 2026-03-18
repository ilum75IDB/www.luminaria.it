---
title: "Binary log in MySQL: cosa sono, come gestirli e quando puoi cancellarli"
description: "Un server MySQL con il disco al 95%, 180 GB di binary log accumulati in sei mesi. Da lì parte un viaggio nei binlog: cosa contengono, perché esistono, come funzionano con la replica e il point-in-time recovery, e soprattutto come gestirli senza fare danni."
date: "2026-03-31T10:00:00+01:00"
draft: false
translationKey: "binary_log_mysql"
tags: ["binlog", "replication", "disk-space", "recovery", "mariadb"]
categories: ["mysql"]
image: "binary-log-mysql.cover.jpg"
---

Il messaggio nel canale Slack del team infrastruttura era di quelli che fanno alzare la testa dallo schermo: "Disco al 95% sul db di produzione. Chi può guardare?"

Il server era un MySQL 8.0 su Rocky Linux, un gestionale usato da un centinaio di utenti. Il database in sé occupava circa 40 GB — niente di straordinario. Ma nella directory dei dati c'erano 180 GB di binary log. Sei mesi di binlog che nessuno aveva mai pensato di gestire.

Non è la prima volta che vedo questa scena. Anzi, direi che è uno dei pattern più ricorrenti nei ticket che mi arrivano. Il binary log è una di quelle funzionalità di MySQL che funzionano in silenzio, senza chiedere nulla — finché il disco non si riempie.

---

## Cosa sono i binary log, in pratica

Il {{< glossary term="binary-log" >}}binary log{{< /glossary >}} è un registro sequenziale di tutti gli eventi che modificano i dati nel database. Ogni INSERT, UPDATE, DELETE, ogni DDL — tutto viene scritto in file binari numerati progressivamente: `mysql-bin.000001`, `mysql-bin.000002` e così via.

Il nome inganna un po'. Non è un "log" nel senso del syslog o dell'error log — non è fatto per essere letto da un umano. È un flusso binario strutturato che MySQL usa internamente per due scopi fondamentali:

1. **Replica**: lo slave legge i binlog del master per replicare le stesse operazioni
2. **{{< glossary term="pitr" >}}Point-in-time recovery (PITR){{< /glossary >}}**: dopo un restore da backup, puoi "riapplicare" i binlog per portare i dati fino a un momento preciso

Senza il binary log, non puoi fare né l'una né l'altra cosa. Questa è la ragione per cui il primo istinto — "disabilitiamo i binlog così non riempiono il disco" — è quasi sempre sbagliato.

---

## Come MySQL genera i binlog

Il binary logging si attiva con il parametro `log_bin`. Da MySQL 8.0 è abilitato di default — un cambio importante rispetto alle versioni precedenti dove bisognava attivarlo esplicitamente.

```ini
[mysqld]
log_bin = /var/lib/mysql/mysql-bin
server-id = 1
```

MySQL crea un nuovo file binlog in diverse circostanze:

- Quando il server si avvia o si riavvia
- Quando il file corrente raggiunge la dimensione definita da `max_binlog_size` (default: 1 GB)
- Quando esegui `FLUSH BINARY LOGS`
- Quando avviene una rotazione manuale

Ogni file binlog ha un file indice associato (`mysql-bin.index`) che tiene traccia di tutti i file binlog attivi. Questo file è critico: se lo corrompi o lo modifichi a mano, MySQL non sa più quali binlog esistono.

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

Centosettantadue file. Ognuno da circa un gigabyte. Il conto torna: 180 GB di binlog mai purgati.

---

## Il ruolo nella replica

In un'architettura master-slave, il binary log è il meccanismo di trasporto dei dati. Il flusso è questo:

1. Il master scrive ogni transazione nel binlog
2. Lo slave ha un thread (I/O thread) che si connette al master e legge i binlog
3. Lo slave scrive ciò che riceve nel proprio {{< glossary term="relay-log" >}}relay log{{< /glossary >}}
4. Un secondo thread (SQL thread) sullo slave esegue gli eventi dal {{< glossary term="relay-log" >}}relay log{{< /glossary >}}

Questo significa che i binlog sul master **devono restare disponibili finché tutti gli slave non li hanno letti**. Se cancelli un binlog che lo slave non ha ancora consumato, la replica si rompe.

Prima di toccare qualsiasi binlog su un master, il comando da eseguire è:

```sql
SHOW REPLICA STATUS\G
-- oppure, su versioni più vecchie:
SHOW SLAVE STATUS\G
```

Il campo che interessa è `Relay_Master_Log_File` (o `Source_Log_File` nelle versioni recenti): ti dice quale binlog lo slave sta leggendo in quel momento. Tutti i file precedenti a quello sono sicuri da eliminare.

---

## Point-in-time recovery: l'altra ragione per cui i binlog esistono

Il secondo uso — spesso sottovalutato — è il point-in-time recovery. Lo scenario è questo: hai un backup fatto alle 3 di notte. Alle 14:30 qualcuno esegue un `DROP TABLE` sbagliato. Senza binlog, puoi fare restore del backup e perdi tutto ciò che è successo tra le 3:00 e le 14:30. Con i binlog, fai il restore e poi riapplichi i binlog fino alle 14:29.

```bash
# Trovare l'evento del DROP TABLE
mysqlbinlog --start-datetime="2026-03-30 14:00:00" \
            --stop-datetime="2026-03-30 15:00:00" \
            /var/lib/mysql/mysql-bin.000318 | grep -i "DROP"

# Riapplicare i binlog fino al momento prima del disastro
mysqlbinlog --stop-datetime="2026-03-30 14:29:00" \
            /var/lib/mysql/mysql-bin.000310 \
            /var/lib/mysql/mysql-bin.000311 \
            ... \
            /var/lib/mysql/mysql-bin.000318 | mysql -u root -p
```

In pratica, i binlog sono la tua assicurazione. Il backup è la base, i binlog coprono il delta. Cancellare i binlog senza un backup recente è come cancellare la polizza assicurativa il giorno prima di un temporale.

---

## PURGE BINARY LOGS: il modo corretto di fare pulizia

Torniamo al nostro server con il disco al 95%. La tentazione di fare un bel `rm -f mysql-bin.*` è forte. Ma è sbagliata, per due ragioni:

1. MySQL non sa che hai cancellato i file — il file indice punta ancora a binlog che non esistono più
2. Se c'è una replica attiva, rischi di rompere la sincronizzazione

Il modo corretto è il comando `PURGE`:

```sql
-- Eliminare tutti i binlog precedenti a un file specifico
PURGE BINARY LOGS TO 'mysql-bin.000300';

-- Oppure, eliminare tutti i binlog più vecchi di una certa data
PURGE BINARY LOGS BEFORE '2026-03-01 00:00:00';
```

`PURGE` fa tre cose che `rm` non fa:

- Aggiorna il file indice
- Verifica che i file non siano necessari alla replica (in teoria — ma controlla tu prima)
- Rimuove i file in modo ordinato

Nel caso del nostro server, prima ho verificato che non ci fossero slave:

```sql
SHOW REPLICAS;
-- Empty set
```

Nessuna replica. Poi ho verificato quale fosse il binlog corrente:

```sql
SHOW MASTER STATUS;
+------------------+----------+
| File             | Position |
+------------------+----------+
| mysql-bin.000318 | 52428800 |
+------------------+----------+
```

Mantenendo gli ultimi 3 file per sicurezza:

```sql
PURGE BINARY LOGS TO 'mysql-bin.000316';
```

Risultato: 175 GB liberati in pochi secondi. Il disco è sceso dal 95% al 28%.

---

## Configurare la retention automatica

Risolvere l'emergenza è un conto. Fare in modo che non ricapiti è un altro. MySQL offre due parametri per la gestione automatica della retention:

### `expire_logs_days` (legacy)

```ini
[mysqld]
expire_logs_days = 14
```

Elimina automaticamente i binlog più vecchi di 14 giorni. Semplice ma grossolano — la granularità è solo in giorni.

### `binlog_expire_logs_seconds` (MySQL 8.0+)

```ini
[mysqld]
binlog_expire_logs_seconds = 1209600   # 14 giorni in secondi
```

Stessa logica, ma con granularità al secondo. Da MySQL 8.0, questo parametro ha la priorità su `expire_logs_days`. Se li imposti entrambi, vince `binlog_expire_logs_seconds`.

La domanda che mi fanno sempre è: "Quanti giorni di retention?"

Dipende. Ma ecco le mie regole pratiche:

| Scenario | Retention consigliata |
|----------|----------------------|
| Server standalone, backup giornaliero | 7 giorni |
| Master con replica, backup giornaliero | 7-14 giorni |
| Master con replica lenta o in zone diverse | 14-30 giorni |
| Ambienti regolamentati (finanza, sanità) | 30-90 giorni, con archivio |

Il principio è: **la retention dei binlog deve coprire almeno il doppio dell'intervallo tra due backup**. Se fai backup ogni notte, mantieni almeno 2-3 giorni di binlog. Se fai backup settimanali, almeno 14 giorni.

Nel caso del nostro server, non era stata impostata nessuna retention. Il default di MySQL 8.0 è 30 giorni — ma quel valore era stato sovrascritto a 0 (nessuna scadenza) in un `my.cnf` personalizzato da qualcuno che "voleva tenere tutto per sicurezza". L'ironia: la sicurezza che voleva garantire stava per crashare il server riempiendo il disco.

---

## I tre formati del binlog: STATEMENT, ROW, MIXED

Non tutti i binlog sono uguali. MySQL supporta tre formati di registrazione, e la scelta ha implicazioni concrete.

### STATEMENT

Registra l'istruzione SQL così com'è stata eseguita. Compatto, leggibile, ma problematico: funzioni come `NOW()`, `UUID()`, `RAND()` producono risultati diversi sul master e sullo slave. Le query con `LIMIT` senza `ORDER BY` possono produrre risultati non deterministici.

```sql
SET binlog_format = 'STATEMENT';
```

### ROW

Registra il cambiamento a livello di riga — prima e dopo. Più pesante in termini di spazio, ma deterministico al 100%. Se aggiorni 10.000 righe, il binlog contiene 10.000 before/after image. Grande, ma sicuro.

```sql
SET binlog_format = 'ROW';
```

### MIXED

MySQL decide caso per caso: usa STATEMENT quando è safe, passa automaticamente a ROW quando rileva operazioni non deterministiche.

```sql
SET binlog_format = 'MIXED';
```

Il mio consiglio: **usa ROW**. È il default da MySQL 5.7.7, è quello che Galera Cluster richiede, è quello che tutti i tool di replica moderni si aspettano. STATEMENT è un retaggio del passato, MIXED è un compromesso che aggiunge complessità senza un reale vantaggio.

L'unico caso in cui ROW diventa un problema è quando fai operazioni massive — un `UPDATE` su milioni di righe genera un binlog enorme perché contiene il before e l'after di ogni riga. In quei casi, la soluzione non è cambiare formato, ma spezzare l'operazione in batch:

```sql
-- Invece di questo (genera binlog gigantesco):
UPDATE orders SET status = 'archived' WHERE order_date < '2025-01-01';

-- Meglio così (batch da 10.000):
UPDATE orders SET status = 'archived'
WHERE order_date < '2025-01-01' AND status != 'archived'
LIMIT 10000;
-- Ripetere fino a 0 rows affected
```

---

## `mysqlbinlog`: leggere i binlog quando serve

Il tool da riga di comando {{< glossary term="mysqlbinlog" >}}`mysqlbinlog`{{< /glossary >}} è l'unico modo per ispezionare il contenuto dei file binlog. Serve in due scenari: debug di problemi di replica e point-in-time recovery.

```bash
# Leggere un binlog in formato leggibile
mysqlbinlog /var/lib/mysql/mysql-bin.000318

# Filtrare per intervallo temporale
mysqlbinlog --start-datetime="2026-03-30 10:00:00" \
            --stop-datetime="2026-03-30 11:00:00" \
            /var/lib/mysql/mysql-bin.000318

# Filtrare per database specifico
mysqlbinlog --database=gestionale /var/lib/mysql/mysql-bin.000318

# Se il formato è ROW, decodificare in SQL leggibile
mysqlbinlog --verbose /var/lib/mysql/mysql-bin.000318
```

Con il formato ROW, senza `--verbose` vedi solo blob binari. Con `--verbose` ottieni le righe in formato pseudo-SQL commentato — non bellissimo, ma leggibile.

---

## Il principio: gestire i binlog, non disabilitarli

Ogni tanto qualcuno suggerisce di risolvere il problema "alla radice" disabilitando i binlog:

```ini
# NON FARE QUESTO in produzione
skip-log-bin
```

Sì, risolve il problema del disco. Ma elimina:

- La possibilità di configurare una replica in futuro
- Il point-in-time recovery
- La capacità di analizzare cosa è successo nel database dopo un incidente
- La compatibilità con strumenti di {{< glossary term="cdc" >}}CDC (Change Data Capture){{< /glossary >}} come Debezium

I binlog non sono un problema. I binlog **non gestiti** sono un problema. La differenza è un parametro di configurazione e un check settimanale. Sul server che ho sistemato, la configurazione finale è stata:

```ini
[mysqld]
log_bin = /var/lib/mysql/mysql-bin
server-id = 1
binlog_format = ROW
binlog_expire_logs_seconds = 604800    # 7 giorni
max_binlog_size = 512M
```

Un `max_binlog_size` di 512 MB invece del default di 1 GB — file più piccoli sono più facili da gestire, trasferire e purgare. La retention a 7 giorni, con backup giornaliero, garantisce copertura PITR completa con occupazione disco prevedibile.

---

## Check-up post intervento

Prima di chiudere il ticket, ho aggiunto un paio di query al sistema di monitoraggio del cliente:

```sql
-- Spazio occupato dai binlog
SELECT
    COUNT(*) AS num_files,
    ROUND(SUM(file_size) / 1024 / 1024 / 1024, 2) AS total_gb
FROM information_schema.BINARY_LOGS;   -- MySQL 8.0+ / Performance Schema

-- Oppure, per tutte le versioni:
SHOW BINARY LOGS;
-- e sommare manualmente o via script
```

```bash
# Alert se i binlog superano i 20 GB
#!/bin/bash
BINLOG_SIZE=$(mysql -u monitor -p'pwd' -Bse \
  "SELECT ROUND(SUM(file_size)/1024/1024/1024,2) FROM performance_schema.binary_log_status" 2>/dev/null)

# Fallback per versioni senza performance_schema.binary_log_status
if [ -z "$BINLOG_SIZE" ]; then
    BINLOG_SIZE=$(du -sh /var/lib/mysql/mysql-bin.* 2>/dev/null | \
      awk '{sum+=$1} END {printf "%.2f", sum/1024}')
fi

if (( $(echo "$BINLOG_SIZE > 20" | bc -l) )); then
    echo "WARNING: binlog size ${BINLOG_SIZE} GB"
fi
```

Tre settimane dopo l'intervento, i binlog occupavano 8 GB — esattamente nella finestra prevista. Il disco non è più andato sopra il 45%.

Il binlog è come l'olio del motore: non ci pensi mai finché non si accende la spia. La differenza è che il motore ti avvisa. MySQL no — continua a scrivere binlog finché il filesystem risponde. Quando smette di rispondere, è troppo tardi per chiedersi perché non avevi impostato la retention.

------------------------------------------------------------------------

## Glossario

**[Binary log](/it/glossary/binary-log/)** — Registro binario sequenziale di MySQL che traccia tutte le modifiche ai dati (INSERT, UPDATE, DELETE, DDL), usato per la replica e il point-in-time recovery. Da MySQL 8.0 è abilitato di default.

**[PITR](/it/glossary/pitr/)** — Point-in-Time Recovery: tecnica di ripristino che combina un backup completo con i binary log per riportare il database a un qualsiasi momento nel tempo, non solo all'ora del backup.

**[Relay log](/it/glossary/relay-log/)** — File di log intermedio sullo slave MySQL che riceve gli eventi dal binary log del master prima che vengano eseguiti localmente dal thread SQL.

**[CDC](/it/glossary/cdc/)** — Change Data Capture: tecnica per intercettare le modifiche ai dati in tempo reale leggendo i log delle transazioni. Strumenti come Debezium leggono i binary log di MySQL per propagare i cambiamenti verso sistemi esterni.

**[mysqlbinlog](/it/glossary/mysqlbinlog/)** — Utility da riga di comando di MySQL per leggere, filtrare e riapplicare il contenuto dei file binary log. Indispensabile per il point-in-time recovery e il debug della replica.
