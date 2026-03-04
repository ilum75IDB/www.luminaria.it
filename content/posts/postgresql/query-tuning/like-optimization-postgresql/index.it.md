---
title: "Quando un LIKE '%valore%' rallenta tutto: un caso reale di ottimizzazione PostgreSQL"
description: "Un caso reale di performance in PostgreSQL dove un LIKE '%valore%' ha generato un full scan degradando i tempi di risposta. Analisi del piano di esecuzione e strategia di indicizzazione scalabile."
date: "2026-02-20T15:52:34+01:00"
lastmod: "2026-02-26T09:34:00+01:00"
draft: false
translationKey: "like_optimization_postgresql"
tags: ["PostgreSQL", "Query Tuning", "Performance", "Indexes", "pg_trgm"]
categories: ["PostgreSQL", "Query Tuning"]
image: "like-optimization-postgresql.cover.jpg"
---

Qualche settimana fa un cliente mi contatta con un problema molto
comune:

> "La ricerca nella console amministrativa è lenta. A volte impiega
> diversi secondi. Abbiamo già ridotto le JOIN, ma il problema non è
> sparito."

Ambiente: PostgreSQL su cloud managed.\
Tabella principale: `payment_report` (\~6 milioni di righe, 3 GB).\
Colonna ricercata: `reference_code`.

Query incriminata:

``` sql
SELECT *
FROM reporting.payment_report r
JOIN reporting.payment_cart c ON c.id = r.cart_id
WHERE c.service_id = 1001
  AND r.reference_code LIKE '%ABC123%'
ORDER BY c.created_at DESC
LIMIT 100;
```

------------------------------------------------------------------------

## 🧠 Prima osservazione: il problema non erano le JOIN

Ho confrontato:

-   versione AS-IS (3 JOIN sulla stessa tabella)
-   versione TO-BE (1 sola JOIN)

Risultato?

Il piano di esecuzione mostrava in entrambi i casi:

``` text
Parallel Seq Scan on payment_report
Rows Removed by Filter: ~2.000.000
Buffers: shared read = centinaia di migliaia
Execution Time: 14–18 secondi
```

La riduzione delle JOIN aveva un impatto marginale.

Il vero problema era un altro.

------------------------------------------------------------------------

## 📌 Il colpevole: `LIKE '%valore%'` senza indice adatto

La ricerca con wildcard iniziale (`%valore%`) rende inutilizzabile un
normale indice B-Tree.

PostgreSQL è costretto a fare una scansione sequenziale dell'intera
tabella.

Nel caso specifico:

-   \~3 GB di dati
-   centinaia di migliaia di pagine da 8KB lette
-   I/O dominante
-   secondi di latenza

Non è un problema di SQL "brutto". È un problema di access path.

------------------------------------------------------------------------

## 🔬 Prima di creare un indice: analisi del rischio

Il cliente giustamente chiede:

> "Se creiamo un indice trigram (GIN), rischiamo di rallentare le
> transazioni di pagamento?"

Qui entra in gioco un concetto spesso ignorato: il **churn**.

### Cos'è il churn?

È quanto una tabella cambia dopo che le righe sono state inserite.

Alta frequenza di: - UPDATE - DELETE

→ alto churn\
→ maggiore costo di manutenzione indice\
→ possibile degrado scritture

Nel nostro caso:

Tabella `payment_report`: - \~12k insert/giorno - 0 update - 0 delete -
0 dead tuples

Profilo: **append-only**

Questo è il miglior scenario possibile per introdurre un GIN.

------------------------------------------------------------------------

## 📊 Verifica fondamentale: sincrono o batch?

La tabella non aveva timestamp di inserimento.

Soluzione: analisi indiretta.

Ho correlato le righe di `payment_report` al timestamp del carrello
(`payment_cart.created_at`) e ho analizzato la distribuzione oraria.

Risultato:

-   andamento continuo 24/7
-   picchi diurni
-   calo notturno
-   perfetta correlazione con il traffico carrelli

Conclusione: popolamento near real-time, non batch notturno.

------------------------------------------------------------------------

## 🛠️ La soluzione

``` sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE INDEX CONCURRENTLY idx_payment_report_reference_trgm
ON reporting.payment_report
USING gin (reference_code gin_trgm_ops);
```

Precauzioni:

-   Creazione in finestra off-peak
-   Modalità CONCURRENTLY
-   Monitoraggio I/O durante la build

------------------------------------------------------------------------

## 📈 Risultato atteso

Dopo l'indice:

-   Eliminazione del Seq Scan
-   Utilizzo di GIN index scan
-   Riduzione drastica I/O
-   Tempo query da secondi → millisecondi

------------------------------------------------------------------------

## 🎯 Lezione chiave

Quando una query è lenta:

1.  Non fermarti al numero di JOIN.
2.  Guarda il piano di esecuzione.
3.  Identifica se il problema è CPU o I/O.
4.  Valuta il churn prima di introdurre un indice GIN.
5.  Misura sempre prima di decidere.

Spesso il problema non è "ottimizzare la query". È dare al planner
l'indice giusto.

------------------------------------------------------------------------

## 💬 Perché condivido questo caso?

Perché è uno scenario estremamente comune:

-   Tabelle grandi
-   Ricerca "contiene"
-   Paura di introdurre indici GIN
-   Timore di degradare le scritture

Con dati alla mano, la decisione diventa tecnica, non emotiva.

L'ottimizzazione non è magia. È misurazione, lettura dei piani e
comprensione del comportamento reale del sistema.

