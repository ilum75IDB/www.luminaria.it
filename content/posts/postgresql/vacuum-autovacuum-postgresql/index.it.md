---
title: "VACUUM e autovacuum: perché PostgreSQL ha bisogno che qualcuno pulisca"
description: "Un database PostgreSQL da 200 GB con tabelle gonfie il triplo del necessario. L'autovacuum era attivo, ma mal configurato. Come diagnosticare il bloat, leggere pg_stat_user_tables e fare tuning senza disabilitare nulla."
date: "2026-03-24T08:03:00+01:00"
lastmod: "2026-03-24T08:03:00+01:00"
draft: false
translationKey: "vacuum_autovacuum_postgresql"
tags: ["vacuum", "autovacuum", "mvcc", "performance", "bloat"]
categories: ["postgresql"]
image: "vacuum-autovacuum-postgresql.cover.jpg"
---

Un paio d'anni fa mi chiedono di guardare un PostgreSQL in produzione che
"rallenta ogni settimana". Sempre lo stesso pattern: lunedì va bene,
venerdì è un disastro. Il weekend qualcuno riavvia il servizio e si
ricomincia.

Database da circa 200 GB. Tabelle principali che occupavano quasi il
triplo dello spazio effettivo dei dati. Query che andavano in sequential
scan su tabelle dove non avrebbero dovuto. Tempi di risposta che
salivano di giorno in giorno.

L'autovacuum era attivo. Nessuno l'aveva disabilitato. Ma nessuno
l'aveva nemmeno configurato.

------------------------------------------------------------------------

## 🧠 MVCC: perché PostgreSQL genera "spazzatura"

Per capire il problema serve un passo indietro. PostgreSQL usa MVCC —
Multi-Version Concurrency Control. Ogni volta che fai un UPDATE, il
database non sovrascrive la riga originale. Crea una nuova versione e
segna la vecchia come "morta".

Stesso discorso per le DELETE: la riga non viene cancellata
fisicamente. Viene marcata come non più visibile alle nuove transazioni.

Queste righe morte si chiamano **dead tuples**. E restano lì, dentro le
pagine dati, occupando spazio su disco e rallentando le scansioni.

È il prezzo che PostgreSQL paga per avere isolamento transazionale senza
lock esclusivi sulle letture. Un prezzo ragionevole — a patto che
qualcuno passi a pulire.

------------------------------------------------------------------------

## 🔧 VACUUM: cosa fa davvero

Il comando `VACUUM` fa una cosa semplice: recupera lo spazio occupato
dai dead tuples e lo rende riutilizzabile per nuovi inserimenti.

Non restituisce spazio al sistema operativo. Non riorganizza la tabella.
Non compatta nulla. Segna le pagine come riscrivibili.

``` sql
VACUUM reporting.transactions;
```

Questo basta nella maggior parte dei casi. Il VACUUM è leggero, non
blocca le scritture, e può girare in parallelo con le query normali.

### E `VACUUM FULL`?

`VACUUM FULL` è un'altra bestia. Riscrive fisicamente l'intera tabella,
eliminando tutto lo spazio morto. Restituisce spazio al filesystem.

Ma ha un costo brutale: **prende un lock esclusivo** sulla tabella per
tutta la durata dell'operazione. Nessuno legge, nessuno scrive. Su
tabelle grandi parliamo di minuti o ore.

``` sql
VACUUM FULL reporting.transactions;
```

In produzione, `VACUUM FULL` va usato rarissimamente. In emergenza. E
sempre fuori orario.

------------------------------------------------------------------------

## ⚙️ Autovacuum: il custode silenzioso

PostgreSQL ha un daemon che fa il VACUUM in automatico: l'autovacuum.

Parte quando una tabella accumula abbastanza dead tuples. Il threshold è
calcolato così:

```
vacuum threshold = autovacuum_vacuum_threshold
                 + autovacuum_vacuum_scale_factor × n_live_tup
```

I default:

- `autovacuum_vacuum_threshold`: **50** dead tuples
- `autovacuum_vacuum_scale_factor`: **0.2** (20%)

Tradotto: su una tabella con 10 milioni di righe, l'autovacuum parte
quando i dead tuples superano **2.000.050**. Due milioni di righe morte
prima che qualcuno faccia pulizia.

Per una tabella con 500.000 update al giorno, significa che
l'autovacuum si attiva forse ogni 4 giorni. Nel frattempo il bloat
cresce, le scansioni rallentano, gli indici si gonfiano.

Ecco perché lunedì andava tutto bene e venerdì era un disastro.

------------------------------------------------------------------------

## 📊 Diagnostica: leggere pg_stat_user_tables

La prima cosa da fare quando sospetti un problema di vacuum è
interrogare `pg_stat_user_tables`:

``` sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
    last_vacuum,
    last_autovacuum,
    autovacuum_count,
    vacuum_count
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

Nel caso del mio cliente, la situazione era questa:

``` text
relname            | n_live_tup | n_dead_tup | dead_pct | last_autovacuum
-------------------+------------+------------+----------+------------------
transactions       | 12.400.000 |  3.800.000 |   23.5%  | 3 days ago
order_lines        |  8.200.000 |  2.100.000 |   20.4%  | 4 days ago
inventory_moves    |  5.600.000 |  1.900.000 |   25.3%  | 5 days ago
```

Quasi un quarto delle righe erano morte. L'autovacuum girava, ma troppo
raramente per tenere il passo.

------------------------------------------------------------------------

## 🎯 Tuning: adattare l'autovacuum alla realtà

Il trucco non è disabilitare l'autovacuum. Mai. Il trucco è configurarlo
per le tabelle che ne hanno bisogno.

PostgreSQL permette di impostare parametri di autovacuum **per singola
tabella**:

``` sql
ALTER TABLE reporting.transactions SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_vacuum_threshold = 1000
);
```

Con questo setting, l'autovacuum parte dopo 1.000 + 1% delle righe vive
di dead tuples. Su 12 milioni di righe, si attiva a ~121.000 dead
tuples invece che a 2 milioni.

### cost_delay: non strangolare il vacuum

Un altro parametro critico è `autovacuum_vacuum_cost_delay`. Controlla
quanto il vacuum "rallenta sé stesso" per non sovraccaricare l'I/O.

Il default è 2 millisecondi. Su server moderni con SSD, è troppo
conservativo. Ridurlo a 0 o 1 ms permette al vacuum di finire prima:

``` sql
ALTER TABLE reporting.transactions SET (
    autovacuum_vacuum_cost_delay = 0
);
```

### max_workers

Il default è 3 worker di autovacuum. Se hai decine di tabelle ad alto
traffico, 3 worker non bastano. Valuta di portarli a 5–6, monitorando
l'impatto su CPU e I/O:

``` text
-- in postgresql.conf
autovacuum_max_workers = 5
```

------------------------------------------------------------------------

## 📏 Misurare il bloat

Come fai a sapere quanto spazio stanno sprecando le tue tabelle?

La query classica usa `pgstattuple`:

``` sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT
    pg_size_pretty(pg_total_relation_size('reporting.transactions')) AS total_size,
    pg_size_pretty(pg_total_relation_size('reporting.transactions')
                   - pg_relation_size('reporting.transactions')) AS index_size,
    *
FROM pgstattuple('reporting.transactions');
```

I campi chiave: `dead_tuple_percent` e `free_space`. Se il dead_tuple
supera il 20–30%, la tabella ha un problema serio.

Un'alternativa meno precisa ma più leggera è stimare il bloat ratio
confrontando `pg_class.relpages` con le righe stimate — ci sono query
consolidate nella community per questo (la classica "bloat estimation
query" di PostgreSQL Experts).

------------------------------------------------------------------------

## 🛠️ Quando VACUUM non basta: pg_repack

Se il bloat è ormai fuori controllo — tabelle al 50–70% di spazio
morto — il VACUUM normale non recupera tutto. Libera i dead tuples, ma
lo spazio frammentato resta.

`VACUUM FULL` funziona, ma blocca tutto.

L'alternativa in produzione è **pg_repack**: ricostruisce la tabella
online, senza lock esclusivi prolungati.

``` bash
pg_repack -d mydb -t reporting.transactions
```

Non è una soluzione da usare ogni settimana. È la cura d'urto per
quando la situazione è già degenerata. La vera soluzione è non
arrivarci, con un autovacuum ben configurato.

------------------------------------------------------------------------

## 💬 Il principio

Disabilitare l'autovacuum è la peggior cosa che puoi fare a un
PostgreSQL in produzione. L'ho visto fare "perché rallenta le query
durante il giorno". Certo, perché nel frattempo il bloat ti mangia
il database da dentro.

L'autovacuum con i default di PostgreSQL è pensato per un database
generico. Nessun database in produzione è generico. Ogni tabella ha il
suo pattern di write, il suo volume, il suo ritmo.

Tre cose da portarsi via:

1. Controlla `pg_stat_user_tables` regolarmente. Se `n_dead_tup` cresce
   più velocemente di quanto l'autovacuum riesca a pulire, hai un
   problema.

2. Configura `scale_factor` e `threshold` per le tabelle ad alto
   traffico. Non esiste una configurazione universale.

3. Non aspettare che il bloat arrivi al 50% per intervenire. A quel
   punto le opzioni sono poche e tutte dolorose.

I database non si mantengono da soli. Nemmeno quelli che hanno un
daemon che ci prova.

------------------------------------------------------------------------

## Glossario

**[VACUUM](/it/glossary/vacuum/)** — Comando PostgreSQL che recupera lo spazio occupato dai dead tuples, rendendolo riutilizzabile per nuovi inserimenti senza restituirlo al sistema operativo.

**[MVCC](/it/glossary/mvcc/)** — Multi-Version Concurrency Control — modello di concorrenza di PostgreSQL che mantiene più versioni delle righe per garantire isolamento transazionale senza lock esclusivi sulle letture.

**[Dead Tuple](/it/glossary/dead-tuple/)** — Riga obsoleta in una tabella PostgreSQL, marcata come non più visibile dopo un UPDATE o DELETE ma non ancora rimossa fisicamente dal disco.

**[Autovacuum](/it/glossary/autovacuum/)** — Daemon PostgreSQL che esegue automaticamente VACUUM e ANALYZE sulle tabelle quando il numero di dead tuples supera una soglia configurabile.

**[Bloat](/it/glossary/bloat/)** — Spazio morto accumulato in una tabella o indice PostgreSQL a causa di dead tuples non rimossi, che gonfia la dimensione su disco e degrada le performance.
