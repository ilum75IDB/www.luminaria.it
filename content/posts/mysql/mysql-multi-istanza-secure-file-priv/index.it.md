---
title: "MySQL multi-istanza: un ticket, un CSV e il muro di secure-file-priv"
description: "Un'operazione che doveva durare cinque minuti — estrarre un CSV da MySQL — si trasforma in un'indagine tra istanze multiple sullo stesso server, socket Unix, porte diverse e la direttiva secure-file-priv che blocca tutto. Dalla connessione all'istanza giusta fino all'export da shell."
date: "2025-11-04T10:00:00+01:00"
draft: false
translationKey: "mysql_multi_istanza_secure_file_priv"
tags: ["multi-instance", "secure-file-priv", "csv-export", "systemd", "socket", "troubleshooting"]
categories: ["mysql"]
image: "mysql-multi-istanza-secure-file-priv.cover.jpg"
---

Il ticket diceva: "Serve un export CSV dalla tabella ordini del gestionale. Entro le 14."

Erano le 11. Tre ore per una SELECT con INTO OUTFILE — roba da cinque minuti, pensavo. Poi ho aperto la VPN, mi sono collegato al server e ho capito che cinque minuti non sarebbero bastati.

Il server era una macchina CentOS 7 con quattro istanze MySQL. Quattro. Sullo stesso host, con quattro servizi {{< glossary term="systemd" >}}systemd{{< /glossary >}} diversi, quattro porte diverse, quattro socket Unix diverse, quattro directory dati diverse. Un setup che qualcuno aveva messo in piedi anni prima — probabilmente per risparmiare su un secondo server — e che da allora nessuno aveva più toccato né documentato.

Il primo problema non era la query. Il primo problema era: a quale delle quattro istanze devo collegarmi?

---

## L'ambiente: quattro MySQL, un solo server

Ambienti multi-istanza su MySQL non sono rari come si potrebbe pensare. Li trovo più spesso di quanto vorrei, soprattutto in aziende medio-piccole dove i server sono pochi e le applicazioni sono tante. La logica è semplice: invece di comprare quattro server, ne compri uno potente e ci fai girare quattro istanze MySQL, ognuna con il suo database, la sua porta, il suo file di configurazione.

Il risultato funziona, finché non devi fare manutenzione. E la manutenzione su un multi-istanza, senza documentazione, è un esercizio di archeologia informatica.

Su quel server, la situazione era questa:

```bash
systemctl list-units --type=service | grep mysql
    mysqld.service          loaded active running  MySQL Server (porta 3306)
    mysqld-app2.service     loaded active running  MySQL Server (porta 3307)
    mysqld-reporting.service loaded active running  MySQL Server (porta 3308)
    mysqld-legacy.service   loaded active running  MySQL Server (porta 3309)
```

Quattro servizi. I nomi erano vagamente indicativi — "app2", "reporting", "legacy" — ma il ticket parlava del "gestionale" senza specificare quale istanza ospitasse quel database. Nessuna wiki interna, nessun file README sul server, nessun commento nei file di configurazione.

---

## Trovare l'istanza giusta

Il primo passo è stato capire quale istanza contenesse il database degli ordini. La tecnica è sempre la stessa: parti dal servizio systemd, risali al file di configurazione, da lì leggi porta e socket.

```bash
systemctl cat mysqld-app2.service | grep ExecStart
    ExecStart=/usr/sbin/mysqld --defaults-file=/etc/mysql/app2.cnf
```

Ogni servizio puntava a un `my.cnf` diverso. Ho controllato tutti e quattro:

```bash
grep -E "^(port|socket|datadir)" /etc/mysql/app2.cnf
    port      = 3307
    socket    = /var/run/mysqld/mysqld-app2.sock
    datadir   = /data/mysql-app2
```

Per ciascuna istanza, ho annotato porta, socket e datadir. Poi ho fatto il giro rapido:

```bash
mysql --socket=/var/run/mysqld/mysqld.sock -u root -p -e "SHOW DATABASES;" 2>/dev/null
mysql --socket=/var/run/mysqld/mysqld-app2.sock -u root -p -e "SHOW DATABASES;" 2>/dev/null
mysql --socket=/var/run/mysqld/mysqld-reporting.sock -u root -p -e "SHOW DATABASES;" 2>/dev/null
mysql --socket=/var/run/mysqld/mysqld-legacy.sock -u root -p -e "SHOW DATABASES;" 2>/dev/null
```

Il database `gestionale_prod` era sulla seconda istanza — quella sulla porta 3307 con socket `/var/run/mysqld/mysqld-app2.sock`.

Un dettaglio che sembra banale ma che in un ambiente multi-istanza fa la differenza: quando ti colleghi a MySQL specificando solo `-h localhost`, il client non usa TCP. Usa il {{< glossary term="unix-socket" >}}socket Unix{{< /glossary >}} di default, che quasi sempre è quello dell'istanza primaria sulla porta 3306. Se il database che cerchi sta su un'altra istanza, ti colleghi a quella sbagliata senza nemmeno accorgertene.

---

## La connessione e la verifica

Una volta identificata l'istanza, mi sono collegato specificando esplicitamente il socket:

```bash
mysql --socket=/var/run/mysqld/mysqld-app2.sock -u root -p
```

Prima cosa dopo il login: verificare di essere sull'istanza giusta.

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

Porta 3307, database presente, tabella ordini al suo posto. La connessione era quella giusta.

Il check sulla porta sembra paranoia, ma non lo è. In un ambiente con quattro istanze, confondere quale socket punta a quale porta è più facile di quanto si pensi. E l'errore lo scopri solo quando i dati che esporti non sono quelli che ti aspetti — o peggio, quando fai una modifica pensando di essere sul database di test e scopri che eri in produzione.

---

## Il primo tentativo: INTO OUTFILE

La query era semplice. Il richiedente voleva gli ordini dell'ultimo trimestre con importo, cliente e data:

```sql
SELECT o.id_ordine, o.data_ordine, c.ragione_sociale, o.importo_totale
FROM ordini o
JOIN clienti c ON o.id_cliente = c.id_cliente
WHERE o.data_ordine >= '2025-07-01'
ORDER BY o.data_ordine;
```

Il primo istinto è stato usare {{< glossary term="into-outfile" >}}`INTO OUTFILE`{{< /glossary >}}, il modo nativo di MySQL per scrivere risultati su file:

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

La risposta di MySQL è stata secca:

```
ERROR 1290 (HY000): The MySQL server is running with the
--secure-file-priv option so it cannot execute this statement
```

Ecco il muro.

---

## {{< glossary term="secure-file-priv" >}}secure-file-priv{{< /glossary >}}: la direttiva che blocca tutto (e fa bene)

La variabile `secure_file_priv` è il modo in cui MySQL limita le operazioni di lettura e scrittura su file. Controlla dove `LOAD DATA INFILE`, `SELECT INTO OUTFILE` e la funzione `LOAD_FILE()` possono operare.

```sql
SHOW VARIABLES LIKE 'secure_file_priv';
+------------------+------------------------+
| Variable_name    | Value                  |
+------------------+------------------------+
| secure_file_priv | /var/lib/mysql-files/  |
+------------------+------------------------+
```

Questa variabile ha tre modalità:

1. **Un percorso specifico** (es. `/var/lib/mysql-files/`): le operazioni su file funzionano, ma solo dentro quella directory
2. **Stringa vuota** (`""`): nessuna restrizione — MySQL può leggere e scrivere ovunque il suo utente di sistema abbia permessi
3. **NULL**: le operazioni su file sono completamente disabilitate

La mia istanza era configurata con un percorso specifico. Il tentativo di scrivere in `/tmp/` era stato bloccato perché `/tmp/` non è `/var/lib/mysql-files/`.

La prima reazione — quella che vedo fare a molti — sarebbe stata: "cambiamo secure-file-priv a stringa vuota nel my.cnf e riavviamo". No. Assolutamente no. Su un server di produzione con quattro istanze MySQL, riavviare un'istanza alle 11:30 del mattino per un export CSV non è un'opzione. E disabilitare una protezione di sicurezza non è mai la risposta giusta, nemmeno in emergenza.

L'alternativa ovvia era scrivere il file nella directory autorizzata:

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

Ma c'era un altro problema. La directory `/var/lib/mysql-files/` era quella dell'istanza primaria (porta 3306). L'istanza sulla porta 3307 aveva il suo datadir separato in `/data/mysql-app2/`, e la sua `secure_file_priv` puntava a `/data/mysql-app2/files/` — una directory che non esisteva e che nessuno aveva mai creato.

Avrei potuto creare la directory, assegnare i permessi corretti all'utente `mysql` e scrivere lì. Ma a quel punto stavo già perdendo tempo. E c'è un modo più pulito.

---

## La soluzione: export da shell con il client mysql

Quando `INTO OUTFILE` è bloccato o scomodo, la soluzione più pratica è bypassare completamente il meccanismo di scrittura file di MySQL e usare il client da riga di comando per redirigere l'output.

Il trucco sta nelle opzioni `-B` (batch mode) e `-e` (execute):

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

L'opzione `-B` produce un output tab-separated senza i bordi ASCII delle tabelle. Il risultato è un file TSV pulito che si apre senza problemi in qualsiasi foglio di calcolo.

Se serve un vero CSV con le virgole come separatore, basta un passaggio con `sed`:

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

L'opzione `-N` rimuove la riga di intestazione con i nomi delle colonne. Se la vuoi, togli il flag.

Il file era pronto in meno di un minuto. 12.400 righe, 1,2 MB. L'ho copiato sulla mia macchina con `scp`, verificato l'apertura in LibreOffice Calc, e inviato al richiedente. Erano le 11:45. Il ticket che doveva durare cinque minuti ne aveva richiesti quarantacinque — ma almeno non avevo riavviato nessuna istanza.

---

## Perché non disabilitare secure-file-priv

La tentazione di impostare `secure_file_priv = ""` è forte, soprattutto su server di sviluppo o su macchine dove "tanto siamo solo noi". Il problema è che quella protezione esiste per un motivo preciso.

Senza `secure_file_priv`, un utente MySQL con il privilegio `FILE` può:

- Leggere qualsiasi file leggibile dall'utente di sistema `mysql` — inclusi `/etc/passwd`, file di configurazione, chiavi SSH se i permessi non sono blindati
- Scrivere file ovunque l'utente `mysql` abbia permessi di scrittura — inclusa la webroot di un eventuale Apache o Nginx sullo stesso server

In un contesto di {{< glossary term="sql-injection" >}}SQL injection{{< /glossary >}}, il privilegio `FILE` combinato con un `secure_file_priv` vuoto è una porta aperta. L'attaccante può leggere file di sistema, scrivere web shell, fare escalation. Non è teoria — è uno dei vettori di attacco più documentati nelle penetration test su applicazioni web con MySQL dietro.

La regola è semplice: `secure_file_priv` si configura con un percorso specifico, si creano le directory necessarie per ogni istanza al momento del setup, e si lasciano lì. Se serve fare export occasionali, il client mysql da shell fa lo stesso lavoro senza toccare la configurazione di sicurezza.

---

## Lezioni da un ticket da cinque minuti

Quel ticket mi ha ricordato tre cose che in trent'anni di lavoro con i database ho visto confermate centinaia di volte.

La prima: **in un ambiente multi-istanza, il primo passo è sempre identificare l'istanza**. Sembra ovvio, ma la quantità di errori che nascono dal connettersi all'istanza sbagliata — pensando di essere altrove — è impressionante. Un `SHOW VARIABLES LIKE 'port'` dopo ogni connessione non è paranoia, è igiene operativa.

La seconda: **secure-file-priv non è un ostacolo, è una protezione**. Quando ti blocca, non è il momento di disabilitarla. È il momento di usare un percorso alternativo o un metodo alternativo. La direttiva esiste perché MySQL in mano a un utente con il privilegio FILE e nessun vincolo sul filesystem è un rischio concreto.

La terza: **il client mysql da riga di comando è più potente di quanto la maggior parte dei DBA gli riconosca**. Con `-B`, `-N`, `-e` e una pipe verso `sed` o `awk`, puoi fare export, trasformazioni e automazioni senza mai toccare `INTO OUTFILE`. È meno elegante, forse. Ma funziona sempre, non richiede permessi speciali e non ha bisogno che qualcuno abbia creato la directory giusta sei mesi prima.

Il CSV è arrivato alle 11:45. Il richiedente non ha mai saputo che dietro cinque colonne e 12.400 righe c'erano quarantacinque minuti di archeologia sistemistica. Ma è così che funzionano i ticket: chi li apre vede il risultato, chi li risolve vede il percorso.

------------------------------------------------------------------------

## Glossario

**[secure-file-priv](/it/glossary/secure-file-priv/)** — Direttiva di sicurezza MySQL che limita le directory in cui il server può leggere e scrivere file tramite `INTO OUTFILE`, `LOAD DATA INFILE` e `LOAD_FILE()`.

**[Unix Socket](/it/glossary/unix-socket/)** — Meccanismo di comunicazione locale tra processi su sistemi Linux, usato da MySQL come metodo di connessione predefinito quando ci si collega a `localhost`.

**[INTO OUTFILE](/it/glossary/into-outfile/)** — Clausola SQL di MySQL per esportare il risultato di una query direttamente in un file sul filesystem del server. Soggetta alle restrizioni di `secure-file-priv`.

**[systemd](/it/glossary/systemd/)** — Gestore dei servizi su Linux moderno, usato per gestire istanze multiple di MySQL sullo stesso server tramite unit file separati.

**[SQL Injection](/it/glossary/sql-injection/)** — Tecnica di attacco che inserisce codice SQL malevolo negli input di un'applicazione. La direttiva `secure-file-priv` contribuisce a mitigarne l'impatto.
