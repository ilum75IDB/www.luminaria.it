---
title: "EXPLAIN ANALYZE non basta: come leggere davvero un piano di esecuzione PostgreSQL"
description: "Un caso reale dove l'optimizer ha scelto un nested loop su 2 milioni di righe perché le statistiche erano vecchie. Come leggere un execution plan, trovare i segnali d'allarme e intervenire."
date: "2025-10-28T10:00:00+01:00"
draft: false
translationKey: "explain_analyze_postgresql"
tags: ["explain", "query-tuning", "execution-plan", "optimizer", "performance"]
categories: ["postgresql"]
image: "explain-analyze-postgresql.cover.jpg"
---

L'altro giorno un collega mi manda uno screenshot su Teams. Una query che gira su una tabella da 2 milioni di righe, 45 secondi di esecuzione. Mi scrive:

> "Ho fatto EXPLAIN ANALYZE, ma non capisco cosa c'è che non va. Il piano sembra corretto."

Spoiler: il piano non era affatto corretto. L'optimizer aveva scelto un nested loop join dove serviva un hash join, e la ragione era banale — statistiche non aggiornate. Ma per arrivarci ho dovuto leggere il piano riga per riga, e lì mi sono reso conto che la maggior parte dei DBA che conosco usa EXPLAIN ANALYZE come un oracolo binario: se il tempo è alto, la query è lenta. Fine dell'analisi.

No. EXPLAIN ANALYZE è uno strumento di diagnostica, non un verdetto. Bisogna saperlo leggere.

------------------------------------------------------------------------

## 🔧 EXPLAIN, EXPLAIN ANALYZE, EXPLAIN (ANALYZE, BUFFERS): tre cose diverse

Partiamo dalle basi, perché la confusione è più diffusa di quanto si creda.

**EXPLAIN** da solo mostra il piano *stimato*. L'optimizer decide cosa farebbe, ma non esegue nulla. Utile per capire la strategia, inutile per capire la realtà.

``` sql
EXPLAIN
SELECT * FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.created_at > '2025-01-01';
```

**EXPLAIN ANALYZE** esegue la query e aggiunge i tempi reali. Ora vedi quanto ha impiegato ogni nodo, quante righe ha effettivamente restituito. Ma manca un pezzo.

``` sql
EXPLAIN ANALYZE
SELECT * FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.created_at > '2025-01-01';
```

**EXPLAIN (ANALYZE, BUFFERS)** è quello che uso sempre. Aggiunge le informazioni su quante pagine disco ha letto, quante erano in cache (shared hit) e quante ha dovuto caricare da disco (shared read). Senza BUFFERS stai guidando di notte senza fari.

``` sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.created_at > '2025-01-01';
```

Regola personale: se qualcuno mi manda un EXPLAIN senza BUFFERS, glielo rimando indietro.

------------------------------------------------------------------------

## 📖 Anatomia di un nodo: cosa leggere e in che ordine

Un piano di esecuzione è un albero. Ogni nodo ha questa struttura:

``` text
->  Hash Join  (cost=1234.56..5678.90 rows=50000 width=120)
      (actual time=12.345..89.012 rows=48750 loops=1)
      Buffers: shared hit=1200 read=3400
```

Ecco cosa guardare:

**cost** — sono due numeri separati da `..`. Il primo è il costo di startup (quanto prima di restituire la prima riga), il secondo è il costo totale stimato. Sono unità arbitrarie dell'optimizer, non millisecondi. Servono per confrontare piani alternativi, non per misurare performance assolute.

**rows** — le righe stimate dall'optimizer. Confrontale con `actual rows`. Se c'è un ordine di grandezza di differenza, hai trovato il problema.

**actual time** — tempo reale in millisecondi. Anche qui due valori: startup e totale. Attenzione al campo `loops`: se loops=10, il tempo totale va moltiplicato per 10.

**Buffers** — `shared hit` sono le pagine trovate in memoria, `shared read` quelle lette da disco. Se `read` domina, il tuo working set non sta in RAM.

------------------------------------------------------------------------

## 🚨 Il segnale d'allarme numero uno: rows stimate vs rows reali

Torno al caso del mio collega. Il piano mostrava:

``` text
->  Nested Loop  (cost=0.87..45678.12 rows=150 width=200)
      (actual time=0.034..44890.123 rows=1950000 loops=1)
```

L'optimizer stimava 150 righe. In realtà ne arrivavano quasi 2 milioni.

Quando la stima è sbagliata di 4 ordini di grandezza, il piano è inevitabilmente sbagliato. L'optimizer ha scelto un nested loop perché pensava di iterare su 150 righe. Un nested loop su 150 righe è velocissimo. Su 2 milioni è un disastro.

Un hash join o un merge join sarebbero stati la scelta corretta. Ma l'optimizer non poteva saperlo con le statistiche che aveva.

Come regola pratica: se il rapporto tra righe stimate e righe reali supera 10x, hai un problema di statistiche. Sopra 100x, il piano è quasi certamente sbagliato.

------------------------------------------------------------------------

## 🔍 Perché le statistiche mentono

PostgreSQL mantiene statistiche sulle tabelle in `pg_statistic` (leggibili tramite `pg_stats`). Queste statistiche includono:

- distribuzione dei valori (most common values)
- istogramma dei valori
- numero di valori distinti
- percentuale di NULL

L'optimizer usa queste informazioni per stimare la selettività di ogni condizione WHERE e la cardinalità di ogni join.

Il problema? Le statistiche si aggiornano con `ANALYZE` — che può essere manuale o gestito dall'autovacuum. Ma l'autovacuum lancia ANALYZE solo quando il numero di righe modificate supera una soglia:

``` text
threshold = autovacuum_analyze_threshold + autovacuum_analyze_scale_factor × n_live_tuples
```

Di default: 50 righe + 10% delle righe vive. Su una tabella da 2 milioni di righe, servono 200.000 modifiche prima che scatti l'ANALYZE automatico.

Nel caso del mio collega, la tabella `orders` era cresciuta da 500.000 a 2 milioni di righe in tre settimane — un import massivo da un sistema legacy. L'autovacuum non aveva ancora aggiornato le statistiche perché il 10% di 500.000 (la dimensione nota) era 50.000, e le righe aggiunte erano state inserite in batch che non avevano superato la soglia singolarmente.

Risultato: l'optimizer ragionava ancora come se la tabella avesse 500.000 righe con la vecchia distribuzione dei valori.

------------------------------------------------------------------------

## 🛠️ Aggiornare le statistiche: la prima cosa da fare

La soluzione immediata era ovvia:

``` sql
ANALYZE orders;
```

Dopo l'ANALYZE, ho rilanciato la query con EXPLAIN (ANALYZE, BUFFERS):

``` text
->  Hash Join  (cost=8500.00..32000.00 rows=1940000 width=200)
      (actual time=120.000..2800.000 rows=1950000 loops=1)
      Buffers: shared hit=28000 read=4500
```

Da 45 secondi a meno di 3 secondi. L'optimizer aveva scelto un hash join, la stima delle righe era corretta, e il piano era completamente diverso.

Ma non mi sono fermato qui. Se il problema si è presentato una volta, si ripresenterà.

------------------------------------------------------------------------

## 📊 default_statistics_target: quando 100 non basta

PostgreSQL raccoglie 100 valori di campione per colonna come default. Per tabelle piccole o con distribuzione uniforme, è sufficiente. Per tabelle grandi con distribuzione non uniforme, 100 campioni possono dare una rappresentazione distorta.

Nel caso della tabella `orders`, la colonna `customer_id` aveva una distribuzione molto skewed: il 5% dei clienti generava il 60% degli ordini. Con 100 campioni, l'optimizer non coglieva questa asimmetria.

La soluzione:

``` sql
ALTER TABLE orders
ALTER COLUMN customer_id SET STATISTICS 500;

ANALYZE orders;
```

Dopo aver aumentato il target a 500, le stime dell'optimizer sulla cardinalità dei join con `customers` sono diventate molto più accurate.

Regola: se una colonna è usata frequentemente in WHERE o JOIN e ha distribuzione non uniforme, alza il target. 500 è un buon punto di partenza. Puoi arrivare a 1000, ma oltre raramente porta benefici e rallenta l'ANALYZE stesso.

------------------------------------------------------------------------

## ⚠️ Quando forzare il planner: enable_nestloop e enable_hashjoin

A volte, anche con statistiche aggiornate, l'optimizer prende una strada sbagliata. Capita con query complesse, molte tabelle in join, o quando la correlazione tra colonne inganna le stime.

PostgreSQL offre dei parametri per disabilitare specifiche strategie:

``` sql
SET enable_nestloop = off;
```

Questo forza l'optimizer a non usare nested loop. Non è una soluzione, è un cerotto diagnostico. Se disabiliti il nested loop e la query passa da 45 secondi a 3 secondi, hai confermato che il problema era la scelta del join. Ma non puoi lasciare `enable_nestloop = off` in produzione perché ci sono mille query dove il nested loop è la scelta giusta.

Uso questi parametri solo in due scenari:

1. **Diagnostica**: per confermare quale strategia di join è il problema
2. **Emergenza**: quando il business è fermo e devi far ripartire una query critica mentre cerchi la vera soluzione

Dopo la diagnostica, il fix corretto è sempre su statistiche, indici, o riscrittura della query.

------------------------------------------------------------------------

## 📋 Il mio workflow quando una query è lenta

Dopo trent'anni a fare questo mestiere, il mio processo è diventato quasi meccanico:

**1. EXPLAIN (ANALYZE, BUFFERS)** — sempre con BUFFERS. Salvo l'output completo, non solo le ultime righe.

**2. Cerco la discrepanza rows** — confronto `rows=` stimato con `actual rows=` reale su ogni nodo. Parto dai nodi foglia e salgo verso la root. La prima discrepanza significativa è quasi sempre la causa.

**3. Controllo le statistiche** — guardo `pg_stats` per le colonne coinvolte. Verifico `last_autoanalyze` e `last_analyze` in `pg_stat_user_tables`. Se l'ultimo ANALYZE è vecchio, lo lancio e rivaluto.

**4. Valuto BUFFERS** — se `shared read` è molto alto rispetto a `shared hit`, il problema potrebbe essere I/O, non il piano. In quel caso il fix è `shared_buffers` o il working set semplicemente non sta in RAM.

**5. Testo alternative** — se le statistiche sono aggiornate ma il piano è ancora sbagliato, uso `enable_nestloop`, `enable_hashjoin`, `enable_mergejoin` per capire quale strategia funziona meglio. Poi cerco di guidare l'optimizer verso quella strategia con indici o riscrittura.

Niente di spettacolare. Nessun trucco magico. Solo lettura sistematica del piano, una riga alla volta.

------------------------------------------------------------------------

## 💬 La lezione di quel giorno

Il mio collega, dopo aver visto la differenza, mi ha detto: "Ma allora bastava un ANALYZE?"

Sì e no. In quel caso specifico, sì. Ma il punto non è il comando. Il punto è saper leggere il piano per capire *dove* guardare. EXPLAIN ANALYZE ti dà i dati. Sta a te interpretarli.

Ho visto DBA con anni di esperienza lanciare EXPLAIN ANALYZE, guardare il tempo totale in fondo, e dire "la query è lenta". È come guardare la temperatura di un paziente e dire "ha la febbre". Sì, ma da cosa dipende?

Il piano di esecuzione ti dice da cosa dipende. Ogni nodo è un organo. Le righe stimate contro quelle reali sono i valori di laboratorio. I buffer sono le lastre. E l'ANALYZE è l'antibiotico che risolve il 70% dei casi.

Ma per quel restante 30%, devi leggere. Riga per riga. Nodo per nodo. Non c'è scorciatoia.
