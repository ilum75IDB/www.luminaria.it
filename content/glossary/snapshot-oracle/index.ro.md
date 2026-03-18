---
title: "Snapshot (Oracle)"
description: "Captura punctuala a statisticilor de performanta preluata periodic de AWR si folosita pentru a genera rapoarte de diagnostic comparative."
translationKey: "glossary_snapshot_oracle"
articles:
  - "/posts/oracle/oracle-awr-ash"
---

**Snapshot** in Oracle este o captura punctuala a statisticilor de performanta ale bazei de date stocata in repository-ul AWR. Implicit Oracle genereaza un snapshot la fiecare 60 de minute si le pastreaza timp de 8 zile.

## Cum functioneaza

Fiecare snapshot inregistreaza sute de metrici: wait events, statistici SQL, metrici de memorie (SGA, PGA), I/O per datafile, statistici de sistem. Compararea a doua snapshot-uri genereaza raportul AWR, care arata ce s-a schimbat intre cele doua momente.

## Snapshot-uri manuale

In situatii de urgenta se poate genera un snapshot manual pentru a captura starea curenta:

    EXEC DBMS_WORKLOAD_REPOSITORY.create_snapshot;

Acest lucru este util cand vrei un punct de referinta imediat — de exemplu, inainte si dupa un deploy — fara a astepta ciclul automat.

## Gestionare

Snapshot-urile sunt accesibile prin view-ul `DBA_HIST_SNAPSHOT`. Retentia (cate zile sa le pastrezi) si intervalul (la cate minute sa le generezi) se configureaza cu:

    EXEC DBMS_WORKLOAD_REPOSITORY.modify_snapshot_settings(
      retention => 43200,   -- 30 zile in minute
      interval  => 30       -- la fiecare 30 minute
    );

## De ce sunt importante

Fara snapshot-uri, nu exista AWR. Fara AWR, diagnosticarea unui problem de performanta devine un exercitiu de intuitie in loc de o analiza bazata pe date. Snapshot-urile sunt fundatia observabilitatii in Oracle.
