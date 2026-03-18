---
title: "SCD Tipo 2: la storia che il business non sapeva di volere"
description: "Un direttore commerciale chiede quanti clienti aveva la regione Nord a giugno scorso. Il DWH non sa rispondere perché ogni aggiornamento sovrascrive i dati precedenti. Come ho implementato una SCD Tipo 2 con chiavi surrogate e date di validità per restituire al business la memoria storica."
date: "2025-11-11T10:00:00+01:00"
draft: false
translationKey: "scd_tipo_2"
tags: ["scd", "dimensional-modeling", "etl", "kimball", "data-warehouse"]
categories: ["data-warehouse"]
image: "scd-tipo-2.cover.jpg"
---

Il direttore commerciale si presenta alla riunione del lunedì mattina con una domanda semplice: "Quanti clienti avevamo nella regione Nord a giugno scorso?"

Risposta del DWH: silenzio.

Non perché il sistema fosse giù, o perché mancasse la tabella. Il dato c'era, tecnicamente. Ma era sbagliato. Il DWH rispondeva con i clienti che *oggi* sono nella regione Nord — non quelli che c'erano a giugno. Perché ogni notte, il processo di caricamento sovrascriveva l'anagrafica clienti con i dati correnti, cancellando ogni traccia di quello che c'era prima.

Un cliente che a giugno era in regione Nord e a settembre si è spostato in regione Centro? Per il DWH, quel cliente è sempre stato in regione Centro. La storia non esiste.

---

## Il progetto e il modello originale

Il contesto era un data warehouse nel settore assicurativo — gestione sinistri e portafoglio clienti. Il sistema sorgente conteneva un'anagrafica con i dati di ogni cliente: nome, regione, agente di riferimento, classe di rischio, tipo di polizza.

La dimensione nel DWH era modellata così:

``` sql
CREATE TABLE dim_cliente (
    cliente_id      NUMBER(10)    NOT NULL,
    nome            VARCHAR2(100) NOT NULL,
    regione         VARCHAR2(50)  NOT NULL,
    agente          VARCHAR2(100),
    classe_rischio  VARCHAR2(20),
    tipo_polizza    VARCHAR2(50),
    CONSTRAINT pk_dim_cliente PRIMARY KEY (cliente_id)
);
```

L'ETL notturno era un semplice {{< glossary term="merge-sql" >}}MERGE{{< /glossary >}}: se il cliente esiste, aggiorna tutti i campi; se non esiste, inserisci.

``` sql
MERGE INTO dim_cliente d
USING stg_cliente s ON (d.cliente_id = s.cliente_id)
WHEN MATCHED THEN UPDATE SET
    d.nome           = s.nome,
    d.regione        = s.regione,
    d.agente         = s.agente,
    d.classe_rischio = s.classe_rischio,
    d.tipo_polizza   = s.tipo_polizza
WHEN NOT MATCHED THEN INSERT (
    cliente_id, nome, regione, agente, classe_rischio, tipo_polizza
) VALUES (
    s.cliente_id, s.nome, s.regione, s.agente, s.classe_rischio, s.tipo_polizza
);
```

Semplice, pulito, veloce. E completamente sbagliato per un data warehouse.

Questo è quello che {{< glossary term="kimball" >}}Kimball{{< /glossary >}} chiama **SCD Tipo 1** — Slowly Changing Dimension di Tipo 1. Sovrascrivi il vecchio valore con il nuovo. Nessuna storia, nessun versioning. Il dato attuale cancella il dato precedente.

Per un sistema OLTP è perfetto: vuoi sempre l'indirizzo corrente del cliente, il telefono aggiornato, la mail valida. Ma un data warehouse non è un sistema transazionale. Un data warehouse è una macchina del tempo. E una macchina del tempo che sovrascrive il passato è inutile.

---

## Cosa si perde con la Tipo 1

Il direttore commerciale non era l'unico a fare domande a cui il DWH non poteva rispondere. Ecco un campione delle richieste che si sono accumulate in tre mesi:

- *"Quanti clienti sono passati dalla classe di rischio Alta alla Bassa nell'ultimo anno?"* — Impossibile. La classe precedente non esiste più.
- *"L'agente Rossi ha perso clienti rispetto al trimestre scorso?"* — Impossibile. Se un cliente è stato riassegnato all'agente Bianchi, non c'è traccia che fosse mai stato dell'agente Rossi.
- *"Il fatturato della regione Sud è calato o i clienti si sono spostati?"* — Impossibile da distinguere. Se un cliente da 200K si è spostato dalla regione Sud alla regione Centro, il fatturato della regione Sud cala ma non perché il business vada male — semplicemente il cliente ha cambiato indirizzo.

Ogni volta la risposta era la stessa: "Il sistema non tiene la storia." Che tradotto in linguaggio business significa: "Non lo sappiamo."

A un certo punto il CFO ha chiesto un report di analisi trimestrale che confrontasse la composizione del portafoglio clienti tra Q1 e Q2. Il team di BI ha provato a costruirlo. Ci ha messo tre giorni. Il risultato era inaffidabile perché i dati di Q1 non c'erano più — erano stati sovrascritti dai dati di Q2. Il report confrontava Q2 con Q2 travestito da Q1.

È stato quel momento che ha fatto scattare il progetto di ristrutturazione.

---

## SCD Tipo 2: il principio

La Tipo 2 non sovrascrive. Versiona.

Quando un attributo cambia, il record corrente viene chiuso — gli si assegna una data di fine validità — e viene inserito un nuovo record con i valori aggiornati e una nuova data di inizio validità. Il vecchio record resta nel database, intatto, con tutti i valori che aveva quando era corrente.

Per farlo servono tre elementi aggiuntivi nella tabella dimensionale:

1. **Una {{< glossary term="chiave-surrogata" >}}chiave surrogata{{< /glossary >}}** — un identificativo generato dal DWH, distinto dalla chiave naturale del sistema sorgente. Serve perché lo stesso cliente avrà più record (uno per ogni versione), quindi la chiave naturale non è più univoca.
2. **Date di validità** — `valid_from` e `valid_to` — che definiscono l'intervallo temporale in cui ogni versione del record era corrente.
3. **Un flag di versione corrente** — `is_current` — che permette di recuperare rapidamente la versione attiva senza filtrare sulle date.

### La nuova tabella dimensionale

``` sql
CREATE TABLE dim_cliente (
    cliente_key     NUMBER(10)    NOT NULL,
    cliente_id      NUMBER(10)    NOT NULL,
    nome            VARCHAR2(100) NOT NULL,
    regione         VARCHAR2(50)  NOT NULL,
    agente          VARCHAR2(100),
    classe_rischio  VARCHAR2(20),
    tipo_polizza    VARCHAR2(50),
    valid_from      DATE          NOT NULL,
    valid_to        DATE          NOT NULL,
    is_current      CHAR(1)       DEFAULT 'Y' NOT NULL,
    CONSTRAINT pk_dim_cliente PRIMARY KEY (cliente_key)
);

CREATE INDEX idx_dim_cliente_natural ON dim_cliente (cliente_id, is_current);
CREATE INDEX idx_dim_cliente_validity ON dim_cliente (cliente_id, valid_from, valid_to);

CREATE SEQUENCE seq_dim_cliente START WITH 1 INCREMENT BY 1;
```

La `cliente_key` è la chiave surrogata — generata dalla sequence, mai presa dal sistema sorgente. La `cliente_id` è la chiave naturale — serve per collegare le diverse versioni dello stesso cliente.

La `valid_to` per il record corrente la fisso a `DATE '9999-12-31'` — una convenzione standard che semplifica le query temporali. Quando cerchi il record valido a una certa data, il filtro `WHERE data_riferimento BETWEEN valid_from AND valid_to` funziona senza casi speciali.

---

## La logica ETL

L'ETL della Tipo 2 ha due fasi: prima chiudi i record che sono cambiati, poi inserisci le nuove versioni. L'ordine è importante — se inserisci prima di chiudere, hai un momento in cui esistono due versioni "correnti" dello stesso cliente.

### Fase 1: identificare e chiudere i record modificati

``` sql
MERGE INTO dim_cliente d
USING (
    SELECT s.cliente_id,
           s.nome,
           s.regione,
           s.agente,
           s.classe_rischio,
           s.tipo_polizza
    FROM   stg_cliente s
    JOIN   dim_cliente d
           ON  s.cliente_id = d.cliente_id
           AND d.is_current = 'Y'
    WHERE  (s.regione        != d.regione
         OR s.agente         != d.agente
         OR s.classe_rischio != d.classe_rischio
         OR s.tipo_polizza   != d.tipo_polizza
         OR s.nome           != d.nome)
) changed ON (d.cliente_id = changed.cliente_id AND d.is_current = 'Y')
WHEN MATCHED THEN UPDATE SET
    d.valid_to   = TRUNC(SYSDATE) - 1,
    d.is_current = 'N';
```

Il WHERE confronta ogni attributo tracciato. Se anche uno solo è diverso, il record corrente viene chiuso: la `valid_to` viene impostata a ieri e `is_current` diventa 'N'.

Una nota pratica: il confronto con `!=` non gestisce i NULL. Se `agente` può essere NULL, servono le funzioni di confronto NULL-safe. In Oracle uso `DECODE`:

``` sql
WHERE DECODE(s.regione, d.regione, 0, 1) = 1
   OR DECODE(s.agente, d.agente, 0, 1) = 1
   OR DECODE(s.classe_rischio, d.classe_rischio, 0, 1) = 1
   -- ...
```

`DECODE` tratta due NULL come uguali — esattamente il comportamento che serve.

### Fase 2: inserire le nuove versioni

``` sql
INSERT INTO dim_cliente (
    cliente_key, cliente_id, nome, regione, agente,
    classe_rischio, tipo_polizza, valid_from, valid_to, is_current
)
SELECT seq_dim_cliente.NEXTVAL,
       s.cliente_id,
       s.nome,
       s.regione,
       s.agente,
       s.classe_rischio,
       s.tipo_polizza,
       TRUNC(SYSDATE),
       DATE '9999-12-31',
       'Y'
FROM   stg_cliente s
WHERE  NOT EXISTS (
    SELECT 1
    FROM   dim_cliente d
    WHERE  d.cliente_id = s.cliente_id
    AND    d.is_current = 'Y'
);
```

Questa INSERT cattura due casi: i clienti completamente nuovi (che non esistono in dim_cliente) e i clienti la cui versione corrente è stata appena chiusa nella Fase 1 (che quindi non hanno più un record con `is_current = 'Y'`).

La `valid_from` è la data di oggi. La `valid_to` è il "fine del tempo" — `9999-12-31`. La `cliente_key` è generata dalla sequence.

---

## I dati: prima e dopo

Vediamo un esempio concreto. Il cliente 2001 — "Alfa Assicurazioni Srl" — è nella regione Nord, assegnato all'agente Rossi, classe di rischio Media.

A luglio il cliente viene riassegnato all'agente Bianchi. A ottobre cambia classe di rischio da Media ad Alta.

**Con la Tipo 1** (il modello precedente), a ottobre la dim_cliente contiene una sola riga:

``` text
CLIENTE_ID  NOME                     REGIONE  AGENTE   CLASSE_RISCHIO
----------  -----------------------  -------  -------  --------------
2001        Alfa Assicurazioni Srl   Nord     Bianchi  Alta
```

Nessuna traccia di Rossi. Nessuna traccia della classe Media. Per il DWH, questo cliente è sempre stato dell'agente Bianchi con classe Alta.

**Con la Tipo 2**, a ottobre la dim_cliente contiene tre righe:

``` text
KEY   CLIENTE_ID  NOME                     REGIONE  AGENTE   CLASSE  VALID_FROM  VALID_TO    CURRENT
----  ----------  -----------------------  -------  -------  ------  ----------  ----------  -------
1001  2001        Alfa Assicurazioni Srl   Nord     Rossi    Media   2025-01-15  2025-07-09  N
1002  2001        Alfa Assicurazioni Srl   Nord     Bianchi  Media   2025-07-10  2025-10-04  N
1003  2001        Alfa Assicurazioni Srl   Nord     Bianchi  Alta    2025-10-05  9999-12-31  Y
```

Tre versioni dello stesso cliente. Ogni versione racconta un pezzo della storia: chi era l'agente, quale era la classe di rischio, e in quale periodo. Le date non si sovrappongono. Il flag `is_current` identifica la versione attiva.

---

## Le query temporali

Ora il direttore commerciale può avere la sua risposta.

### Quanti clienti nella regione Nord a giugno?

``` sql
SELECT COUNT(DISTINCT cliente_id) AS clienti_nord_giugno
FROM   dim_cliente
WHERE  regione = 'Nord'
AND    DATE '2025-06-15' BETWEEN valid_from AND valid_to;
```

La query è diretta: prendi tutti i record che erano validi il 15 giugno 2025 e filtra per regione. Nessun CASE WHEN, nessuna logica condizionale, nessuna approssimazione.

### Clienti che hanno cambiato classe di rischio nell'ultimo anno

``` sql
SELECT c1.cliente_id,
       c1.nome,
       c1.classe_rischio AS classe_precedente,
       c2.classe_rischio AS classe_attuale,
       c1.valid_to + 1   AS data_cambio
FROM   dim_cliente c1
JOIN   dim_cliente c2
       ON  c1.cliente_id = c2.cliente_id
       AND c1.valid_to + 1 = c2.valid_from
WHERE  c1.classe_rischio != c2.classe_rischio
AND    c1.valid_to >= ADD_MONTHS(TRUNC(SYSDATE), -12)
ORDER BY data_cambio DESC;
```

Due versioni consecutive dello stesso cliente, unite per data di transizione. Se la classe di rischio è diversa tra le due versioni, il cliente ha cambiato classe. La data del cambio è il giorno dopo la chiusura della versione precedente.

### Confronto portafoglio Q1 vs Q2

``` sql
SELECT regione,
       COUNT(DISTINCT CASE
           WHEN DATE '2025-03-31' BETWEEN valid_from AND valid_to
           THEN cliente_id END) AS clienti_q1,
       COUNT(DISTINCT CASE
           WHEN DATE '2025-06-30' BETWEEN valid_from AND valid_to
           THEN cliente_id END) AS clienti_q2
FROM   dim_cliente
WHERE  DATE '2025-03-31' BETWEEN valid_from AND valid_to
   OR  DATE '2025-06-30' BETWEEN valid_from AND valid_to
GROUP BY regione
ORDER BY regione;
```

Un singolo scan della tabella, due conteggi distinti filtrati per data. Il CFO ha il suo report trimestrale — quello vero, non quello che confrontava Q2 con sé stesso.

---

## La fact table e le chiavi surrogate

Un punto che spesso viene sottovalutato: la {{< glossary term="fact-table" >}}fact table{{< /glossary >}} deve usare la **chiave surrogata**, non la chiave naturale.

``` sql
CREATE TABLE fact_sinistro (
    sinistro_key    NUMBER(10)    NOT NULL,
    cliente_key     NUMBER(10)    NOT NULL,  -- FK alla versione specifica
    data_key        NUMBER(8)     NOT NULL,
    importo         NUMBER(15,2),
    tipo_sinistro   VARCHAR2(50),
    CONSTRAINT pk_fact_sinistro PRIMARY KEY (sinistro_key),
    CONSTRAINT fk_fact_cliente FOREIGN KEY (cliente_key)
        REFERENCES dim_cliente (cliente_key)
);
```

Il `cliente_key` nella fact punta alla *versione* del cliente che era corrente al momento del sinistro. Se un sinistro avviene a maggio, quando il cliente era ancora dell'agente Rossi, la fact punta alla versione con agente Rossi. Se un altro sinistro avviene a settembre, con il cliente ormai dell'agente Bianchi, la fact punta alla versione con agente Bianchi.

Il risultato è che ogni fatto è associato al contesto dimensionale corretto per il momento in cui è accaduto. Se interroghi i sinistri di maggio, li vedi con l'agente Rossi. Se interroghi quelli di settembre, li vedi con l'agente Bianchi. Senza nessuna logica temporale nella query — il JOIN diretto tra fact e dimension restituisce il contesto giusto.

``` sql
-- Sinistri per agente, con il contesto corretto al momento del sinistro
SELECT d.agente,
       COUNT(*)       AS num_sinistri,
       SUM(f.importo) AS importo_totale
FROM   fact_sinistro f
JOIN   dim_cliente d ON f.cliente_key = d.cliente_key
GROUP BY d.agente
ORDER BY importo_totale DESC;
```

Nessuna clausola temporale. Il JOIN sulla chiave surrogata fa tutto il lavoro.

---

## Le dimensioni della Tipo 2

Il costo della Tipo 2 è la crescita della tabella dimensionale. Con la Tipo 1, ogni cliente è una riga. Con la Tipo 2, ogni cliente può avere N righe — una per ogni cambio di attributo tracciato.

Nel progetto assicurativo i numeri erano questi:

| Metrica | Valore |
|---------|--------|
| Clienti attivi | ~120.000 |
| Attributi tracciati | 4 (regione, agente, classe rischio, tipo polizza) |
| Tasso di cambio medio | ~8% dei clienti/anno |
| Righe dim_cliente dopo 1 anno | ~140.000 |
| Righe dim_cliente dopo 3 anni | ~180.000 |
| Righe dim_cliente dopo 5 anni | ~220.000 |

Da 120K a 220K in cinque anni. Un aumento del 83% — che sembra tanto in percentuale ma è trascurabile in termini assoluti. 220K righe sono niente per Oracle. La query con indice sulla chiave surrogata resta nell'ordine dei millisecondi.

Il problema si pone quando hai milioni di clienti con alti tassi di cambio. In quel caso monitori la crescita, consideri il partizionamento della dimensione, e sopratutto scegli con cura *quali* attributi tracciare. Non tutti gli attributi meritano la Tipo 2. Il numero di telefono del cliente? Tipo 1, sovrascrittura. La regione commerciale? Tipo 2, perché ha impatto sull'analisi del fatturato.

La scelta di quali attributi tracciare con Tipo 2 è una decisione di business, non tecnica. Chiedi al business: "Se questo campo cambia, vi serve sapere qual era il valore precedente?" Se la risposta è sì, è Tipo 2. Se è no, è Tipo 1.

---

## Quando non serve la Tipo 2

Non tutte le dimensioni hanno bisogno della storia. Ho visto progetti in cui ogni dimensione era Tipo 2 "per sicurezza" — il risultato era un modello inutilmente complesso, ETL lenti, e nessuno che avesse mai interrogato la storia della dimensione "tipo_pagamento" o "canale_vendita".

La Tipo 2 ha un costo: complessità dell'ETL, crescita della tabella, necessità di gestire le chiavi surrogate nella fact. È un costo che vale la pena pagare quando il business ha bisogno della storia. Se non ce l'ha, la Tipo 1 è la scelta giusta.

Ci sono anche casi in cui la Tipo 2 non basta. Se serve sapere non solo *cosa* è cambiato ma anche *chi* ha fatto il cambio e *perché*, allora serve un audit trail — una tabella separata con un log completo delle modifiche. La Tipo 2 traccia le versioni, non le cause.

E per le dimensioni con cambiamenti molto frequenti — prezzi che cambiano ogni giorno, scoring che si aggiorna ogni ora — la Tipo 2 può generare una crescita insostenibile. In quei casi si valuta la **Tipo 6** (una combinazione di Tipo 1, 2 e 3) o approcci mini-dimension.

Ma per il caso più comune — anagrafiche clienti, prodotti, dipendenti, punti vendita — la Tipo 2 è lo strumento giusto. Semplice abbastanza da implementare senza framework esotici, potente abbastanza da restituire al business la dimensione che gli mancava: il tempo.

---

## Quello che ho imparato

Il direttore commerciale non sapeva di avere bisogno della storia finché non gli è servita. E quando gli è servita, il DWH non ce l'aveva.

Questo è il punto. Non si implementa la Tipo 2 perché "è best practice" o perché "Kimball lo dice nel capitolo 5". Si implementa perché un data warehouse senza storia è un database operativo con una {{< glossary term="star-schema" >}}star schema{{< /glossary >}} appiccicata sopra. Funziona per i report del mese corrente, ma non risponde alla domanda che prima o poi qualcuno farà: "Com'era prima?"

La domanda arriva sempre. La questione è se il tuo DWH è pronto a rispondere.

---

## Glossario

**[Chiave surrogata](/it/glossary/chiave-surrogata/)** — Identificativo numerico generato dal data warehouse, distinto dalla chiave naturale del sistema sorgente. Nella SCD Tipo 2 è indispensabile perché lo stesso record può avere più versioni, e la chiave naturale non è più univoca.

**[Fact table](/it/glossary/fact-table/)** — Tabella centrale dello star schema che contiene le misure numeriche (importi, quantità, conteggi) e le chiavi esterne verso le tabelle dimensionali. Ogni riga rappresenta un evento o una transazione di business.

**[Kimball](/it/glossary/kimball/)** — Ralph Kimball, autore della metodologia di progettazione data warehouse basata su dimensional modeling, star schema e processi ETL bottom-up. Il suo framework classifica le Slowly Changing Dimensions nei tipi da 0 a 7.

**[MERGE](/it/glossary/merge-sql/)** — Istruzione SQL che combina INSERT e UPDATE in un'unica operazione: se il record esiste lo aggiorna, se non esiste lo inserisce. In Oracle è anche nota come "upsert" ed è il meccanismo base dell'ETL per le dimensioni SCD.

**[Star schema](/it/glossary/star-schema/)** — Modello di dati tipico del data warehouse: una fact table al centro collegata a più tabelle dimensionali tramite chiavi esterne. Semplifica le query analitiche e ottimizza le performance delle aggregazioni.
