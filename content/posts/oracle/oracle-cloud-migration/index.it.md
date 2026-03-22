---
title: "Oracle da on-premises a cloud: strategia, pianificazione e cutover"
description: "Un Oracle 19c Enterprise con RAC, Data Guard e 2 TB di dati. Tre mesi per migrare tutto su OCI senza perdere una transazione. Dal licensing assessment al cutover notturno, la cronaca di una migrazione reale."
date: "2026-04-28T10:00:00+01:00"
draft: false
translationKey: "oracle_cloud_migration"
tags: ["migration", "cloud", "oci", "data-guard", "architecture", "licensing"]
categories: ["oracle"]
image: "oracle-cloud-migration.cover.jpg"
---

La settimana scorsa un collega mi ha scritto: "Devo portare Oracle in cloud, quanto ci vuole?" Gli ho risposto con una domanda: "Sai quante feature usi della Enterprise Edition?" Silenzio.

È la scena che si ripete ogni volta. Qualcuno in alto decide che bisogna andare in cloud — perché il contratto dell'hosting scade, perché il CFO ha letto un report di Gartner, perché il CTO nuovo vuole modernizzare. E la prima cosa che salta fuori è: facciamo un lift-and-shift, prendiamo quello che c'è e lo spostiamo. Tre mesi, budget approvato, via.

Il problema è che Oracle non è un'applicazione che impacchetti in un container e spedisci. È un ecosistema con licenze, dipendenze, configurazioni kernel, connessioni di rete che attraversano firewall e VPN. Spostarlo senza capirlo prima significa ritrovarsi in cloud con gli stessi problemi di prima — più qualcuno di nuovo.

## Il cliente e il contesto

Il progetto era per un'azienda manifatturiera del nord Italia, una di quelle che fattura parecchio ma ha un IT snello — quattro persone per tutto, dal gestionale agli ERP. Il loro Oracle 19c Enterprise Edition girava su due nodi RAC con Data Guard verso un sito secondario a venti chilometri. Due terabyte di dati, circa duecento utenti concorrenti nelle ore di punta, un batch notturno che alimentava il data warehouse.

L'hosting provider aveva comunicato che il contratto sarebbe scaduto entro sei mesi e il rinnovo prevedeva un aumento del 40%. Il management aveva deciso: migriamo su cloud. Tre mesi di tempo, il progetto doveva chiudersi prima della scadenza contrattuale.

Quando sono arrivato, il piano era già scritto: lift-and-shift su AWS. Il system integrator di turno aveva proposto un paio di istanze EC2, un po' di storage EBS e via. Sul foglio Excel sembrava tutto semplice. Due righe: costo attuale, costo futuro. Il costo futuro era più basso. Tutti contenti.

## Perché ho detto no ad AWS

La prima cosa che ho fatto è stata chiedere il report delle licenze. Oracle 19c Enterprise Edition con le opzioni RAC, Data Guard, Partitioning e Advanced Compression. Su hardware on-premises con un contratto di supporto diretto con Oracle.

Qui è dove la cosa si complica. Oracle ha una policy di licensing per il cloud che non è esattamente intuitiva. Su AWS, ogni vCPU conta come mezzo processore ai fini del licensing. Due nodi RAC su EC2 con, diciamo, 8 vCPU ciascuno significano 8 processor licenses. Con Enterprise Edition più le opzioni attive, il conto delle licenze esplode. E Oracle, quando fa audit — e li fa — non guarda cosa c'è scritto nel contratto del cloud provider. Guarda cosa gira sui server.

Su OCI — Oracle Cloud Infrastructure — la situazione è diversa. Oracle riconosce le proprie OCPU con un rapporto 1:1, e soprattutto offre il programma BYOL (Bring Your Own License) che permette di riutilizzare le licenze on-premises esistenti. Il cliente aveva già pagato quelle licenze. Spostarle su OCI non costava nulla in più. Spostarle su AWS significava ricomprarle o rischiare un audit.

Ho preparato un foglio di confronto con tre scenari: AWS con nuove licenze, AWS con il rischio audit, OCI con BYOL. I numeri parlavano da soli. Il management ha cambiato idea in mezz'ora.

## L'assessment: due settimane che ne valgono sei

Prima di toccare qualsiasi cosa, ho chiesto due settimane per un assessment completo. Non è una fase che si può saltare. L'ho imparato a mie spese su un progetto precedente dove abbiamo scoperto a metà migrazione che il database usava Advanced Queuing con procedure PL/SQL che dipendevano da un indirizzo IP hardcoded. Due giorni di fermo per una cosa che si scopriva in cinque minuti con un grep.

L'assessment ha coperto quattro aree.

**Feature in uso.** Ho eseguito il Database Feature Usage Report di Oracle (`DBMS_FEATURE_USAGE_INTERNAL`) per capire quali opzioni della Enterprise Edition erano effettivamente attive. Il RAC era ovvio, il Data Guard pure. Ma il Partitioning lo usavano solo su tre tabelle, e l'Advanced Compression era stato attivato anni prima da un consulente e nessuno sapeva se servisse ancora. Ho verificato: le tabelle compresse erano tutte nell'archivio storico, roba che si leggeva una volta l'anno per gli auditor. L'Advanced Compression si poteva disattivare senza impatto.

**Dipendenze esterne.** Il database riceveva dati da quattro sistemi sorgente tramite DB link, due dei quali puntavano a database MySQL su server nello stesso data center. C'erano anche chiamate HTTP in uscita da procedure PL/SQL verso un'API REST interna. Tutto questo doveva continuare a funzionare dopo la migrazione, il che significava VPN site-to-site o FastConnect tra OCI e il data center on-premises.

**Rete e latenza.** Ho misurato la latenza tra il data center e la region OCI più vicina (Frankfurt). Con un test sustained su tnsping, il round-trip era di 12 millisecondi. Accettabile per le query interattive, ma il batch notturno faceva un join massiccio via DB link con un MySQL remoto — e lì 12 millisecondi moltiplicati per milioni di righe significavano ore in più. La soluzione è stata semplice: replicare i dati del MySQL in una staging table su Oracle prima di lanciare il batch. Un passaggio in più, ma il batch è passato da sei ore a due.

**Sizing.** Ho analizzato gli AWR report delle ultime quattro settimane per capire il profilo di carico reale. Il picco di CPU era al 35% sui due nodi RAC, la memoria usata non superava mai i 48 GB. Su OCI ho dimensionato due VM.Standard.E4.Flex con 16 OCPU e 256 GB di RAM ciascuna per il RAC, più una terza per il Data Guard standby. Storage su Block Volume con performance tier bilanciato — 60 IOPS per GB, sufficiente per il profilo I/O misurato.

## La strategia di migrazione: Data Guard, non Data Pump

Quando si parla di migrazione Oracle, le opzioni principali sono tre: Data Pump (export/import logico), Zero Downtime Migration (ZDM), e Data Guard.

Data Pump era fuori discussione. Due terabyte di dati con export logico significano ore di export, ore di trasferimento, ore di import. E durante tutto quel tempo il database sorgente deve restare fermo, o ti ritrovi con dati inconsistenti. Per un'azienda manifatturiera che lavora su tre turni, fermare il database per un giorno intero non era un'opzione.

ZDM è lo strumento che Oracle propone per le migrazioni verso OCI. Funziona, ma aggiunge un layer di automazione sopra Data Guard e Data Pump. Su un'infrastruttura con RAC e configurazioni non standard — come i DB link cross-engine — preferisco avere il controllo diretto.

La strategia è stata: configurare il Data Guard tra il RAC on-premises e un'istanza standby su OCI, lasciare che si sincronizzi, e poi fare il cutover con un switchover controllato. Downtime previsto: meno di un'ora. Downtime effettivo: quarantadue minuti.

### La configurazione del Data Guard cross-site

Il pezzo complicato non è stato il Data Guard in sé — quello lo configuriamo ogni settimana. Il pezzo complicato è stato farlo funzionare attraverso la rete. Il Data Guard ha bisogno di un canale di redo transport tra primary e standby, e quel canale deve essere affidabile e con latenza prevedibile.

Ho configurato un tunnel VPN site-to-site tra il data center e OCI, con una bandwidth dedicata di 500 Mbps. Il redo generate rate medio era di 15 MB al minuto — ampiamente dentro il budget di banda. Ma ho voluto testare il worst case: durante il batch notturno il redo arrivava a 180 MB al minuto. Anche quello passava, ma con un transport lag che saliva a 45 secondi. Accettabile per un Data Guard in modalità Maximum Performance.

La configurazione del broker è stata standard:

    DGMGRL> CREATE CONFIGURATION dg_migration AS
             PRIMARY DATABASE IS prod_rac
             CONNECT IDENTIFIER IS prod_rac;

    DGMGRL> ADD DATABASE oci_standby AS
             CONNECT IDENTIFIER IS oci_standby
             MAINTAINED AS PHYSICAL;

    DGMGRL> ENABLE CONFIGURATION;

Il primo sync completo ha richiesto 14 ore — due terabyte su una VPN a 500 Mbps fanno esattamente quel conto. Dopo la sincronizzazione iniziale, il Data Guard ha mantenuto lo standby allineato con un apply lag medio di 3 secondi.

## Il cutover: una notte, un piano, zero sorprese

Il cutover era pianificato per un sabato notte. Ho preparato un runbook di 47 passi — sì, quarantasette. Ogni passo con il tempo stimato, il comando esatto, il criterio di successo e il rollback. Perché se qualcosa va storto alle tre di notte, non vuoi trovarti a improvvisare.

La sequenza critica:

1. **22:00** — Stop applicativo. Verificato che tutte le sessioni attive fossero terminate.
2. **22:15** — Ultimo check del transport lag: 2 secondi. Apply lag: 0.
3. **22:20** — Switchover via Data Guard Broker:

        DGMGRL> SWITCHOVER TO oci_standby;

4. **22:22** — Il switchover si è completato in 98 secondi. Il nuovo primary era su OCI.
5. **22:25** — Aggiornamento dei connection string nel connection pool dell'applicativo. Il SCAN listener su OCI era già configurato.
6. **22:30** — Test di connettività: login, query su tabelle critiche, insert di test.
7. **22:45** — Test del batch job: esecuzione di un mini-batch su un campione dati.
8. **23:00** — Apertura graduale agli utenti del turno di notte.
9. **23:30** — Monitoraggio: AWR snapshot ogni 15 minuti invece del default di 60.

A mezzanotte e mezza tutto girava. Il downtime reale — dal momento in cui l'ultimo utente si è disconnesso al momento in cui il primo si è riconnesso — è stato di 42 minuti.

## Dopo la migrazione: le cose che nessun piano prevede

La prima settimana dopo il cutover è quella che separa una migrazione riuscita da una migrazione "tecnicamente riuscita ma tutti si lamentano". Tre cose sono emerse.

**I timezone.** Le VM su OCI usavano UTC, il database on-premises usava Europe/Rome. Le procedure PL/SQL che calcolavano date con `SYSDATE` restituivano orari sbagliati. L'`ALTER DATABASE SET TIME_ZONE` richiede il riavvio del database e la ricostruzione delle colonne `TIMESTAMP WITH LOCAL TIME ZONE`. L'ho scoperto il lunedì mattina quando il responsabile della logistica mi ha chiamato dicendo che gli ordini avevano date "del futuro". Fix in due ore, ma poteva essere evitato se avessi incluso il timezone nel runbook.

**Il TLS.** Le chiamate HTTP in uscita dalle procedure PL/SQL usavano `UTL_HTTP` con wallet Oracle per i certificati. Il wallet era stato configurato con i certificati del data center. Su OCI, i certificati CA erano diversi. Le procedure fallivano con `ORA-29024: Certificate validation failure`. Ho dovuto ricreare il wallet importando i nuovi certificati CA e ridistribuirlo.

**Lo scheduler.** I job Oracle Scheduler (`DBMS_SCHEDULER`) avevano window e schedule basati sul timezone del database. Dopo il fix del timezone, le finestre di manutenzione si sono riallineate, ma tre job che usavano `SYSTIMESTAMP` direttamente nel codice PL/SQL hanno continuato a partire un'ora in anticipo per una settimana — fino a quando non li ho trovati e corretti uno per uno.

## I costi reali: oltre il foglio Excel

Dopo tre mesi dalla migrazione, ho preparato un report dei costi reali per il management. Il confronto con il vecchio hosting è stato istruttivo.

| Voce | On-premises | OCI |
|------|-------------|-----|
| Compute (RAC 2 nodi + standby) | incluso nel contratto hosting | €4.200/mese |
| Storage (2 TB + backup) | incluso | €680/mese |
| Networking (VPN + egress) | €200/mese | €350/mese |
| Licensing Oracle | €18.000/anno (supporto) | €0 (BYOL) |
| Hosting/housing | €8.500/mese | €0 |
| Totale annuo | ~€120.000 | ~€63.000 |

Il risparmio c'era, ed era significativo. Ma il numero che ha colpito di più il CFO non era il totale: era il costo del networking. La VPN site-to-site e il traffico egress da OCI costavano quasi il doppio di prima. È una voce che nei preventivi cloud viene sempre sottostimata.

E poi c'era il costo nascosto: il mio tempo. Due mesi di consulenza per l'assessment, la pianificazione, la migrazione e il tuning post-migrazione. Quel costo non appariva nel confronto mensile, ma era reale.

## Cosa ho imparato (di nuovo)

Ogni migrazione insegna qualcosa, anche quando pensi di averle viste tutte.

Il licensing Oracle in cloud è un campo minato. Non è sufficiente leggere la documentazione: bisogna parlare con Oracle, ottenere conferme scritte, e tenere traccia di tutto. Un audit post-migrazione può trasformare un risparmio in una catastrofe.

L'assessment non è un optional. Quelle due settimane iniziali hanno evitato almeno tre problemi che avrebbero richiesto settimane di fix a migrazione avvenuta. Il report delle feature in uso, la mappa delle dipendenze esterne, i test di latenza — sono cose noiose, ma sono la differenza tra un cutover da 42 minuti e uno da 42 ore.

Il Data Guard cross-site è la strategia di migrazione più pulita per Oracle. Ti dà una rete di sicurezza permanente: se qualcosa va storto, fai switchback e sei al punto di partenza. Con Data Pump, se qualcosa va storto a metà import, ricominci da zero.

E il timezone. Dio, il timezone. Mettetelo in cima alla checklist.

------------------------------------------------------------------------

## Glossario

**[OCI](/it/glossary/oci/)** — Oracle Cloud Infrastructure, la piattaforma cloud di Oracle. Per i database Oracle offre vantaggi significativi in termini di licensing grazie al programma BYOL e al rapporto 1:1 delle OCPU.

**[BYOL](/it/glossary/byol/)** — Bring Your Own License, programma che permette di riutilizzare le licenze Oracle on-premises nel cloud OCI senza costi aggiuntivi di licensing.

**[RAC](/it/glossary/rac/)** — Real Application Clusters, tecnologia Oracle che permette a più istanze di accedere contemporaneamente allo stesso database, garantendo alta disponibilità e scalabilità orizzontale.

**[Data Guard](/it/glossary/data-guard/)** — Tecnologia Oracle per la replica in tempo reale di un database su uno o più server standby, garantendo alta disponibilità e disaster recovery.

**[ZDM](/it/glossary/zdm/)** — Zero Downtime Migration, strumento Oracle per automatizzare le migrazioni verso OCI combinando Data Guard e Data Pump sotto un layer di orchestrazione.

**[Switchover](/it/glossary/switchover/)** — Operazione pianificata di Data Guard che inverte i ruoli tra primary e standby senza perdita di dati. A differenza del failover, è reversibile e controllata.

**[AWR](/it/glossary/awr/)** — Automatic Workload Repository, strumento diagnostico integrato in Oracle Database per la raccolta e l'analisi delle statistiche di performance.

**[Transport Lag](/it/glossary/transport-lag/)** — Ritardo nella trasmissione dei redo log dal database primary allo standby in una configurazione Data Guard. Un indicatore critico della salute della replica.

**[SCAN Listener](/it/glossary/scan-listener/)** — Single Client Access Name, componente Oracle RAC che fornisce un unico punto di accesso al cluster, distribuendo automaticamente le connessioni tra i nodi disponibili.

**[Cutover](/it/glossary/cutover/)** — Momento critico di una migrazione in cui il sistema di produzione viene spostato definitivamente dalla vecchia alla nuova infrastruttura.
