---
categories:
- oracle
date: '2026-08-04'
description: Oracle 26ai introduce CREATE ASSERTION, il vincolo cross-tabella promesso
  da SQL-92. Sintassi, confronto con i trigger e un caso reale assicurativo.
draft: false
image: articolo-oracle-assertions-in-oracle-26ai.cover.jpg
seoTitle: 'Assertions in Oracle 26ai: vincoli cross-tabella dichiarativi'
tags:
- oracle-26ai
- integrity-constraints
- assertions
- sql-standard
- oracle-23ai
title: 'Assertions in Oracle 26ai: finalmente un vincolo che attraversa le tabelle'
translationKey: articolo_oracle_assertions_in_oracle_26ai
webo_generated_at: 2026-07-31
webo_status: scheduled
---

## Tre trigger, un job notturno e 1.247 polizze orfane

Era una migrazione batch di routine — o almeno così sembrava. Un grande gruppo assicurativo italiano stava consolidando dati storici da un sistema legacy: qualche milione di righe su `polizze` e `beneficiari`, finestra di manutenzione sabato notte, script testato in staging. Per velocizzare il caricamento, il team aveva disabilitato temporaneamente i trigger sulle due tabelle. Procedura standard, documentata nel runbook.

Il problema è emerso domenica mattina, quando il job notturno di riconciliazione ha terminato la sua corsa di 45 minuti su 2,1 milioni di polizze e ha prodotto un report con 1.247 righe anomale: polizze in stato `ATTIVA` senza nessun beneficiario associato. Violazione diretta di una regola di business critica — "ogni polizza attiva deve avere almeno un beneficiario con quota complessiva pari al 100%" — che il sistema doveva garantire in ogni momento.

La regola era implementata con tre trigger coordinati: uno `AFTER INSERT OR UPDATE` su `polizze` che verificava la presenza di beneficiari quando lo stato diventava `ATTIVA`, uno `AFTER DELETE` su `beneficiari` che controllava che la polizza collegata non restasse orfana, e un terzo `AFTER UPDATE` su `beneficiari` che ricalcolava le quote. Più il job notturno come rete di sicurezza per i casi che sfuggivano — e qualcosa sfuggiva sempre, in particolare durante le operazioni batch con trigger disabilitati.

Tempo di detection: circa 18 ore. Tempo di correzione manuale: mezza giornata di lavoro. Costo reale: basso, per fortuna. Ma la domanda che è rimasta sul tavolo dopo l'incidente era quella giusta: esiste un modo per esprimere questa regola a livello di schema, in modo che non possa essere aggirata da nessuna operazione DML, batch o meno?

Con Oracle 26ai, la risposta è sì.

## SQL-92 aveva ragione, ma nessuno l'aveva ascoltata

Lo standard SQL definisce `CREATE ASSERTION` dal 1992. L'idea è semplice: un vincolo di integrità non deve essere limitato a una singola riga o a una singola tabella — deve poter esprimere predicati sull'intero stato del database. La sintassi originale dello standard è:

```sql
CREATE ASSERTION nome_vincolo CHECK (condizione_booleana)
```

dove `condizione_booleana` può coinvolgere subquery, aggregazioni, join tra tabelle diverse. Semanticamente, il vincolo è soddisfatto quando la condizione è `TRUE` o `UNKNOWN`; viene violato quando è `FALSE`.

Il problema è che nessun RDBMS mainstream aveva mai implementato questa feature in modo completo. PostgreSQL non ce l'ha. MySQL non ce l'ha. SQL Server non ce l'ha. I motivi sono noti: la valutazione di un predicato che attraversa più tabelle ha un costo potenzialmente elevato, e il momento corretto in cui valutarlo — dopo ogni singola istruzione DML, o alla fine della transazione? — apre questioni di implementazione non banali. Risultato: per trent'anni, la feature è rimasta nel documento dello standard senza atterrare nei prodotti.

Oracle 23ai (poi rinominato 26ai nella versione successiva) è il primo RDBMS enterprise mainstream a portarla in produzione [1]. È una scelta che ha un peso concettuale preciso: segnala che Oracle sta prendendo sul serio la conformità allo standard SQL su feature che altri vendor hanno ignorato, e che il modello relazionale — con i suoi vincoli dichiarativi — è ancora la direzione di sviluppo, non un'eredità da gestire.

La distinzione fondamentale rispetto a un `CHECK` constraint ordinario è questa: un `CHECK` valuta un predicato sulle colonne della riga corrente, al momento dell'inserimento o dell'aggiornamento. Un'`ASSERTION` valuta un predicato sull'intero contenuto delle tabelle coinvolte, dopo ogni operazione DML che potrebbe renderlo falso. Sono strumenti diversi per problemi diversi.

## `CREATE ASSERTION` in Oracle 26ai: la sintassi

Riprendendo lo schema del progetto assicurativo, le due tabelle di riferimento sono:

```sql
-- Schema ins_core su oracle-node-01
CREATE TABLE polizze (
    id            NUMBER PRIMARY KEY,
    numero        VARCHAR2(20) NOT NULL,
    stato         VARCHAR2(10) CHECK (stato IN ('BOZZA','ATTIVA','SCADUTA','ANNULLATA')),
    data_inizio   DATE NOT NULL,
    data_fine     DATE
);

CREATE TABLE beneficiari (
    id            NUMBER PRIMARY KEY,
    id_polizza    NUMBER NOT NULL REFERENCES polizze(id),
    nome          VARCHAR2(100) NOT NULL,
    quota_pct     NUMBER(5,2) NOT NULL CHECK (quota_pct > 0 AND quota_pct <= 100)
);
```

Il vincolo che i tre trigger cercavano di garantire si esprime così:

```sql
-- Ogni polizza ATTIVA deve avere almeno un beneficiario
CREATE ASSERTION ins_core.polizza_ha_beneficiario CHECK (
    NOT EXISTS (
        SELECT 1 FROM ins_core.polizze p
        WHERE  p.stato = 'ATTIVA'
        AND    NOT EXISTS (
                   SELECT 1 FROM ins_core.beneficiari b
                   WHERE  b.id_polizza = p.id
               )
    )
);
```

Questo è il pattern existential classico: "non esiste nessuna polizza attiva per cui non esiste nessun beneficiario". La doppia negazione è il modo standard per esprimere "per ogni X deve esistere almeno un Y" in SQL, e le Assertions lo rendono finalmente dichiarabile a livello di schema [2].

Il secondo vincolo — le quote devono sommare a 100 — usa un pattern con aggregazione:

```sql
-- Le quote dei beneficiari di ogni polizza ATTIVA devono sommare a 100
CREATE ASSERTION ins_core.quote_beneficiari_complete CHECK (
    NOT EXISTS (
        SELECT b.id_polizza
        FROM   ins_core.beneficiari b
        JOIN   ins_core.polizze p ON p.id = b.id_polizza
        WHERE  p.stato = 'ATTIVA'
        GROUP BY b.id_polizza
        HAVING SUM(b.quota_pct) <> 100
    )
);
```

Questo secondo pattern è quello che con i `CHECK` constraint tradizionali è semplicemente impossibile esprimere: la condizione coinvolge un'aggregazione su righe diverse della stessa tabella, filtrata per join con un'altra tabella.

Per rimuovere un'Assertion:

```sql
DROP ASSERTION ins_core.polizza_ha_beneficiario;
```

La documentazione Oracle 26ai prevede anche la possibilità di disabilitare temporaneamente un'Assertion con `DISABLE` e riabilitarla con `ENABLE`, analogamente ai constraint tradizionali [1]. Questo punto è rilevante per le operazioni di manutenzione — ma, come vedremo, è anche esattamente il punto critico da gestire con attenzione.

## Perché le alternative pre-26ai non erano equivalenti

Vale la pena essere precisi su questo, perché la tentazione di dire "si può fare anche con i trigger" è reale — e tecnicamente vera, ma nasconde differenze importanti.

| Meccanismo | Dichiarativo | Resiste a bulk load | Leggibile da schema | Costo di valutazione |
|---|---|---|---|---|
| `CHECK` constraint | ✅ | ✅ | ✅ | Basso (riga singola) |
| Trigger `AFTER` | ❌ | ❌ (disabilitabile) | ❌ | Medio (per riga) |
| `DEFERRABLE` constraint | ✅ | Parziale | ✅ | Basso-medio |
| Materialised view + `WITH CHECK OPTION` | Parziale | ❌ | Parziale | Alto (refresh) |
| **ASSERTION** | ✅ | ✅ (se non disabilitata) | ✅ | Medio-alto (cross-table) |

Il `CHECK` constraint è dichiarativo e leggibile, ma non può contenere subquery che referenziano altre tabelle — limitazione nota e documentata [3]. Oracle la applica in modo esplicito: un `CHECK` che tenta di fare `SELECT` su un'altra tabella genera un errore in fase di creazione.

I trigger `AFTER INSERT/UPDATE/DELETE` funzionano, ma sono procedurali. Non compaiono nella definizione dello schema in modo leggibile; si disabilitano con `ALTER TABLE DISABLE ALL TRIGGERS` o con `ALTER TABLE ... DISABLE TRIGGER nome`; in DML complessi con ordini di firing non banali possono produrre risultati inattesi. L'incidente delle 1.247 polizze orfane è esattamente la conseguenza di questa fragilità.

I `DEFERRABLE` constraint gestiscono la temporaneità — permettono di posticipare la verifica alla fine della transazione anziché a ogni singola istruzione — ma non esprimono predicati cross-tabella. Sono utili per DML multi-step (inserisci la riga padre, poi le figlie, verifica la foreign key solo al commit), non per vincoli che attraversano tabelle diverse [3].

Le materialised view con `WITH CHECK OPTION` sono un'approssimazione creativa: si crea una vista che espone le violazioni, si aggiunge un vincolo sulla vista. Non è un vincolo in senso stretto, ha costi di refresh, e il comportamento in scenari di concorrenza è meno prevedibile.

## Quando ha senso usarle, e quando no

Le Assertions non sono gratuite. Il costo di valutazione è reale: ogni operazione DML su una tabella coinvolta in un'Assertion può richiedere l'esecuzione della subquery di verifica. Su tabelle con DML ad alta frequenza — milioni di insert al secondo, sistemi OLTP con latenza critica — questo overhead va misurato prima di adottare la feature in produzione.

I casi in cui le Assertions hanno senso:

- **Regole di integrità stabili nel tempo**: se il predicato cambia raramente, il costo di `DROP/CREATE` è accettabile. Se la regola cambia ogni sprint, i trigger sono più flessibili.
- **Frequenza DML moderata**: sistemi transazionali normali, non pipeline di ingestion ad alto throughput.
- **Predicati che coinvolgono aggregazioni o esistenza cross-tabella**: esattamente i casi che i `CHECK` constraint non coprono.
- **Ambienti dove la leggibilità dello schema è critica**: audit, compliance, onboarding di nuovi DBA — avere il vincolo dichiarato nello schema è un vantaggio documentale concreto.

I casi da evitare o da valutare con attenzione:

- **Bulk load con direct-path insert**: l'`INSERT /*+ APPEND */` in Oracle bypassa il buffer cache e scrive direttamente nei datafile. Il comportamento delle Assertions in questo scenario — se vengono valutate, quando, con quale granularità — va verificato nell'ambiente specifico [4]. Non dare per scontato che il comportamento sia identico al conventional insert.
- **Tabelle con DML ad altissima frequenza**: il costo di valutazione si moltiplica per ogni operazione. Misurare prima.
- **Feature in fase di maturazione**: Oracle 26ai è una versione recente. Le Assertions sono una feature nuova, e il comportamento in edge case — concorrenza elevata, rollback parziali, operazioni DDL concorrenti — va testato nell'ambiente di destinazione prima di adottare in produzione [4].

Un punto che vale la pena sottolineare: anche un'Assertion si può disabilitare, con `DISABLE`. Questo significa che il problema delle operazioni di manutenzione non sparisce — si sposta. La differenza rispetto ai trigger è che la disabilitazione di un'Assertion è un'operazione DDL esplicita, visibile nel catalogo di sistema, più difficile da fare "per sbaglio" in uno script di migrazione. Non è una protezione assoluta, ma è un guardrail più robusto.

## Pattern ricorrenti: il ricettario

Tre pattern coprono la maggior parte dei vincoli cross-tabella che si incontrano in pratica.

**Pattern "almeno uno"** — ogni riga della tabella A deve avere almeno una riga corrispondente in B:

```sql
-- Ogni polizza ATTIVA deve avere almeno un beneficiario
CREATE ASSERTION ins_core.polizza_ha_beneficiario CHECK (
    NOT EXISTS (
        SELECT 1 FROM ins_core.polizze p
        WHERE  p.stato = 'ATTIVA'
        AND    NOT EXISTS (
                   SELECT 1 FROM ins_core.beneficiari b
                   WHERE  b.id_polizza = p.id
               )
    )
);
```

**Pattern "somma vincolata"** — un'aggregazione su righe di B, raggruppate per chiave di A, deve rispettare una condizione:

```sql
-- Le quote dei beneficiari di ogni polizza ATTIVA devono sommare a 100
CREATE ASSERTION ins_core.quote_beneficiari_complete CHECK (
    NOT EXISTS (
        SELECT b.id_polizza
        FROM   ins_core.beneficiari b
        JOIN   ins_core.polizze p ON p.id = b.id_polizza
        WHERE  p.stato = 'ATTIVA'
        GROUP BY b.id_polizza
        HAVING SUM(b.quota_pct) <> 100
    )
);
```

**Pattern "tutti devono"** — ogni riga di A deve soddisfare una condizione che coinvolge B (variante con condizione negata):

```sql
-- Ogni sinistro aperto deve riferirsi a una polizza in stato ATTIVA
-- (esempio con tabella sinistri aggiuntiva)
CREATE ASSERTION ins_core.sinistro_su_polizza_attiva CHECK (
    NOT EXISTS (
        SELECT 1 FROM ins_core.sinistri s
        JOIN   ins_core.polizze p ON p.id = s.id_polizza
        WHERE  s.stato = 'APERTO'
        AND    p.stato <> 'ATTIVA'
    )
);
```

Questi tre pattern, combinati, coprono la stragrande maggioranza dei vincoli di integrità cross-tabella che nei sistemi pre-26ai finivano nei trigger. La sintassi è più verbosa di un `CHECK` constraint semplice, ma è dichiarativa, leggibile, e vive nello schema.

## Il collegamento con l'articolo #88 e la direzione Oracle

Chi ha letto l'articolo sull'evoluzione di Oracle da 19c a 26ai ricorderà che le Assertions erano citate nell'ultima sezione, come una delle feature più significative della nuova versione — ma senza approfondimento. Questo articolo è quel approfondimento.

Nel contesto più ampio della roadmap Oracle, le Assertions si inseriscono in una tendenza precisa: avvicinare il prodotto alla conformità completa con lo standard SQL, su feature che erano rimaste teoriche per decenni. JSON Relational Duality, True Cache, e le Assertions condividono questa caratteristica — sono risposte a esigenze reali che il modello relazionale aveva già teorizzato, ma che i prodotti non avevano mai implementato completamente.

Non è nostalgia per il purismo relazionale. È riconoscere che alcune idee del modello relazionale — i vincoli dichiarativi in primo luogo — hanno un valore pratico concreto che l'industria ha sottovalutato per anni, delegando ai trigger e alla logica applicativa quello che avrebbe dovuto stare nello schema.

## Il vincolo che non si rompe perché non c'è codice da rompere

Tre trigger coordinati, un job notturno di 45 minuti, e 1.247 polizze orfane trovate 18 ore dopo l'incidente. Non è un disastro — è una situazione gestita, corretta, documentata. Ma il costo di manutenzione di quei tre trigger nel tempo è reale: ogni modifica alla logica di business richiede di aggiornare il codice procedurale, testarlo, coordinare i firing order, ricordarsi di riabilitarli dopo ogni operazione batch.

L'Assertion non è una bacchetta magica. Ha un costo di valutazione, ha limitazioni in scenari di bulk load, è una feature nuova su una versione Oracle che non è ancora in produzione ovunque. Prima di adottarla in un sistema critico, vale la pena testare il comportamento specifico nell'ambiente di destinazione — in particolare per i pattern di DML che si usano realmente, non solo per i casi standard.

Il punto concettuale, però, è solido: il codice che non si scrive non si rompe. Un vincolo dichiarato nello schema è visibile, leggibile, difficile da aggirare per sbaglio. Un trigger è codice procedurale che vive in un posto separato, si disabilita con un'istruzione, e richiede che chi fa la migrazione batch sappia che esiste e che importa.

Spostare la regola di business nel posto giusto — lo schema — è una scelta di design, non solo una questione tecnica. Le Assertions di Oracle 26ai rendono questa scelta possibile per la prima volta in modo dichiarativo su un RDBMS enterprise mainstream. Vale la pena capirle, anche per chi non migrerà a 26ai la settimana prossima.

## Fonti ufficiali

1. Oracle Database 23ai — `CREATE ASSERTION` syntax and semantics — `<TODO: scout fonte ufficiale per "CREATE ASSERTION Oracle 23ai/26ai documentation">`
2. Oracle Database 23ai New Features Guide — Integrity Constraints — `<TODO: scout fonte ufficiale per "Oracle 23ai New Features Guide — Integrity Constraints">`
3. Oracle Database SQL Language Reference — [Constraint Clauses](https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/constraint.html) — copre `CHECK`, `DEFERRABLE`, vincoli di integrità esistenti (URL da verificare per versione 26ai)
4. Oracle Database 26ai Release Notes — stato GA vs preview delle Assertions, known limitations — `<TODO: scout fonte ufficiale per "Oracle 26ai Release Notes">`
5. ISO/IEC 9075 SQL Standard, SQL-92 — definizione originale di `CREATE ASSERTION` — `<TODO: scout riferimento pubblico accessibile per SQL-92 CREATE ASSERTION>`

## Glossario
- **[ASSERTION](/it/glossary/predicato-existential/)** (Oracle 26ai / SQL standard) — vincolo di integrità dichiarativo che esprime un predicato sull'intero stato del database, potenzialmente coinvolgendo più tabelle. Definito in SQL-92, implementato per la prima volta in un RDBMS enterprise mainstream con Oracle 23ai/26ai.

- **[Predicato existential](/it/glossary/assertion/)** (SQL) — espressione logica che afferma l'esistenza di almeno una riga che soddisfa una condizione. In SQL si esprime tipicamente con `EXISTS` o `NOT EXISTS` in subquery correlata; è il pattern base per le Assertions di tipo "almeno uno".

- **[Predicato universal](/it/glossary/predicato-existential/)** (SQL) — espressione logica che afferma che una condizione vale per tutte le righe di un insieme. In SQL si esprime indirettamente con `NOT EXISTS (... WHERE NOT condizione)`, perché SQL non ha un quantificatore universale nativo.

- **[DEFERRABLE constraint](/it/glossary/assertion/)** (Oracle / SQL standard) — vincolo di integrità la cui verifica può essere posticipata alla fine della transazione anziché a ogni singola istruzione DML. Utile per DML multi-step, ma non equivalente a un'Assertion: non esprime predicati cross-tabella.

- **[Direct-path insert](/it/glossary/predicato-existential/)** (Oracle) — modalità di caricamento dati che bypassa il buffer cache e scrive direttamente nei datafile, attivabile con il hint `/*+ APPEND */`. Interagisce con i vincoli di integrità in modo diverso rispetto al conventional insert; il comportamento con le Assertions va verificato caso per caso.
