---
title: "Da single instance a Data Guard: il giorno in cui il CEO ha capito il DR"
description: "Un database Oracle in produzione senza alcuna ridondanza. Un guasto disco che ha fermato tutto per sei ore. E la decisione del CEO di investire in un'architettura Active Data Guard con switchover automatico."
date: "2025-12-02T10:00:00+01:00"
draft: false
translationKey: "oracle_data_guard"
tags: ["data-guard", "disaster-recovery", "high-availability", "switchover", "architecture"]
categories: ["oracle"]
image: "oracle-data-guard.cover.jpg"
---

Il cliente era una media azienda nel settore assicurativo. Trecento dipendenti, un gestionale interno che girava su Oracle 19c, un solo server fisico in sala macchine al piano terra della sede. Nessuna replica. Nessuno standby. Nessun piano di disaster recovery.

Per cinque anni aveva funzionato tutto. E quando le cose funzionano, nessuno vuole spendere soldi per proteggersi da problemi che non si sono mai visti.

## Il giorno in cui si è fermato tutto

Un mercoledì mattina di novembre, alle 8:47, il disco del gruppo dati principale ha avuto un guasto fisico. Non un errore logico, non una corruzione recuperabile. Un guasto hardware. Il controller RAID ha perso due dischi contemporaneamente — uno già degradato da settimane senza che nessuno se ne accorgesse, l'altro ceduto di colpo.

Il database si è fermato. Le policy non si emettevano. I sinistri non si lavoravano. Il call center rispondeva ai clienti dicendo "problemi tecnici, richiamate più tardi."

Ho ricevuto la chiamata alle 9:15. Quando sono arrivato in sede, il sistemista stava già cercando dischi compatibili. Li ha trovati nel primo pomeriggio. Tra sostituzione, rebuild del RAID e recovery del database dal backup della notte precedente, il sistema è tornato operativo alle 15:20.

Sei ore e mezza di fermo totale. E la perdita di tutte le transazioni dalle 23:00 della sera prima alle 8:47 del mattino — circa dieci ore di dati, perché il backup era solo notturno e gli archived log non venivano copiati su un'altra macchina.

Il CEO quella sera ha mandato una mail a tutta l'azienda. Il giorno dopo mi ha chiamato: "Cosa dobbiamo fare perché non succeda mai più?"

## Il disegno

La risposta era semplice nel concetto, meno nella realizzazione: serviva un secondo database, sincronizzato in tempo reale, pronto a prendere il posto del primario in caso di guasto.

Oracle Active Data Guard fa esattamente questo. Un database primario che genera redo log, e uno standby che li riceve e li applica continuamente. Se il primario muore, lo standby diventa primario. Se tutto va bene, lo standby si può anche usare in sola lettura — per report, per backup, per alleggerire il carico.

Ho disegnato un'architettura a due nodi:

- **Primario** (`oraprod1`): il server esistente, con i dischi nuovi, nella sede principale
- **Standby** (`oraprod2`): un nuovo server identico, nel data center del provider di hosting, a 12 km di distanza

La scelta della distanza non era casuale. Abbastanza lontano da sopravvivere a un evento localizzato (incendio, allagamento, blackout prolungato), abbastanza vicino da permettere la replica sincrona senza latenza percepibile.

## La configurazione

### Preparazione del primario

Il primo passo è stato verificare che il primario fosse in `ARCHIVELOG` mode e con `FORCE LOGGING` attivo. Senza questi due prerequisiti, Data Guard non ha niente da replicare.

```sql
-- Verifica archivelog mode
SELECT log_mode FROM v$database;

-- Se necessario, attivare
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;

-- Force logging: impedisce operazioni NOLOGGING
ALTER DATABASE FORCE LOGGING;
```

Il `FORCE LOGGING` è fondamentale. Senza di esso, qualsiasi operazione con clausola `NOLOGGING` — un `CREATE TABLE AS SELECT`, un `ALTER INDEX REBUILD` — non genera redo e crea buchi nella replica. L'ho visto succedere tre volte in carriera. La terza volta ho deciso che `FORCE LOGGING` si attiva sempre, senza eccezioni.

### Standby redo log

Sul primario ho creato gli standby redo log — gruppi dedicati che verranno usati quando (e se) questo server diventerà standby a seguito di uno switchover.

```sql
-- Standby redo log: n+1 rispetto ai redo log online
-- Se hai 3 gruppi online, crei 4 standby
ALTER DATABASE ADD STANDBY LOGFILE GROUP 4 SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 5 SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 6 SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 7 SIZE 200M;
```

La regola è n+1: se il primario ha tre gruppi di redo log, lo standby ne vuole quattro. Non è documentata in modo chiarissimo, ma l'ho imparata a spese mie — con tre gruppi uguali, sotto carico pesante lo standby può andare in stallo aspettando un gruppo libero.

### Configurazione di rete

Il `tnsnames.ora` su entrambi i nodi deve conoscere sia il primario che lo standby. La configurazione è simmetrica:

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

Il `listener.ora` sullo standby deve includere un'entry statica per il database, perché durante il restore lo standby non è ancora aperto e il listener non può registrarlo dinamicamente:

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

Il suffisso `_DGMGRL` serve al Data Guard Broker per identificare l'istanza. Senza questa entry statica, il broker non riesce a connettersi allo standby e le operazioni di switchover falliscono con errori criptici che ti fanno perdere mezza giornata.

### Creazione dello standby

Per la copia iniziale del database ho usato un `DUPLICATE` via RMAN attraverso la rete. Nessun backup su nastro, nessun trasferimento manuale di file. Diretto, dal primario allo standby:

```
-- Sul server standby, avviare l'istanza in NOMOUNT
STARTUP NOMOUNT PFILE='/u01/app/oracle/product/19c/dbs/initoraprod.ora';

-- Da RMAN, connesso a entrambi
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

Il `NOFILENAMECHECK` serve quando i path dei file sono identici su entrambe le macchine — stessa struttura di directory, stessa naming convention. Se i path differiscono, servono i parametri `DB_FILE_NAME_CONVERT` e `LOG_FILE_NAME_CONVERT`.

La copia ha richiesto circa tre ore per 400 GB attraverso una linea dedicata a 1 Gbps. Non velocissima, ma è un'operazione che si fa una volta sola.

### Data Guard Broker

Il Broker è il componente che gestisce la configurazione Data Guard in modo centralizzato e consente lo switchover con un singolo comando. Senza il Broker puoi fare tutto a mano, ma non vuoi farlo a mano quando il primario è appena caduto e il CEO ti sta chiamando ogni cinque minuti.

```sql
-- Sul primario
ALTER SYSTEM SET dg_broker_start=TRUE;

-- Sullo standby
ALTER SYSTEM SET dg_broker_start=TRUE;
```

Poi, da `DGMGRL` sul primario:

```
DGMGRL> CREATE CONFIGURATION dg_config AS
         PRIMARY DATABASE IS oraprod
         CONNECT IDENTIFIER IS ORAPROD1;

DGMGRL> ADD DATABASE oraprod_stby AS
         CONNECT IDENTIFIER IS ORAPROD2
         MAINTAINED AS PHYSICAL;

DGMGRL> ENABLE CONFIGURATION;
```

A quel punto, `SHOW CONFIGURATION` deve restituire:

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

La parola che vuoi vedere è `SUCCESS`. Qualsiasi altra cosa significa che c'è un problema di rete, di configurazione o di permessi da risolvere prima di andare avanti.

## Il primo switchover

Due settimane dopo il go-live dell'architettura, ho fatto il primo test di switchover. Di sabato mattina, con il gestionale chiuso, ma con il CEO presente — voleva vedere con i suoi occhi.

```
DGMGRL> SWITCHOVER TO oraprod_stby;
```

Un solo comando. Quarantadue secondi. Il primario è diventato standby, lo standby è diventato primario. Le applicazioni, configurate con il servizio corretto, si sono riconnesse automaticamente.

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

Poi abbiamo fatto lo switchback — ritorno al primario originale. Altri trentotto secondi. Tutto pulito.

Il CEO ha guardato lo schermo, ha guardato me, e ha detto: "Quarantadue secondi contro sei ore. Perché non l'abbiamo fatto prima?"

La risposta non gliel'ho data. La sapevamo entrambi.

## Quello che non ti dicono

La configurazione che ho descritto funziona. Ma ci sono cose che la documentazione Oracle non enfatizza abbastanza.

**Il gap di rete.** La replica sincrona (`SYNC`) garantisce zero data loss ma introduce latenza su ogni commit. Con 12 km e una buona fibra, la latenza aggiunta era di 1-2 millisecondi — accettabile. Ma a 100 km sarebbe stata 5-8 ms, e su un'applicazione con migliaia di commit al secondo il rallentamento si sarebbe sentito. Per questo ho scelto la modalità `MaxPerformance` (asincrona) come default, accettando la possibilità teorica di perdere qualche secondo di transazioni in caso di disastro totale. Per quel cliente, perdere cinque secondi di dati era infinitamente meglio di perderne dieci ore.

**Il password file.** Il password file dell'utente `SYS` deve essere identico su primario e standby. Se lo cambi su uno e non sull'altro, il redo transport si blocca silenziosamente. Nessun errore evidente, solo un gap che cresce. L'ho scoperto dopo un'ora di debugging una domenica sera.

**I temp tablespace.** Lo standby non replica i tablespace temporanei. Se apri lo standby in lettura per i report (Active Data Guard), devi creare manualmente i temp tablespace, altrimenti le query con sort o hash join falliscono con errori che non c'entrano niente con il vero problema.

```sql
-- Sullo standby aperto in read-only
ALTER TABLESPACE TEMP ADD TEMPFILE SIZE 2G AUTOEXTEND ON;
```

**I patch.** Primario e standby devono avere lo stesso livello di patch. Se applichi una PSU al primario senza applicarla allo standby, il redo potrebbe contenere strutture che lo standby non sa interpretare. Il switchover funzionerà, ma dopo potresti avere corruzioni silenziose. La procedura corretta è: patch allo standby prima, switchover, patch al vecchio primario (ora standby), switchback.

## I numeri

A sei mesi dall'implementazione, il bilancio era chiaro:

| Metrica | Prima | Dopo |
|---------|-------|------|
| RPO (Recovery Point Objective) | ~10 ore (backup notturno) | < 5 secondi |
| RTO (Recovery Time Objective) | 6+ ore (restore da backup) | < 1 minuto (switchover) |
| Disponibilità report in parallelo | No | Sì (Active Data Guard) |
| Costo infrastruttura aggiuntiva | — | 1 server + linea dedicata |
| Test di switchover eseguiti | 0 | 6 (uno al mese) |

Il costo totale del progetto — server, licenze, linea dedicata, implementazione — era circa un quarto di quanto era costata quella singola giornata di fermo. Non in termini tecnici. In termini di polizze non emesse, sinistri non lavorati, clienti non serviti.

## Quello che ho imparato

Il disaster recovery non è un problema tecnico. È un problema di percezione del rischio. Finché il database funziona, il DR è una spesa. Quando il database si ferma, il DR è un investimento che si doveva fare sei mesi prima.

Non puoi convincere un CEO con un disegno architetturale. Puoi solo aspettare che il disastro succeda e poi essere pronto con la soluzione. È cinico, ma è così che funziona nel novanta per cento dei casi.

L'unica cosa che puoi fare prima è documentare il rischio, mettere per iscritto che l'hai segnalato, e tenere pronto il progetto nel cassetto. Io quel progetto l'avevo proposto diciotto mesi prima. Era stato accantonato con un "ne riparliamo l'anno prossimo."

L'anno prossimo è arrivato un mercoledì mattina di novembre, alle 8:47.
