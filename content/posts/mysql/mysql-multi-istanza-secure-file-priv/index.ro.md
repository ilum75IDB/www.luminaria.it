---
title: "MySQL multi-instanță: un ticket, un CSV și zidul secure-file-priv"
description: "O operațiune care trebuia să dureze cinci minute — extragerea unui CSV din MySQL — se transformă într-o investigație printre instanțe multiple pe același server, socket-uri Unix, porturi diferite și directiva secure-file-priv care blochează totul. De la conectarea la instanța corectă până la exportul din shell."
date: "2025-11-04T10:00:00+01:00"
draft: false
translationKey: "mysql_multi_istanza_secure_file_priv"
tags: ["multi-instance", "secure-file-priv", "csv-export", "systemd", "socket", "troubleshooting"]
categories: ["mysql"]
image: "mysql-multi-istanza-secure-file-priv.cover.jpg"
---

Ticket-ul spunea: „Avem nevoie de un export CSV din tabelul ordini al aplicației de gestiune. Până la ora 14."

Era 11 dimineața. Trei ore pentru un SELECT cu INTO OUTFILE — treabă de cinci minute, mă gândeam. Apoi am deschis VPN-ul, m-am conectat la server și am înțeles că cinci minute n-o să fie de ajuns.

Serverul era o mașină CentOS 7 cu patru instanțe MySQL. Patru. Pe același host, cu patru servicii systemd diferite, patru porturi diferite, patru socket-uri Unix diferite, patru directoare de date diferite. Un setup pe care cineva îl pusese în picioare cu ani în urmă — probabil ca să economisească un al doilea server — și pe care de atunci nimeni nu-l mai atinsese și nici nu-l documentase.

Prima problemă nu era query-ul. Prima problemă era: la care dintre cele patru instanțe trebuie să mă conectez?

---

## Mediul: patru MySQL, un singur server

Mediile multi-instanță pe MySQL nu sunt atât de rare pe cât s-ar crede. Le întâlnesc mai des decât mi-aș dori, mai ales în companiile mici și medii unde serverele sunt puține și aplicațiile sunt multe. Logica e simplă: în loc să cumperi patru servere, cumperi unul puternic și rulezi patru instanțe MySQL pe el, fiecare cu baza ei de date, portul ei, fișierul ei de configurare.

Rezultatul funcționează, până când trebuie să faci mentenanță. Iar mentenanța pe un multi-instanță, fără documentație, e un exercițiu de arheologie informatică.

Pe acel server, situația era următoarea:

```bash
systemctl list-units --type=service | grep mysql
    mysqld.service          loaded active running  MySQL Server (porta 3306)
    mysqld-app2.service     loaded active running  MySQL Server (porta 3307)
    mysqld-reporting.service loaded active running  MySQL Server (porta 3308)
    mysqld-legacy.service   loaded active running  MySQL Server (porta 3309)
```

Patru servicii. Numele erau vag sugestive — „app2", „reporting", „legacy" — dar ticket-ul vorbea despre „aplicația de gestiune" fără să specifice care instanță găzduiește acea bază de date. Niciun wiki intern, niciun fișier README pe server, niciun comentariu în fișierele de configurare.

---

## Găsirea instanței corecte

Primul pas a fost să identific care instanță conține baza de date cu comenzile. Tehnica e mereu aceeași: pornești de la serviciul systemd, urci la fișierul de configurare, de acolo citești portul și socket-ul.

```bash
systemctl cat mysqld-app2.service | grep ExecStart
    ExecStart=/usr/sbin/mysqld --defaults-file=/etc/mysql/app2.cnf
```

Fiecare serviciu avea un `my.cnf` diferit. Le-am verificat pe toate patru:

```bash
grep -E "^(port|socket|datadir)" /etc/mysql/app2.cnf
    port      = 3307
    socket    = /var/run/mysqld/mysqld-app2.sock
    datadir   = /data/mysql-app2
```

Pentru fiecare instanță am notat portul, socket-ul și datadir-ul. Apoi am făcut o trecere rapidă:

```bash
mysql --socket=/var/run/mysqld/mysqld.sock -u root -p -e "SHOW DATABASES;" 2>/dev/null
mysql --socket=/var/run/mysqld/mysqld-app2.sock -u root -p -e "SHOW DATABASES;" 2>/dev/null
mysql --socket=/var/run/mysqld/mysqld-reporting.sock -u root -p -e "SHOW DATABASES;" 2>/dev/null
mysql --socket=/var/run/mysqld/mysqld-legacy.sock -u root -p -e "SHOW DATABASES;" 2>/dev/null
```

Baza de date `gestionale_prod` era pe a doua instanță — cea de pe portul 3307 cu socket-ul `/var/run/mysqld/mysqld-app2.sock`.

Un detaliu care pare banal dar care într-un mediu multi-instanță face diferența: când te conectezi la MySQL specificând doar `-h localhost`, clientul nu folosește TCP. Folosește socket-ul Unix implicit, care aproape întotdeauna e cel al instanței primare de pe portul 3306. Dacă baza de date pe care o cauți e pe altă instanță, te conectezi la cea greșită fără să-ți dai seama.

---

## Conexiunea și verificarea

Odată identificată instanța, m-am conectat specificând explicit socket-ul:

```bash
mysql --socket=/var/run/mysqld/mysqld-app2.sock -u root -p
```

Primul lucru după autentificare: verificarea că sunt pe instanța corectă.

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

Portul 3307, baza de date prezentă, tabelul ordini la locul lui. Conexiunea era cea corectă.

Verificarea portului pare paranoia, dar nu este. Într-un mediu cu patru instanțe, a confunda care socket duce la care port e mai ușor decât crezi. Iar eroarea o descoperi doar când datele pe care le exporți nu sunt cele așteptate — sau mai rău, când faci o modificare crezând că ești pe baza de test și descoperi că erai în producție.

---

## Prima tentativă: INTO OUTFILE

Query-ul era simplu. Solicitantul voia comenzile din ultimul trimestru cu sumă, client și dată:

```sql
SELECT o.id_ordine, o.data_ordine, c.ragione_sociale, o.importo_totale
FROM ordini o
JOIN clienti c ON o.id_cliente = c.id_cliente
WHERE o.data_ordine >= '2025-07-01'
ORDER BY o.data_ordine;
```

Primul instinct a fost să folosesc `INTO OUTFILE`, metoda nativă a MySQL-ului pentru a scrie rezultate în fișier:

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

Răspunsul MySQL-ului a fost sec:

```
ERROR 1290 (HY000): The MySQL server is running with the
--secure-file-priv option so it cannot execute this statement
```

Iată zidul.

---

## secure-file-priv: directiva care blochează totul (și face bine)

Variabila `secure_file_priv` este modul în care MySQL limitează operațiunile de citire și scriere pe fișiere. Controlează unde `LOAD DATA INFILE`, `SELECT INTO OUTFILE` și funcția `LOAD_FILE()` pot opera.

```sql
SHOW VARIABLES LIKE 'secure_file_priv';
+------------------+------------------------+
| Variable_name    | Value                  |
+------------------+------------------------+
| secure_file_priv | /var/lib/mysql-files/  |
+------------------+------------------------+
```

Această variabilă are trei moduri:

1. **O cale specifică** (ex. `/var/lib/mysql-files/`): operațiunile pe fișiere funcționează, dar doar în acel director
2. **Șir gol** (`""`): nicio restricție — MySQL poate citi și scrie oriunde utilizatorul său de sistem are permisiuni
3. **NULL**: operațiunile pe fișiere sunt complet dezactivate

Instanța mea era configurată cu o cale specifică. Tentativa de a scrie în `/tmp/` fusese blocată pentru că `/tmp/` nu este `/var/lib/mysql-files/`.

Prima reacție — cea pe care o văd la mulți — ar fi fost: „schimbăm secure-file-priv la șir gol în my.cnf și repornim". Nu. Absolut nu. Pe un server de producție cu patru instanțe MySQL, repornirea unei instanțe la 11:30 dimineața pentru un export CSV nu e o opțiune. Iar dezactivarea unei protecții de securitate nu e niciodată răspunsul corect, nici măcar în urgență.

Alternativa evidentă era să scriu fișierul în directorul autorizat:

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

Dar era o altă problemă. Directorul `/var/lib/mysql-files/` era cel al instanței primare (portul 3306). Instanța de pe portul 3307 avea datadir-ul separat în `/data/mysql-app2/`, iar `secure_file_priv`-ul ei indica spre `/data/mysql-app2/files/` — un director care nu exista și pe care nimeni nu-l crease vreodată.

Aș fi putut crea directorul, să atribui permisiunile corecte utilizatorului `mysql` și să scriu acolo. Dar la acel punct deja pierdeam timp. Și există o metodă mai curată.

---

## Soluția: export din shell cu clientul mysql

Când `INTO OUTFILE` e blocat sau incomod, soluția cea mai practică e să ocolești complet mecanismul de scriere în fișier al MySQL-ului și să folosești clientul din linia de comandă pentru a redirecționa output-ul.

Trucul stă în opțiunile `-B` (batch mode) și `-e` (execute):

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

Opțiunea `-B` produce un output tab-separated fără chenarele ASCII ale tabelelor. Rezultatul e un fișier TSV curat care se deschide fără probleme în orice foaie de calcul.

Dacă e nevoie de un CSV real cu virgule ca separator, e suficient un pas cu `sed`:

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

Opțiunea `-N` elimină rândul de antet cu numele coloanelor. Dacă vrei antetul, scoate flag-ul.

Fișierul a fost gata în mai puțin de un minut. 12.400 de rânduri, 1,2 MB. L-am copiat pe mașina mea cu `scp`, am verificat deschiderea în LibreOffice Calc și l-am trimis solicitantului. Era 11:45. Ticket-ul care trebuia să dureze cinci minute consumase patruzeci și cinci — dar cel puțin nu repornisem nicio instanță.

---

## De ce să nu dezactivezi secure-file-priv

Tentația de a seta `secure_file_priv = ""` e puternică, mai ales pe servere de dezvoltare sau pe mașini unde „oricum suntem doar noi". Problema e că acea protecție există dintr-un motiv precis.

Fără `secure_file_priv`, un utilizator MySQL cu privilegiul `FILE` poate:

- Citi orice fișier accesibil utilizatorului de sistem `mysql` — inclusiv `/etc/passwd`, fișiere de configurare, chei SSH dacă permisiunile nu sunt blindate
- Scrie fișiere oriunde utilizatorul `mysql` are permisiuni de scriere — inclusiv în webroot-ul unui eventual Apache sau Nginx de pe același server

Într-un context de SQL injection, privilegiul `FILE` combinat cu un `secure_file_priv` gol e o ușă deschisă. Atacatorul poate citi fișiere de sistem, scrie web shell-uri, face escaladare de privilegii. Nu e teorie — este unul dintre vectorii de atac cei mai documentați în testele de penetrare pe aplicații web cu MySQL în spate.

Regula e simplă: `secure_file_priv` se configurează cu o cale specifică, se creează directoarele necesare pentru fiecare instanță în momentul setup-ului și se lasă acolo. Dacă trebuie să faci exporturi ocazionale, clientul mysql din shell face aceeași treabă fără să atingi configurația de securitate.

---

## Lecții de la un ticket de cinci minute

Acel ticket mi-a reamintit trei lucruri pe care în treizeci de ani de lucru cu bazele de date le-am văzut confirmate de sute de ori.

Primul: **într-un mediu multi-instanță, primul pas e întotdeauna identificarea instanței**. Pare evident, dar cantitatea de erori care se nasc din conectarea la instanța greșită — crezând că ești în altă parte — e impresionantă. Un `SHOW VARIABLES LIKE 'port'` după fiecare conectare nu e paranoia, e igienă operațională.

Al doilea: **secure-file-priv nu e un obstacol, e o protecție**. Când te blochează, nu e momentul s-o dezactivezi. E momentul să folosești o cale alternativă sau o metodă alternativă. Directiva există pentru că MySQL în mâinile unui utilizator cu privilegiul FILE și fără nicio constrângere pe filesystem e un risc concret.

Al treilea: **clientul mysql din linia de comandă e mai puternic decât îi recunosc majoritatea DBA-ilor**. Cu `-B`, `-N`, `-e` și o pipe spre `sed` sau `awk`, poți face exporturi, transformări și automatizări fără să atingi vreodată `INTO OUTFILE`. E mai puțin elegant, poate. Dar funcționează mereu, nu necesită permisiuni speciale și nu depinde de faptul că cineva a creat directorul potrivit cu șase luni înainte.

CSV-ul a ajuns la 11:45. Solicitantul n-a aflat niciodată că în spatele a cinci coloane și 12.400 de rânduri se ascundeau patruzeci și cinci de minute de arheologie de sistem. Dar așa funcționează ticket-urile: cine le deschide vede rezultatul, cine le rezolvă vede drumul.
