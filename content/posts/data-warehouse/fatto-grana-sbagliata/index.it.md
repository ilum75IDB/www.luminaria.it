---
title: "Fatto a grana sbagliata: quando la fact table non risponde alle domande giuste"
description: "Una fact table costruita sul totale mensile della fattura sembrava perfetta. Poi il business ha chiesto il dettaglio per prodotto, per riga, per cliente. E il data warehouse è diventato muto."
date: "2025-10-21T10:00:00+01:00"
draft: false
translationKey: "fatto_grana_sbagliata"
tags: ["data-warehouse", "fact-table", "granularity", "grain", "dimensional-modeling", "kimball"]
categories: ["Data Warehouse"]
image: "fatto-grana-sbagliata.cover.jpg"
---

La riunione era cominciata bene. Il direttore commerciale di un'azienda di distribuzione industriale — una sessantina di milioni di fatturato, tremila clienti attivi, catalogo con dodicimila referenze — aveva aperto la presentazione del nuovo data warehouse con un sorriso. I numeri tornavano, i cruscotti erano belli, i totali mensili per agente e per zona battevano con la contabilità.

Poi qualcuno ha fatto la domanda sbagliata. O meglio, quella giusta.

*"Posso vedere quanto ha comprato il cliente Bianchi nel mese di marzo, riga per riga, prodotto per prodotto?"*

Silenzio.

Il responsabile BI ha guardato me. Io ho guardato lo schermo. Lo schermo mostrava una {{< glossary term="fact-table" >}}fact table{{< /glossary >}} con una riga per cliente per mese: importo totale fatturato, quantità totale, numero fatture. Nessun dettaglio. Nessuna riga di fattura. Nessun prodotto.

Quella fact table rispondeva a una sola domanda: *quanto ha fatturato ciascun cliente in un dato mese?* Tutto il resto — per prodotto, per famiglia merceologica, per singola fattura — era fuori portata.

## 🔍 Il grain: la decisione che determina tutto

Nel {{< glossary term="star-schema" >}}dimensional modeling{{< /glossary >}}, il **{{< glossary term="grain" >}}grain{{< /glossary >}}** (la grana) della fact table è la prima decisione da prendere. Non la seconda, non una tra tante: la prima. Kimball lo ripete in ogni capitolo, e ha ragione.

Il grain risponde alla domanda: *cosa rappresenta una singola riga della fact table?*

Nel progetto che ho descritto, chi aveva progettato il modello aveva scelto un grain mensile-cliente: una riga = un cliente in un mese. I motivi sembravano ragionevoli: il sistema sorgente esportava un riepilogo mensile, il caricamento era veloce, le tabelle erano piccole, le query semplici.

Ma il grain determina le domande a cui il data warehouse può rispondere. Se la grana è il riepilogo mensile per cliente, non puoi scendere sotto quel livello. Non puoi fare {{< glossary term="drill-down" >}}drill-down{{< /glossary >}} per prodotto. Non puoi sapere se il cliente Bianchi ha comprato dieci volte lo stesso articolo o dieci articoli diversi. Non puoi confrontare margini per famiglia merceologica.

Hai un totale. Punto.

## 📊 I numeri del problema

La fact table originale aveva questa struttura:

```sql
CREATE TABLE fact_fatturato_mensile (
    sk_cliente        INT NOT NULL,
    sk_tempo          INT NOT NULL,  -- mese (YYYYMM)
    sk_agente         INT NOT NULL,
    sk_zona           INT NOT NULL,
    importo_totale    DECIMAL(15,2),
    quantita_totale   INT,
    num_fatture       INT,
    num_righe         INT,
    FOREIGN KEY (sk_cliente) REFERENCES dim_cliente(sk_cliente),
    FOREIGN KEY (sk_tempo)   REFERENCES dim_tempo(sk_tempo)
);
```

Righe in tabella: circa 180.000 all'anno (3.000 clienti × 12 mesi × qualche variazione). Piccola, veloce, facile da caricare. L'{{< glossary term="etl" >}}ETL{{< /glossary >}} girava in meno di cinque minuti.

Il problema? Le {{< glossary term="additive-measure" >}}misure additive{{< /glossary >}} erano già aggregate. `importo_totale` era la somma di tutte le righe di fattura del mese. Impossibile risalire alla composizione. Come avere il totale di uno scontrino senza sapere cosa hai comprato.

## 🏗️ La ristrutturazione: scendere alla riga di fattura

La soluzione era una sola: cambiare il grain. Portare la fact table al livello più basso disponibile nel sistema sorgente — la singola riga di fattura.

```sql
CREATE TABLE fact_fatturato_riga (
    sk_riga_fattura   INT PRIMARY KEY,
    sk_fattura        INT NOT NULL,
    sk_cliente        INT NOT NULL,
    sk_prodotto       INT NOT NULL,
    sk_tempo          INT NOT NULL,  -- giorno (YYYYMMDD)
    sk_agente         INT NOT NULL,
    sk_zona           INT NOT NULL,
    sk_famiglia       INT NOT NULL,
    quantita          INT,
    prezzo_unitario   DECIMAL(12,4),
    importo_riga      DECIMAL(15,2),
    sconto_perc       DECIMAL(5,2),
    importo_netto     DECIMAL(15,2),
    costo_prodotto    DECIMAL(15,2),
    margine           DECIMAL(15,2),
    FOREIGN KEY (sk_cliente)  REFERENCES dim_cliente(sk_cliente),
    FOREIGN KEY (sk_prodotto) REFERENCES dim_prodotto(sk_prodotto),
    FOREIGN KEY (sk_tempo)    REFERENCES dim_tempo(sk_tempo)
);
```

Righe in tabella: circa 2,4 milioni all'anno (3.000 clienti × ~800 righe/anno in media). Un ordine di grandezza in più. Ma ogni riga porta con sé il dettaglio completo: quale prodotto, quale fattura, quale prezzo, quale sconto, quale margine.

## ⚡ L'impatto sull'ETL

Il cambiamento di grain ha avuto un effetto a cascata sull'ETL che nessuno aveva previsto — o meglio, che chi aveva scelto il grain aggregato aveva evitato di affrontare.

**Nuove dimensioni necessarie:**

| Dimensione       | Cardinalità | Note                                |
|-------------------|-------------|-------------------------------------|
| `dim_prodotto`    | ~12.000     | Non esisteva: prima non serviva     |
| `dim_famiglia`    | ~180        | Gerarchia merceologica a 3 livelli  |
| `dim_fattura`     | ~45.000/anno| Header fattura con dati di testata  |

**Nuova finestra di caricamento:**

| Fase                | Prima   | Dopo      |
|---------------------|---------|-----------|
| Estrazione          | 40 sec  | 3 min     |
| Trasformazione      | 1 min   | 8 min     |
| Caricamento fact    | 30 sec  | 4 min     |
| **Totale**          | **~2 min** | **~15 min** |

Quindici minuti contro due. Un prezzo accettabile per un data warehouse che adesso rispondeva a domande reali.

## 🔬 Le domande che prima erano impossibili

Con il nuovo grain, le query che il business voleva diventavano banali:

**Dettaglio acquisti di un cliente per prodotto:**

```sql
SELECT
    c.ragione_sociale,
    p.codice_prodotto,
    p.descrizione,
    SUM(f.quantita)       AS pezzi,
    SUM(f.importo_netto)  AS fatturato_netto,
    SUM(f.margine)        AS margine_totale
FROM fact_fatturato_riga f
JOIN dim_cliente  c ON f.sk_cliente  = c.sk_cliente
JOIN dim_prodotto p ON f.sk_prodotto = p.sk_prodotto
JOIN dim_tempo    t ON f.sk_tempo    = t.sk_tempo
WHERE c.ragione_sociale = 'Bianchi Srl'
  AND t.anno = 2024
  AND t.mese = 3
GROUP BY c.ragione_sociale, p.codice_prodotto, p.descrizione
ORDER BY fatturato_netto DESC;
```

**Top 10 prodotti per marginalità in un trimestre:**

```sql
SELECT
    p.codice_prodotto,
    p.descrizione,
    fm.desc_famiglia,
    SUM(f.importo_netto)  AS fatturato,
    SUM(f.margine)        AS margine,
    ROUND(SUM(f.margine) / NULLIF(SUM(f.importo_netto), 0) * 100, 1) AS margine_perc
FROM fact_fatturato_riga f
JOIN dim_prodotto  p  ON f.sk_prodotto = p.sk_prodotto
JOIN dim_famiglia  fm ON f.sk_famiglia = fm.sk_famiglia
JOIN dim_tempo     t  ON f.sk_tempo    = t.sk_tempo
WHERE t.anno = 2024
  AND t.trimestre = 1
GROUP BY p.codice_prodotto, p.descrizione, fm.desc_famiglia
ORDER BY margine DESC
LIMIT 10;
```

**Confronto agente: fatturato medio per riga di fattura:**

```sql
SELECT
    a.nome_agente,
    COUNT(*)                     AS num_righe,
    SUM(f.importo_netto)         AS fatturato_totale,
    ROUND(AVG(f.importo_netto), 2) AS media_per_riga
FROM fact_fatturato_riga f
JOIN dim_agente a ON f.sk_agente = a.sk_agente
JOIN dim_tempo  t ON f.sk_tempo  = t.sk_tempo
WHERE t.anno = 2024
GROUP BY a.nome_agente
ORDER BY fatturato_totale DESC;
```

Nessuna di queste query era possibile con il grain mensile-cliente. Zero. Non era un problema di ottimizzazione o di indici — era un problema strutturale, scritto nel DNA del modello.

## 📋 La regola di Kimball che avevamo ignorato

Ralph Kimball lo dice in modo chiaro: *"sempre modellare al livello di dettaglio più fine disponibile nel sistema sorgente"*.

Non è un suggerimento. Non è un'opzione tra tante. È il principio fondante del dimensional modeling. E il motivo è semplice: si può sempre aggregare dal dettaglio al totale, ma non si può mai disaggregare un totale nel suo dettaglio.

L'aggregazione è un'operazione irreversibile. Come mescolare i colori: dal rosso e dal giallo puoi ottenere l'arancione, ma dall'arancione non torni più ai colori originali.

Nel caso del nostro progetto, la scelta del grain aggregato era stata dettata dalla pigrizia progettuale, non da un vincolo tecnico. Il sistema sorgente aveva il dettaglio per riga di fattura — semplicemente nessuno aveva voluto affrontare la complessità di modellarlo, gestire le dimensioni aggiuntive, allungare i tempi dell'ETL.

Il risultato? Un data warehouse che andava ricostruito da zero dopo sei mesi dal go-live.

## 🎯 Quando il grain aggregato ha senso

Non sempre la grana fine è l'unica risposta. Ci sono casi legittimi per le fact table aggregate:

- **Tabelle di aggregazione** (aggregate fact table) affiancate alla tabella di dettaglio, per velocizzare le query più frequenti
- **Snapshot periodici** dove il business ragiona effettivamente per periodo (saldo mensile di un conto corrente, giacenza di magazzino a fine settimana)
- **Vincoli di sorgente** quando il sistema a monte non espone il dettaglio e non c'è modo di ottenerlo

Ma la regola è: parti dal dettaglio, poi aggrega. Mai il contrario. Le aggregate fact table sono un'ottimizzazione, non un sostituto della grana fine.

Nel nostro caso, dopo la ristrutturazione, abbiamo creato anche una vista materializzata con il riepilogo mensile per cliente — la stessa struttura di prima — per i cruscotti direzionali che non avevano bisogno del dettaglio. Il meglio dei due mondi, senza sacrificare nulla.

## Quello che ho imparato

Quel progetto mi ha insegnato una cosa che porto con me in ogni incarico successivo: la prima mezz'ora di progettazione di un data warehouse, quella in cui si decide il grain, vale più di tutte le ottimizzazioni che verranno dopo. Un ETL perfetto, indici calibrati, hardware potente — niente di tutto questo compensa un grain sbagliato.

Se la tua fact table non risponde alle domande del business, non è colpa delle query. È colpa del modello. E il modello si decide al grain.

------------------------------------------------------------------------

## Glossario

**[Grain](/it/glossary/grain/)** — Il livello di dettaglio (grana) di una fact table nel data warehouse. Determina cosa rappresenta ciascuna riga e quali domande il modello può soddisfare. È la prima decisione da prendere nella progettazione dimensionale.

**[Fact table](/it/glossary/fact-table/)** — Tabella centrale dello star schema che contiene le misure numeriche (importi, quantità, margini) e le chiavi esterne verso le dimensioni. La sua grana determina il livello di analisi possibile.

**[Additive Measure](/it/glossary/additive-measure/)** — Misura numerica che può essere sommata lungo tutte le dimensioni (es. importo, quantità). Una volta aggregata a livello superiore, il dettaglio originale è perso irreversibilmente.

**[Drill-down](/it/glossary/drill-down/)** — Navigazione nei report dal livello aggregato al dettaglio, lungo una gerarchia. Possibile solo se la fact table contiene dati al livello di grana sufficiente.

**[Star Schema](/it/glossary/star-schema/)** — Modello di dati con una fact table al centro e tabelle dimensionali collegate. La struttura più usata nei data warehouse per la semplicità delle query analitiche.

**[ETL](/it/glossary/etl/)** — Extract, Transform, Load: il processo di estrazione, trasformazione e caricamento dei dati nel data warehouse. Un cambio di grain impatta direttamente tempi e complessità dell'ETL.
