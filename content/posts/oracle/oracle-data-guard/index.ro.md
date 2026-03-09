---
title: "De la single instance la Data Guard: ziua în care CEO-ul a înțeles DR-ul"
description: "O bază de date Oracle în producție fără nicio redundanță. O defecțiune de disc care a oprit totul timp de șase ore. Și decizia CEO-ului de a investi într-o arhitectură Active Data Guard cu switchover automat."
date: "2025-12-02T10:00:00+01:00"
draft: false
translationKey: "oracle_data_guard"
tags: ["data-guard", "disaster-recovery", "high-availability", "switchover", "architecture"]
categories: ["oracle"]
image: "oracle-data-guard.cover.jpg"
---

Clientul era o companie de dimensiuni medii din sectorul asigurărilor. Trei sute de angajați, o aplicație de gestiune internă pe Oracle 19c, un singur server fizic în camera serverelor de la parter. Fără replică. Fără standby. Fără plan de disaster recovery.

Timp de cinci ani totul funcționase. Și când lucrurile merg, nimeni nu vrea să cheltuie bani protejându-se de probleme pe care nu le-a văzut niciodată.

## Ziua în care s-a oprit totul

Într-o dimineață de miercuri din noiembrie, la 8:47, discul grupului principal de date a suferit o defecțiune fizică. Nu o eroare logică, nu o corupție recuperabilă. O defecțiune hardware. Controlerul RAID a pierdut două discuri simultan — unul era degradat de săptămâni fără ca nimeni să observe, celălalt a cedat brusc.

Baza de date s-a oprit. Polițele nu se emiteau. Daunele nu se procesau. Call center-ul le răspundea clienților cu "probleme tehnice, sunați mai târziu."

Am primit apelul la 9:15. Când am ajuns la sediu, administratorul de sistem căuta deja discuri compatibile. Le-a găsit după-amiaza devreme. Între înlocuire, reconstrucția RAID-ului și recuperarea bazei de date din backup-ul din noaptea precedentă, sistemul a revenit operațional la 15:20.

Șase ore și jumătate de oprire totală. Și pierderea tuturor tranzacțiilor de la 23:00 seara precedentă până la 8:47 dimineața — aproximativ zece ore de date, pentru că backup-ul era doar nocturn și archived log-urile nu erau copiate pe altă mașină.

CEO-ul a trimis în seara aceea un email întregii companii. A doua zi m-a sunat: "Ce trebuie să facem ca să nu se mai întâmple niciodată?"

## Designul

Răspunsul era simplu ca și concept, mai puțin simplu în realizare: aveau nevoie de o a doua bază de date, sincronizată în timp real, gata să preia rolul primarului în caz de defecțiune.

Oracle Active Data Guard face exact asta. O bază de date primară generează redo log-uri, iar un standby le primește și le aplică continuu. Dacă primarul moare, standby-ul devine primar. Dacă totul e în ordine, standby-ul poate fi folosit și în mod read-only — pentru rapoarte, pentru backup-uri, pentru a ușura încărcarea.

Am proiectat o arhitectură cu două noduri:

- **Primar** (`oraprod1`): serverul existent, cu discuri noi, la sediul principal
- **Standby** (`oraprod2`): un server nou identic, în data center-ul providerului de hosting, la 12 km distanță

Distanța nu era întâmplătoare. Suficient de departe pentru a supraviețui unui eveniment localizat (incendiu, inundație, pană de curent prelungită), suficient de aproape pentru a permite replicarea sincronă fără latență perceptibilă.

## Configurarea

### Pregătirea primarului

Primul pas a fost verificarea că primarul era în mod `ARCHIVELOG` cu `FORCE LOGGING` activ. Fără aceste două precondiții, Data Guard nu are ce replica.

```sql
-- Verificare mod archivelog
SELECT log_mode FROM v$database;

-- Dacă e necesar, activare
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;

-- Force logging: previne operațiunile NOLOGGING
ALTER DATABASE FORCE LOGGING;
```

`FORCE LOGGING` este fundamental. Fără el, orice operațiune cu clauza `NOLOGGING` — un `CREATE TABLE AS SELECT`, un `ALTER INDEX REBUILD` — nu generează redo și creează goluri în replicare. Am văzut asta întâmplându-se de trei ori în carieră. A treia oară am decis că `FORCE LOGGING` se activează întotdeauna, fără excepții.

### Standby redo log-uri

Pe primar am creat standby redo log-urile — grupuri dedicate care vor fi folosite când (și dacă) acest server devine standby după un switchover.

```sql
-- Standby redo log-uri: n+1 față de redo log-urile online
-- Dacă ai 3 grupuri online, creezi 4 standby
ALTER DATABASE ADD STANDBY LOGFILE GROUP 4 SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 5 SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 6 SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 7 SIZE 200M;
```

Regula este n+1: dacă primarul are trei grupuri de redo log, standby-ul are nevoie de patru. Nu e documentată foarte clar, dar am învățat-o pe propria piele — cu trei grupuri egale, sub încărcare grea standby-ul se poate bloca așteptând un grup liber.

### Configurarea rețelei

`tnsnames.ora` pe ambele noduri trebuie să cunoască atât primarul cât și standby-ul. Configurarea este simetrică:

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

`listener.ora` pe standby trebuie să includă o intrare statică pentru baza de date, pentru că în timpul restore-ului standby-ul nu este încă deschis și listener-ul nu îl poate înregistra dinamic:

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

Sufixul `_DGMGRL` este folosit de Data Guard Broker pentru a identifica instanța. Fără această intrare statică, broker-ul nu se poate conecta la standby și operațiunile de switchover eșuează cu erori criptice care te fac să pierzi o jumătate de zi.

### Crearea standby-ului

Pentru copia inițială a bazei de date am folosit un `DUPLICATE` prin RMAN prin rețea. Fără backup pe bandă, fără transfer manual de fișiere. Direct, de la primar la standby:

```
-- Pe serverul standby, pornirea instanței în NOMOUNT
STARTUP NOMOUNT PFILE='/u01/app/oracle/product/19c/dbs/initoraprod.ora';

-- Din RMAN, conectat la ambele
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

`NOFILENAMECHECK` se folosește când căile fișierelor sunt identice pe ambele mașini — aceeași structură de directoare, aceeași convenție de nume. Dacă căile diferă, sunt necesari parametrii `DB_FILE_NAME_CONVERT` și `LOG_FILE_NAME_CONVERT`.

Copia a durat aproximativ trei ore pentru 400 GB printr-o linie dedicată de 1 Gbps. Nu cea mai rapidă, dar este o operațiune care se face o singură dată.

### Data Guard Broker

Broker-ul este componenta care gestionează configurația Data Guard centralizat și permite switchover-ul cu o singură comandă. Fără Broker poți face totul manual, dar nu vrei să faci asta manual când primarul tocmai a căzut și CEO-ul te sună la fiecare cinci minute.

```sql
-- Pe primar
ALTER SYSTEM SET dg_broker_start=TRUE;

-- Pe standby
ALTER SYSTEM SET dg_broker_start=TRUE;
```

Apoi, din `DGMGRL` pe primar:

```
DGMGRL> CREATE CONFIGURATION dg_config AS
         PRIMARY DATABASE IS oraprod
         CONNECT IDENTIFIER IS ORAPROD1;

DGMGRL> ADD DATABASE oraprod_stby AS
         CONNECT IDENTIFIER IS ORAPROD2
         MAINTAINED AS PHYSICAL;

DGMGRL> ENABLE CONFIGURATION;
```

În acel moment, `SHOW CONFIGURATION` trebuie să returneze:

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

Cuvântul pe care vrei să-l vezi este `SUCCESS`. Orice altceva înseamnă că există o problemă de rețea, configurare sau permisiuni de rezolvat înainte de a merge mai departe.

## Primul switchover

La două săptămâni după punerea în funcțiune a arhitecturii, am făcut primul test de switchover. Într-o sâmbătă dimineața, cu aplicația oprită, dar cu CEO-ul prezent — voia să vadă cu ochii lui.

```
DGMGRL> SWITCHOVER TO oraprod_stby;
```

O singură comandă. Patruzeci și două de secunde. Primarul a devenit standby, standby-ul a devenit primar. Aplicațiile, configurate cu serviciul corect, s-au reconectat automat.

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

Apoi am făcut switchback-ul — revenirea la primarul original. Alte treizeci și opt de secunde. Curat.

CEO-ul s-a uitat la ecran, s-a uitat la mine, și a spus: "Patruzeci și două de secunde versus șase ore. De ce n-am făcut asta mai devreme?"

Nu i-am dat răspunsul. Îl știam amândoi.

## Ce nu-ți spun

Configurarea pe care am descris-o funcționează. Dar sunt lucruri pe care documentația Oracle nu le subliniază suficient.

**Gap-ul de rețea.** Replicarea sincronă (`SYNC`) garantează zero pierderi de date dar introduce latență la fiecare commit. Cu 12 km și o fibră bună, latența adăugată era de 1-2 milisecunde — acceptabilă. Dar la 100 km ar fi fost 5-8 ms, și pe o aplicație cu mii de commit-uri pe secundă, încetinirea s-ar fi simțit. De aceea am ales modul `MaxPerformance` (asincron) ca implicit, acceptând posibilitatea teoretică de a pierde câteva secunde de tranzacții în caz de dezastru total. Pentru acel client, pierderea a cinci secunde de date era infinit mai bună decât pierderea a zece ore.

**Fișierul de parole.** Fișierul de parole al utilizatorului `SYS` trebuie să fie identic pe primar și standby. Dacă îl schimbi pe unul și nu pe celălalt, redo transport-ul se oprește silențios. Nicio eroare evidentă, doar un gap care crește. Am descoperit asta după o oră de debugging într-o duminică seara.

**Tablespace-urile temporare.** Standby-ul nu replică tablespace-urile temporare. Dacă deschizi standby-ul în citire pentru rapoarte (Active Data Guard), trebuie să creezi manual tablespace-urile temporare, altfel query-urile cu sort sau hash join eșuează cu erori care n-au nicio legătură cu problema reală.

```sql
-- Pe standby-ul deschis în mod read-only
ALTER TABLESPACE TEMP ADD TEMPFILE SIZE 2G AUTOEXTEND ON;
```

**Patch-urile.** Primarul și standby-ul trebuie să fie la același nivel de patch-uri. Dacă aplici o PSU pe primar fără să o aplici pe standby, redo-ul ar putea conține structuri pe care standby-ul nu le poate interpreta. Switchover-ul va funcționa, dar după aceea ai putea avea corupții silențioase. Procedura corectă este: patch pe standby mai întâi, switchover, patch pe vechiul primar (acum standby), switchback.

## Cifrele

La șase luni de la implementare, bilanțul era clar:

| Metrică | Înainte | După |
|---------|---------|------|
| RPO (Recovery Point Objective) | ~10 ore (backup nocturn) | < 5 secunde |
| RTO (Recovery Time Objective) | 6+ ore (restore din backup) | < 1 minut (switchover) |
| Disponibilitate rapoarte în paralel | Nu | Da (Active Data Guard) |
| Cost infrastructură suplimentară | — | 1 server + linie dedicată |
| Teste de switchover efectuate | 0 | 6 (unul pe lună) |

Costul total al proiectului — server, licențe, linie dedicată, implementare — era aproximativ un sfert din cât costase acea singură zi de oprire. Nu în termeni tehnici. În termeni de polițe neemise, daune neprocesate, clienți neserviți.

## Ce am învățat

Disaster recovery nu este o problemă tehnică. Este o problemă de percepție a riscului. Cât timp baza de date funcționează, DR-ul este o cheltuială. Când baza de date se oprește, DR-ul este o investiție care trebuia făcută cu șase luni înainte.

Nu poți convinge un CEO cu o diagramă arhitecturală. Poți doar să aștepți ca dezastrul să se întâmple și apoi să fii pregătit cu soluția. E cinic, dar așa funcționează în nouăzeci la sută din cazuri.

Singurul lucru pe care îl poți face dinainte este să documentezi riscul, să pui în scris că l-ai semnalat, și să ții proiectul pregătit în sertar. Eu propusesem acel proiect cu optsprezece luni înainte. Fusese pus deoparte cu un "revenim anul viitor."

Anul viitor a sosit într-o dimineață de miercuri din noiembrie, la 8:47.
