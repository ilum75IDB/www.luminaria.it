---
title: "Binary log în MySQL: ce sunt, cum le gestionezi și când le poți șterge"
description: "Un server MySQL cu discul la 95%, 180 GB de binary log acumulate în șase luni. De acolo pornește o călătorie prin binlog: ce conțin, de ce există, cum funcționează cu replicarea și point-in-time recovery, și mai ales cum le gestionezi fără să faci pagube."
date: "2026-03-31T10:00:00+01:00"
draft: false
translationKey: "binary_log_mysql"
tags: ["binlog", "replication", "disk-space", "recovery", "mariadb"]
categories: ["mysql"]
image: "binary-log-mysql.cover.jpg"
---

Mesajul pe canalul Slack al echipei de infrastructură era din acelea care te fac să ridici capul de la ecran: "Disc la 95% pe db-ul de producție. Cine poate să se uite?"

Serverul era un MySQL 8.0 pe Rocky Linux, un sistem de gestiune folosit de aproximativ o sută de utilizatori. Baza de date în sine ocupa circa 40 GB — nimic extraordinar. Dar în directorul de date se aflau 180 GB de binary log. Șase luni de binlog pe care nimeni nu se gândise să le gestioneze.

Nu e prima dată când văd acest scenariu. De fapt, aș spune că este unul dintre cele mai recurente pattern-uri din tichetele care îmi ajung pe masă. Binary log-ul este una din acele funcționalități MySQL care lucrează în tăcere, fără să ceară nimic — până când discul se umple.

---

## Ce sunt binary log-urile, concret

{{< glossary term="binary-log" >}}Binary log{{< /glossary >}}-ul este un registru secvențial al tuturor evenimentelor care modifică datele din baza de date. Fiecare INSERT, UPDATE, DELETE, fiecare DDL — totul se scrie în fișiere binare numerotate progresiv: `mysql-bin.000001`, `mysql-bin.000002` și așa mai departe.

Numele este ușor înșelător. Nu e un "log" în sensul syslog-ului sau al error log-ului — nu este făcut pentru a fi citit de un om. Este un flux binar structurat pe care MySQL îl folosește intern pentru două scopuri fundamentale:

1. **Replicare**: slave-ul citește binlog-urile master-ului pentru a replica aceleași operațiuni
2. **{{< glossary term="pitr" >}}Point-in-time recovery (PITR){{< /glossary >}}**: după restaurarea unui backup, poți "reaplica" binlog-urile pentru a aduce datele până la un moment precis

Fără binary log, nu poți face nici una, nici cealaltă. Acesta e motivul pentru care primul instinct — "dezactivăm binlog-urile ca să nu umple discul" — este aproape întotdeauna greșit.

---

## Cum generează MySQL binlog-urile

Binary logging-ul se activează prin parametrul `log_bin`. De la MySQL 8.0 este activat implicit — o schimbare importantă față de versiunile anterioare unde trebuia activat explicit.

```ini
[mysqld]
log_bin = /var/lib/mysql/mysql-bin
server-id = 1
```

MySQL creează un nou fișier binlog în mai multe circumstanțe:

- Când serverul pornește sau repornește
- Când fișierul curent atinge dimensiunea definită de `max_binlog_size` (implicit: 1 GB)
- Când execuți `FLUSH BINARY LOGS`
- Când are loc o rotație manuală

Fiecare fișier binlog are un fișier index asociat (`mysql-bin.index`) care ține evidența tuturor fișierelor binlog active. Acest fișier este critic: dacă îl corupi sau îl editezi manual, MySQL nu mai știe ce binlog-uri există.

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

O sută șaptezeci și două de fișiere. Fiecare de aproximativ un gigabyte. Calculul se confirmă: 180 GB de binlog-uri niciodată purgate.

---

## Rolul în replicare

Într-o arhitectură master-slave, binary log-ul este mecanismul de transport al datelor. Fluxul este acesta:

1. Master-ul scrie fiecare tranzacție în binlog
2. Slave-ul are un thread (I/O thread) care se conectează la master și citește binlog-urile
3. Slave-ul scrie ce primește în propriul {{< glossary term="relay-log" >}}relay log{{< /glossary >}}
4. Un al doilea thread (SQL thread) pe slave execută evenimentele din {{< glossary term="relay-log" >}}relay log{{< /glossary >}}

Aceasta înseamnă că binlog-urile pe master **trebuie să rămână disponibile până când toate slave-urile le-au citit**. Dacă ștergi un binlog pe care slave-ul nu l-a consumat încă, replicarea se rupe.

Înainte de a atinge orice binlog pe un master, comanda de executat este:

```sql
SHOW REPLICA STATUS\G
-- sau, pe versiuni mai vechi:
SHOW SLAVE STATUS\G
```

Câmpul care interesează este `Relay_Master_Log_File` (sau `Source_Log_File` în versiunile recente): îți spune ce binlog citește slave-ul în acel moment. Toate fișierele anterioare sunt sigure de eliminat.

---

## Point-in-time recovery: celălalt motiv pentru care binlog-urile există

Al doilea uz — adesea subestimat — este point-in-time recovery. Scenariul este acesta: ai un backup făcut la ora 3 noaptea. La 14:30 cineva execută un `DROP TABLE` greșit. Fără binlog, poți restaura backup-ul și pierzi tot ce s-a întâmplat între 3:00 și 14:30. Cu binlog-urile, faci restore-ul și apoi reaplici binlog-urile până la 14:29.

```bash
# Găsirea evenimentului DROP TABLE
mysqlbinlog --start-datetime="2026-03-30 14:00:00" \
            --stop-datetime="2026-03-30 15:00:00" \
            /var/lib/mysql/mysql-bin.000318 | grep -i "DROP"

# Reaplicarea binlog-urilor până în momentul dinaintea dezastrului
mysqlbinlog --stop-datetime="2026-03-30 14:29:00" \
            /var/lib/mysql/mysql-bin.000310 \
            /var/lib/mysql/mysql-bin.000311 \
            ... \
            /var/lib/mysql/mysql-bin.000318 | mysql -u root -p
```

În practică, binlog-urile sunt asigurarea ta. Backup-ul este baza, binlog-urile acoperă delta. Ștergerea binlog-urilor fără un backup recent e ca și cum ai anula asigurarea cu o zi înainte de furtună.

---

## PURGE BINARY LOGS: modul corect de a face curățenie

Ne întoarcem la serverul nostru cu discul la 95%. Tentația de a face un `rm -f mysql-bin.*` e puternică. Dar e greșită, din două motive:

1. MySQL nu știe că ai șters fișierele — fișierul index încă arată spre binlog-uri care nu mai există
2. Dacă există o replică activă, riști să rupi sincronizarea

Modul corect este comanda `PURGE`:

```sql
-- Eliminarea tuturor binlog-urilor anterioare unui fișier specific
PURGE BINARY LOGS TO 'mysql-bin.000300';

-- Sau eliminarea tuturor binlog-urilor mai vechi decât o anumită dată
PURGE BINARY LOGS BEFORE '2026-03-01 00:00:00';
```

`PURGE` face trei lucruri pe care `rm` nu le face:

- Actualizează fișierul index
- Verifică dacă fișierele nu sunt necesare pentru replicare (teoretic — dar verifică tu mai întâi)
- Elimină fișierele în mod ordonat

În cazul serverului nostru, mai întâi am verificat că nu existau slave-uri:

```sql
SHOW REPLICAS;
-- Empty set
```

Nicio replică. Apoi am verificat care era binlog-ul curent:

```sql
SHOW MASTER STATUS;
+------------------+----------+
| File             | Position |
+------------------+----------+
| mysql-bin.000318 | 52428800 |
+------------------+----------+
```

Păstrând ultimele 3 fișiere pentru siguranță:

```sql
PURGE BINARY LOGS TO 'mysql-bin.000316';
```

Rezultat: 175 GB eliberate în câteva secunde. Discul a coborât de la 95% la 28%.

---

## Configurarea retenției automate

Rezolvarea urgenței e un lucru. A face ca problema să nu se mai repete e altceva. MySQL oferă doi parametri pentru gestionarea automată a retenției:

### `expire_logs_days` (legacy)

```ini
[mysqld]
expire_logs_days = 14
```

Elimină automat binlog-urile mai vechi de 14 zile. Simplu dar grosier — granularitatea este doar în zile.

### `binlog_expire_logs_seconds` (MySQL 8.0+)

```ini
[mysqld]
binlog_expire_logs_seconds = 1209600   # 14 zile în secunde
```

Aceeași logică, dar cu granularitate la secundă. De la MySQL 8.0, acest parametru are prioritate asupra `expire_logs_days`. Dacă le setezi pe amândouă, câștigă `binlog_expire_logs_seconds`.

Întrebarea pe care mi-o pun mereu este: "Câte zile de retenție?"

Depinde. Dar iată regulile mele practice:

| Scenariu | Retenție recomandată |
|----------|----------------------|
| Server standalone, backup zilnic | 7 zile |
| Master cu replică, backup zilnic | 7-14 zile |
| Master cu replică lentă sau în zone diferite | 14-30 zile |
| Medii reglementate (finanțe, sănătate) | 30-90 zile, cu arhivare |

Principiul este: **retenția binlog-urilor trebuie să acopere cel puțin dublul intervalului dintre două backup-uri**. Dacă faci backup în fiecare noapte, păstrează cel puțin 2-3 zile de binlog. Dacă faci backup săptămânal, cel puțin 14 zile.

În cazul serverului nostru, nu fusese configurată nicio retenție. Valoarea implicită a MySQL 8.0 este 30 de zile — dar acea valoare fusese suprascrisă la 0 (fără expirare) într-un `my.cnf` personalizat de cineva care "voia să păstreze totul pentru siguranță". Ironia: siguranța pe care voia s-o garanteze era pe cale să prăbușească serverul umplând discul.

---

## Cele trei formate ale binlog-ului: STATEMENT, ROW, MIXED

Nu toate binlog-urile sunt la fel. MySQL suportă trei formate de înregistrare, iar alegerea are implicații concrete.

### STATEMENT

Înregistrează instrucțiunea SQL așa cum a fost executată. Compact, lizibil, dar problematic: funcții ca `NOW()`, `UUID()`, `RAND()` produc rezultate diferite pe master și pe slave. Query-urile cu `LIMIT` fără `ORDER BY` pot produce rezultate nedeterministe.

```sql
SET binlog_format = 'STATEMENT';
```

### ROW

Înregistrează modificarea la nivel de rând — înainte și după. Mai greu ca spațiu, dar determinist 100%. Dacă actualizezi 10.000 de rânduri, binlog-ul conține 10.000 de imagini before/after. Mare, dar sigur.

```sql
SET binlog_format = 'ROW';
```

### MIXED

MySQL decide de la caz la caz: folosește STATEMENT când e sigur, trece automat la ROW când detectează operațiuni nedeterministe.

```sql
SET binlog_format = 'MIXED';
```

Sfatul meu: **folosește ROW**. Este valoarea implicită de la MySQL 5.7.7, este ceea ce Galera Cluster cere, este ceea ce toate tool-urile moderne de replicare așteaptă. STATEMENT este o moștenire din trecut, MIXED este un compromis care adaugă complexitate fără un beneficiu real.

Singurul caz în care ROW devine o problemă este când faci operațiuni masive — un `UPDATE` pe milioane de rânduri generează un binlog enorm pentru că conține before și after-ul fiecărui rând. În acele cazuri, soluția nu e să schimbi formatul, ci să spargi operațiunea în batch-uri:

```sql
-- În loc de asta (generează binlog gigantic):
UPDATE orders SET status = 'archived' WHERE order_date < '2025-01-01';

-- Mai bine așa (batch-uri de 10.000):
UPDATE orders SET status = 'archived'
WHERE order_date < '2025-01-01' AND status != 'archived'
LIMIT 10000;
-- Repetă până la 0 rows affected
```

---

## `mysqlbinlog`: citirea binlog-urilor când e nevoie

Tool-ul de linie de comandă {{< glossary term="mysqlbinlog" >}}`mysqlbinlog`{{< /glossary >}} este singura modalitate de a inspecta conținutul fișierelor binlog. Se folosește în două scenarii: debug la probleme de replicare și point-in-time recovery.

```bash
# Citirea unui binlog în format lizibil
mysqlbinlog /var/lib/mysql/mysql-bin.000318

# Filtrare pe interval de timp
mysqlbinlog --start-datetime="2026-03-30 10:00:00" \
            --stop-datetime="2026-03-30 11:00:00" \
            /var/lib/mysql/mysql-bin.000318

# Filtrare pe bază de date specifică
mysqlbinlog --database=gestionale /var/lib/mysql/mysql-bin.000318

# Dacă formatul e ROW, decodificare în SQL lizibil
mysqlbinlog --verbose /var/lib/mysql/mysql-bin.000318
```

Cu format ROW, fără `--verbose` vezi doar blob-uri binare. Cu `--verbose` obții rândurile în format pseudo-SQL comentat — nu e frumos, dar se poate citi.

---

## Principiul: gestionează binlog-urile, nu le dezactiva

Din când în când cineva sugerează să rezolvi problema "din rădăcină" dezactivând binlog-urile:

```ini
# NU FACE ASTA în producție
skip-log-bin
```

Da, rezolvă problema discului. Dar elimină:

- Posibilitatea de a configura o replică în viitor
- Point-in-time recovery
- Capacitatea de a analiza ce s-a întâmplat în baza de date după un incident
- Compatibilitatea cu instrumente de {{< glossary term="cdc" >}}CDC (Change Data Capture){{< /glossary >}} precum Debezium

Binlog-urile nu sunt o problemă. Binlog-urile **negestionate** sunt o problemă. Diferența este un parametru de configurare și o verificare săptămânală. Pe serverul pe care l-am reparat, configurația finală a fost:

```ini
[mysqld]
log_bin = /var/lib/mysql/mysql-bin
server-id = 1
binlog_format = ROW
binlog_expire_logs_seconds = 604800    # 7 zile
max_binlog_size = 512M
```

Un `max_binlog_size` de 512 MB în loc de 1 GB implicit — fișierele mai mici sunt mai ușor de gestionat, transferat și purgat. Retenția la 7 zile, cu backup zilnic, garantează acoperire PITR completă cu ocupare de disc predictibilă.

---

## Verificare post intervenție

Înainte de a închide tichetul, am adăugat câteva query-uri la sistemul de monitorizare al clientului:

```sql
-- Spațiu ocupat de binlog-uri
SELECT
    COUNT(*) AS num_files,
    ROUND(SUM(file_size) / 1024 / 1024 / 1024, 2) AS total_gb
FROM information_schema.BINARY_LOGS;   -- MySQL 8.0+ / Performance Schema

-- Sau, pentru toate versiunile:
SHOW BINARY LOGS;
-- și adună manual sau prin script
```

```bash
# Alertă dacă binlog-urile depășesc 20 GB
#!/bin/bash
BINLOG_SIZE=$(mysql -u monitor -p'pwd' -Bse \
  "SELECT ROUND(SUM(file_size)/1024/1024/1024,2) FROM performance_schema.binary_log_status" 2>/dev/null)

# Fallback pentru versiuni fără performance_schema.binary_log_status
if [ -z "$BINLOG_SIZE" ]; then
    BINLOG_SIZE=$(du -sh /var/lib/mysql/mysql-bin.* 2>/dev/null | \
      awk '{sum+=$1} END {printf "%.2f", sum/1024}')
fi

if (( $(echo "$BINLOG_SIZE > 20" | bc -l) )); then
    echo "WARNING: binlog size ${BINLOG_SIZE} GB"
fi
```

La trei săptămâni după intervenție, binlog-urile ocupau 8 GB — exact în fereastra prevăzută. Discul nu a mai depășit 45%.

Binlog-ul e ca uleiul de motor: nu te gândești niciodată la el până nu se aprinde martorul. Diferența e că motorul te avertizează. MySQL nu — continuă să scrie binlog-uri atâta timp cât filesystem-ul răspunde. Când nu mai răspunde, e prea târziu să te întrebi de ce nu configuraseși retenția.

------------------------------------------------------------------------

## Glosar

**[Binary log](/ro/glossary/binary-log/)** — Registrul binar secvențial al MySQL care urmărește toate modificările de date (INSERT, UPDATE, DELETE, DDL), folosit pentru replicare și point-in-time recovery. Activat implicit de la MySQL 8.0.

**[PITR](/ro/glossary/pitr/)** — Point-in-Time Recovery: tehnică de restaurare care combină un backup complet cu binary log-urile pentru a readuce baza de date la orice moment în timp, nu doar la momentul backup-ului.

**[Relay log](/ro/glossary/relay-log/)** — Fișier de log intermediar pe slave-ul MySQL care primește evenimentele din binary log-ul master-ului înainte de a fi executate local de thread-ul SQL.

**[CDC](/ro/glossary/cdc/)** — Change Data Capture: tehnică pentru interceptarea modificărilor de date în timp real prin citirea log-urilor de tranzacții. Instrumente precum Debezium citesc binary log-urile MySQL pentru a propaga modificările către sisteme externe.

**[mysqlbinlog](/ro/glossary/mysqlbinlog/)** — Utilitar de linie de comandă MySQL pentru citirea, filtrarea și reaplicarea conținutului fișierelor binary log. Esențial pentru point-in-time recovery și debug-ul replicării.
