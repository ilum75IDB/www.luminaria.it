---
title: "Oracle de la On-Premises la Cloud: Strategie, Planificare și Cutover"
description: "Un Oracle 19c Enterprise cu RAC, Data Guard și 2 TB de date. Trei luni pentru a muta totul în OCI fără a pierde nici măcar o tranzacție. De la analiza licențelor până la cutover-ul peste noapte, povestea unei migrații reale."
date: "2026-04-28T10:00:00+01:00"
draft: false
translationKey: "oracle_cloud_migration"
tags: ["migration", "cloud", "oci", "data-guard", "architecture", "licensing"]
categories: ["oracle"]
image: "oracle-cloud-migration.cover.jpg"
---

Săptămâna trecută, un coleg mi-a scris: „Trebuie să mut Oracle în cloud — cât durează?” I-am răspuns cu o întrebare: „Știi exact câte funcționalități Enterprise Edition folosești cu adevărat?” Liniște.

Se repetă de fiecare dată același scenariu. Cineva din management decide că a venit momentul pentru cloud — pentru că expiră contractul de hosting, pentru că CFO-ul a citit un raport Gartner, pentru că noul CTO vrea modernizare. Și prima idee care apare este: lift-and-shift. Luăm ce avem și mutăm. Trei luni, buget aprobat, să-i dăm drumul.

Problema este că Oracle nu e o aplicație pe care o pui într-un container și o muți. Este un ecosistem: licențe, dependențe, configurații de kernel, conexiuni de rețea prin firewall-uri și VPN-uri. Dacă îl muți fără să-l înțelegi, ajungi în cloud cu aceleași probleme — și, de obicei, cu unele noi.

## Clientul și contextul

Proiectul era pentru o companie de producție din nordul Italiei — una dintre acelea care merg bine financiar, dar cu un departament IT foarte mic. Patru oameni pentru tot: ERP, sisteme interne, tot.

Oracle 19c Enterprise Edition rula pe un RAC cu două noduri, cu Data Guard replicând într-un site secundar la 20 km distanță. Aproximativ 2 TB de date, cam 200 de utilizatori simultan la orele de vârf și un batch nocturn care alimenta data warehouse-ul.

Providerul de hosting anunțase că nu mai prelungește contractul decât cu +40%. Managementul a decis: mergem în cloud. Trei luni pentru livrare, deadline fix.

Când am ajuns, planul era deja făcut: lift-and-shift pe AWS. Integratorul propusese EC2, EBS și gata. În Excel totul arăta impecabil. Două rânduri: cost actual vs cost viitor. Costul viitor mai mic. Toată lumea mulțumită.

## De ce am spus nu la AWS

Primul lucru pe care l-am cerut a fost raportul de licențe. Oracle 19c Enterprise Edition cu opțiunile RAC, Data Guard, Partitioning și Advanced Compression. Pe hardware on-premises cu un contract de suport direct cu Oracle.

Aici lucrurile se complică. Oracle are o politică de licențiere pentru cloud care nu e deloc intuitivă. Pe AWS, fiecare vCPU contează ca jumătate de procesor în scopul licențierii. Două noduri RAC pe EC2 cu, să zicem, 8 vCPU fiecare înseamnă 8 licențe de procesor. Cu Enterprise Edition plus opțiunile active, factura de licențe explodează. Și Oracle, când face audit — iar asta face — nu se uită la ce scrie în contractul providerului cloud. Se uită la ce rulează efectiv pe servere.

Pe OCI — Oracle Cloud Infrastructure — situația e diferită. Oracle recunoaște propriile OCPU cu un raport 1:1, și mai ales oferă programul BYOL (Bring Your Own License) care permite reutilizarea licențelor on-premises existente. Clientul plătise deja acele licențe. Mutarea lor pe OCI nu costa nimic în plus. Mutarea pe AWS însemna să le cumpere din nou sau să riște un audit.

Am pregătit un tabel comparativ cu trei scenarii: AWS cu licențe noi, AWS cu riscul de audit, OCI cu BYOL. Cifrele vorbeau de la sine. Managementul și-a schimbat părerea în jumătate de oră.

## Evaluarea: două săptămâni care valorează șase

Înainte să atingem ceva, am cerut două săptămâni pentru un assessment complet. Nu e o fază pe care o poți sări. Am învățat asta pe pielea mea într-un proiect anterior, unde am descoperit la jumătatea migrației că baza de date folosea Advanced Queuing cu proceduri PL/SQL care depindeau de o adresă IP hardcodată. Două zile de downtime pentru ceva ce puteam descoperi în cinci minute cu un grep.

Assessment-ul a acoperit patru zone.

**Funcționalități utilizate.** Am rulat Database Feature Usage Report de la Oracle (`DBMS_FEATURE_USAGE_INTERNAL`) pentru a înțelege care opțiuni Enterprise Edition erau efectiv active. RAC-ul era evident, Data Guard la fel. Dar Partitioning era folosit doar pe trei tabele, iar Advanced Compression fusese activat cu ani în urmă de un consultant și nimeni nu mai știa dacă mai era necesar. Am verificat: tabelele comprimate erau toate în arhiva istorică, lucruri care se citeau o dată pe an pentru auditori. Advanced Compression putea fi dezactivat fără niciun impact.

**Dependențe externe.** Baza de date primea date de la patru sisteme sursă prin DB link-uri, două dintre care indicau către baze de date MySQL pe servere din același data center. Existau și apeluri HTTP outbound din proceduri PL/SQL către un API REST intern. Toate trebuiau să continue să funcționeze după migrare, ceea ce însemna VPN site-to-site sau FastConnect între OCI și data center-ul on-premises.

**Rețea și latență.** Am măsurat latența între data center și regiunea OCI cea mai apropiată (Frankfurt). Cu un test susținut pe tnsping, round-trip-ul era de 12 milisecunde. Acceptabil pentru query-uri interactive, dar batch-ul nocturn făcea un join masiv prin DB link cu un MySQL remote — iar acolo, 12 milisecunde înmulțite cu milioane de rânduri însemnau ore în plus de procesare. Soluția a fost simplă: replicarea datelor MySQL într-o staging table pe Oracle înainte de lansarea batch-ului. Un pas în plus, dar batch-ul a trecut de la șase ore la două.

**Sizing.** Am analizat rapoartele AWR din ultimele patru săptămâni pentru a înțelege profilul real de încărcare. Vârful de CPU era la 35% pe cele două noduri RAC, memoria utilizată nu depășea niciodată 48 GB. Pe OCI am dimensionat două instanțe VM.Standard.E4.Flex cu 16 OCPU și 256 GB RAM fiecare pentru RAC, plus o a treia pentru standby-ul Data Guard. Storage pe Block Volumes cu performance tier echilibrat — 60 IOPS per GB, suficient pentru profilul I/O măsurat.

## Strategia de migrare: Data Guard, nu Data Pump

Când vine vorba de migrare Oracle, opțiunile principale sunt trei: Data Pump (export/import logic), Zero Downtime Migration (ZDM) și Data Guard.

Data Pump era exclus. Două terabyte de date cu export logic înseamnă ore de export, ore de transfer, ore de import. Și în tot acest timp baza de date sursă trebuie să rămână oprită, sau te trezești cu date inconsistente. Pentru o companie de producție care lucrează în trei schimburi, oprirea bazei de date o zi întreagă nu era o opțiune.

ZDM este instrumentul pe care Oracle îl propune pentru migrările către OCI. Funcționează, dar adaugă un layer de automatizare peste Data Guard și Data Pump. Pe o infrastructură cu RAC și configurații non-standard — cum ar fi DB link-urile cross-engine — prefer să am control direct.

Strategia a fost: configurarea Data Guard între RAC-ul on-premises și o instanță standby pe OCI, lăsăm să se sincronizeze, apoi facem cutover-ul cu un switchover controlat. Downtime estimat: sub o oră. Downtime efectiv: patruzeci și două de minute.

### Configurarea Data Guard cross-site

Partea complicată nu a fost Data Guard în sine — ăsta îl configurăm în fiecare săptămână. Partea complicată a fost să-l facă să funcționeze prin rețea. Data Guard are nevoie de un canal de redo transport între primary și standby, iar acel canal trebuie să fie fiabil și cu latență previzibilă.

Am configurat un tunel VPN site-to-site între data center și OCI, cu o bandă dedicată de 500 Mbps. Rata medie de generare a redo-ului era de 15 MB pe minut — confortabil în bugetul de bandă. Dar am vrut să testez cel mai rău caz: în timpul batch-ului nocturn, redo-ul ajungea la 180 MB pe minut. Și ăsta trecea, dar cu un transport lag care urca la 45 de secunde. Acceptabil pentru un Data Guard în modul Maximum Performance.

Configurația broker-ului a fost standard:

    DGMGRL> CREATE CONFIGURATION dg_migration AS
             PRIMARY DATABASE IS prod_rac
             CONNECT IDENTIFIER IS prod_rac;

    DGMGRL> ADD DATABASE oci_standby AS
             CONNECT IDENTIFIER IS oci_standby
             MAINTAINED AS PHYSICAL;

    DGMGRL> ENABLE CONFIGURATION;

Prima sincronizare completă a durat 14 ore — două terabyte printr-un VPN la 500 Mbps dau exact atât. După sincronizarea inițială, Data Guard a menținut standby-ul aliniat cu un apply lag mediu de 3 secunde.

## Cutover-ul: o noapte, un plan, zero surprize

Cutover-ul era planificat pentru o sâmbătă seara. Am pregătit un runbook de 47 de pași — da, patruzeci și șapte. Fiecare pas cu timpul estimat, comanda exactă, criteriul de succes și procedura de rollback. Pentru că dacă ceva merge prost la trei dimineața, nu vrei să improvizezi.

Secvența critică:

1. **22:00** — Oprire aplicație. Verificat că toate sesiunile active s-au terminat.
2. **22:15** — Ultimul check al transport lag-ului: 2 secunde. Apply lag: 0.
3. **22:20** — Switchover prin Data Guard Broker:

        DGMGRL> SWITCHOVER TO oci_standby;

4. **22:22** — Switchover-ul s-a completat în 98 de secunde. Noul primary era pe OCI.
5. **22:25** — Actualizarea connection string-urilor în connection pool-ul aplicației. SCAN listener-ul pe OCI era deja configurat.
6. **22:30** — Test de conectivitate: login, query-uri pe tabele critice, insert de test.
7. **22:45** — Test batch job: execuția unui mini-batch pe un eșantion de date.
8. **23:00** — Deschidere graduală pentru utilizatorii din schimbul de noapte.
9. **23:30** — Monitorizare: snapshot-uri AWR la fiecare 15 minute în loc de cele 60 implicite.

La miezul nopții și jumătate totul funcționa. Downtime-ul real — din momentul în care ultimul utilizator s-a deconectat până în momentul în care primul s-a reconectat — a fost de 42 de minute.

## După migrare: lucrurile pe care niciun plan nu le prevede

Prima săptămână după cutover e cea care separă o migrare reușită de una „tehnic reușită, dar toată lumea se plânge”. Au apărut trei lucruri.

**Timezone-urile.** VM-urile pe OCI foloseau UTC, baza de date on-premises folosea Europe/Rome. Procedurile PL/SQL care calculau date cu `SYSDATE` returnau ore greșite. `ALTER DATABASE SET TIME_ZONE` necesită restart-ul bazei de date și reconstruirea coloanelor `TIMESTAMP WITH LOCAL TIME ZONE`. Am descoperit asta luni dimineață, când responsabilul de logistică m-a sunat spunând că comenzile aveau date „din viitor”. Rezolvat în două ore, dar se putea evita dacă includeam timezone-ul în runbook.

**TLS-ul.** Apelurile HTTP outbound din procedurile PL/SQL foloseau `UTL_HTTP` cu wallet Oracle pentru certificate. Wallet-ul fusese configurat cu certificatele data center-ului. Pe OCI, certificatele CA erau diferite. Procedurile eșuau cu `ORA-29024: Certificate validation failure`. A trebuit să recreez wallet-ul importând noile certificate CA și să-l redistribui.

**Scheduler-ul.** Job-urile Oracle Scheduler (`DBMS_SCHEDULER`) aveau window-uri și schedule-uri bazate pe timezone-ul bazei de date. După fix-ul de timezone, ferestrele de mentenanță s-au realiniat, dar trei job-uri care foloseau `SYSTIMESTAMP` direct în codul PL/SQL au continuat să pornească cu o oră mai devreme timp de o săptămână — până le-am găsit și corectat pe rând.

## Costurile reale: dincolo de foaia Excel

La trei luni după migrare, am pregătit un raport de costuri reale pentru management. Comparația cu vechiul hosting a fost instructivă.

| Element | On-premises | OCI |
|---------|-------------|-----|
| Compute (RAC 2 noduri + standby) | inclus în contractul de hosting | €4.200/lună |
| Storage (2 TB + backup) | inclus | €680/lună |
| Networking (VPN + egress) | €200/lună | €350/lună |
| Licențiere Oracle | €18.000/an (suport) | €0 (BYOL) |
| Hosting/colocare | €8.500/lună | €0 |
| Total anual | ~€120.000 | ~€63.000 |

Economia era acolo, și era semnificativă. Dar cifra care l-a impresionat cel mai mult pe CFO nu era totalul: era costul networking-ului. VPN-ul site-to-site și traficul egress din OCI costau aproape dublu față de înainte. E un rând care în ofertele cloud e întotdeauna subestimat.

Și apoi era costul ascuns: timpul meu. Două luni de consultanță pentru assessment, planificare, migrare și tuning post-migrare. Costul ăla nu apărea în comparația lunară, dar era real.

## Ce am învățat (din nou)

Fiecare migrare te învață ceva, chiar și când crezi că le-ai văzut pe toate.

Licențierea Oracle în cloud e un câmp minat. Nu e suficient să citești documentația: trebuie să vorbești cu Oracle, să obții confirmări scrise și să ții evidența la tot. Un audit post-migrare poate transforma o economie într-o catastrofă.

Assessment-ul nu e opțional. Cele două săptămâni de la început au prevenit cel puțin trei probleme care ar fi necesitat săptămâni de remediere după migrare. Raportul de funcționalități utilizate, harta dependențelor externe, testele de latență — sunt lucruri plicticoase, dar sunt diferența între un cutover de 42 de minute și unul de 42 de ore.

Data Guard cross-site e cea mai curată strategie de migrare pentru Oracle. Îți oferă o plasă de siguranță permanentă: dacă ceva merge prost, faci switchback și ești la punctul de plecare. Cu Data Pump, dacă ceva merge prost la jumătatea importului, o iei de la zero.

Și timezone-ul. Doamne, timezone-ul. Pune-l în capul listei.

------------------------------------------------------------------------

## Glosar

**[OCI](/ro/glossary/oci/)** — Oracle Cloud Infrastructure, platforma cloud a Oracle. Pentru bazele de date Oracle oferă avantaje semnificative de licențiere prin programul BYOL și raportul 1:1 pentru OCPU.

**[BYOL](/ro/glossary/byol/)** — Bring Your Own License, program care permite reutilizarea licențelor Oracle existente on-premises în cloud OCI fără costuri suplimentare de licențiere.

**[RAC](/ro/glossary/rac/)** — Real Application Clusters, tehnologie Oracle care permite mai multor instanțe să acceseze simultan aceeași bază de date, oferind disponibilitate ridicată și scalabilitate orizontală.

**[Data Guard](/ro/glossary/data-guard/)** — tehnologie Oracle pentru replicarea în timp real a unei baze de date către unul sau mai multe servere standby, asigurând disponibilitate ridicată și recuperare în caz de dezastru.

**[ZDM](/ro/glossary/zdm/)** — Zero Downtime Migration, tool Oracle pentru migrarea bazelor de date către OCI, care combină Data Guard și Data Pump într-un strat de orchestrare automatizată.

**[Switchover](/ro/glossary/switchover/)** — operațiune planificată în Data Guard care inversează rolurile între primary și standby fără pierdere de date. Spre deosebire de failover, este controlată și reversibilă.

**[AWR](/ro/glossary/awr/)** — Automatic Workload Repository, instrument integrat în Oracle Database pentru colectarea și analiza statisticilor de performanță.

**[Transport Lag](/ro/glossary/transport-lag/)** — întârzierea în transmiterea redo log-urilor de la baza de date primary către standby în configurațiile Data Guard. Indicator esențial pentru sănătatea replicării.

**[SCAN Listener](/ro/glossary/scan-listener/)** — Single Client Access Name, componentă Oracle RAC care oferă un punct unic de acces către cluster, distribuind automat conexiunile între noduri.

**[Cutover](/ro/glossary/cutover/)** — momentul critic într-o migrare în care sistemul de producție este mutat definitiv de pe infrastructura veche pe cea nouă.
