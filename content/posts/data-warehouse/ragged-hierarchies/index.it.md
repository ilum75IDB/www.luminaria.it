---
title: "Gerarchie sbilanciate: quando il cliente non ha un padre e il gruppo non ha un nonno"
description: "Un cliente con una gerarchia a tre livelli — Top Group, Group, Client — dove non tutti i rami sono completi. Come ho bilanciato una ragged hierarchy con il self-parenting: chi non ha padre diventa padre di sé stesso."
date: "2026-01-20T10:00:00+01:00"
draft: false
translationKey: "ragged_hierarchies"
tags: ["hierarchies", "dimensional-modeling", "etl", "oracle", "olap", "reporting"]
categories: ["data-warehouse"]
image: "ragged-hierarchies.cover.jpg"
---

Tre livelli. Top Group, Group, Client. Sembra una struttura banale — il tipo di gerarchia che disegni su una lavagna in cinque minuti e che qualsiasi tool di BI dovrebbe gestire senza problemi.

Poi scopri che non tutti i clienti hanno un gruppo. E che non tutti i gruppi hanno un top group. E che i report di aggregazione che il business ti chiede — fatturato per top group, numero clienti per gruppo, {{< glossary term="drill-down" >}}drill-down{{< /glossary >}} dal vertice alla foglia — producono risultati sbagliati o incompleti perché la gerarchia ha dei buchi.

In gergo tecnico si chiama **{{< glossary term="ragged-hierarchy" >}}ragged hierarchy{{< /glossary >}}**: una gerarchia in cui non tutti i rami raggiungono la stessa profondità. Nel mondo reale si chiama "il problema che nessuno vede finché non apre il report e i numeri non tornano."

---

## Il cliente e il modello originale

Il progetto era un data warehouse per un'azienda nel settore energy — distribuzione gas e servizi correlati. Il sistema sorgente gestiva un'anagrafica clienti con una struttura gerarchica: i clienti potevano essere raggruppati sotto un'entità commerciale (il **Group**), e i gruppi potevano a loro volta appartenere a un'entità superiore (il **Top Group**).

Il modello nella sorgente era una singola tabella con i riferimenti gerarchici:

``` sql
CREATE TABLE stg_clienti (
    client_id       NUMBER(10)    NOT NULL,
    client_name     VARCHAR2(100) NOT NULL,
    group_id        NUMBER(10),
    group_name      VARCHAR2(100),
    top_group_id    NUMBER(10),
    top_group_name  VARCHAR2(100),
    revenue         NUMBER(15,2),
    region          VARCHAR2(50),
    CONSTRAINT pk_stg_clienti PRIMARY KEY (client_id)
);
```

Ecco un campione dei dati:

``` sql
INSERT INTO stg_clienti VALUES (1001, 'Rossi Energia Srl',     10, 'Consorzio Nord',      100, 'Holding Nazionale',  125000.00, 'Lombardia');
INSERT INTO stg_clienti VALUES (1002, 'Bianchi Gas SpA',       10, 'Consorzio Nord',      100, 'Holding Nazionale',   89000.00, 'Piemonte');
INSERT INTO stg_clienti VALUES (1003, 'Verdi Distribuzione',   20, 'Gruppo Centro',       100, 'Holding Nazionale',   67000.00, 'Toscana');
INSERT INTO stg_clienti VALUES (1004, 'Neri Servizi',          20, 'Gruppo Centro',       NULL, NULL,                  45000.00, 'Lazio');
INSERT INTO stg_clienti VALUES (1005, 'Gialli Utilities',      NULL, NULL,                NULL, NULL,                  38000.00, 'Sicilia');
INSERT INTO stg_clienti VALUES (1006, 'Blu Energia',           NULL, NULL,                NULL, NULL,                  52000.00, 'Sardegna');
INSERT INTO stg_clienti VALUES (1007, 'Viola Gas Srl',         30, 'Rete Sud',            NULL, NULL,                  71000.00, 'Campania');
INSERT INTO stg_clienti VALUES (1008, 'Arancio Distribuzione', 30, 'Rete Sud',            NULL, NULL,                  33000.00, 'Calabria');
```

Guardate i dati con attenzione. Ci sono quattro situazioni diverse:

- **Client 1001, 1002, 1003**: gerarchia completa — Client → Group → Top Group
- **Client 1004**: ha un Group ma il Group non ha un Top Group
- **Client 1005, 1006**: nessun Group, nessun Top Group — clienti diretti
- **Client 1007, 1008**: hanno un Group (Rete Sud) ma il Group non ha un Top Group

Questa è una ragged hierarchy. Tre livelli sulla carta, ma nella realtà i rami hanno profondità diverse.

---

## Il problema: i report non tornano

Il business chiedeva un report semplice: fatturato aggregato per Top Group, con possibilità di drill-down per Group e poi per Client. Una richiesta ragionevole — il tipo di cosa che ti aspetti da qualsiasi DWH.

La query più naturale:

``` sql
SELECT top_group_name,
       group_name,
       COUNT(*)        AS num_clienti,
       SUM(revenue)    AS fatturato_totale
FROM   stg_clienti
GROUP BY top_group_name, group_name
ORDER BY top_group_name, group_name;
```

Il risultato:

``` text
TOP_GROUP_NAME      GROUP_NAME        NUM_CLIENTI  FATTURATO_TOTALE
------------------  ----------------  -----------  ----------------
Holding Nazionale   Consorzio Nord              2        214000.00
Holding Nazionale   Gruppo Centro               1         67000.00
(null)              Gruppo Centro               1         45000.00
(null)              Rete Sud                    2        104000.00
(null)              (null)                      2         90000.00
```

Cinque righe. E almeno tre problemi.

Il Gruppo Centro appare due volte: una sotto "Holding Nazionale" (il client 1003 che ha il top group) e una sotto NULL (il client 1004 il cui top group è NULL). Lo stesso gruppo, spaccato su due righe, con totali separati. Chiunque guardi questo report penserà che Gruppo Centro abbia 67K di fatturato sotto la holding e 45K da qualche altra parte. In realtà è un unico gruppo con 112K totali.

I clienti diretti (Gialli Utilities e Blu Energia) finiscono in una riga con due NULL. Il management non sa cosa farsene di una riga senza nome.

Il totale per Top Group è sbagliato perché mancano le righe con NULL. Se sommi solo le righe con un top group, perdi 239K di fatturato — il 30% del totale.

---

## L'approccio classico: COALESCE e preghiere

La prima reazione, quella che vedo fare nel 90% dei casi, è aggiungere {{< glossary term="coalesce" >}}`COALESCE`{{< /glossary >}} nella query:

``` sql
SELECT COALESCE(top_group_name, group_name, client_name) AS top_group_name,
       COALESCE(group_name, client_name)                 AS group_name,
       client_name,
       revenue
FROM   stg_clienti;
```

Funziona? In un certo senso sì — riempie i buchi. Ma introduce problemi nuovi.

Il client "Gialli Utilities" ora appare come Top Group, Group e Client contemporaneamente. Se il business vuole contare quanti Top Group ci sono, il numero è gonfiato. Se vuole filtrare per "veri" top group, non ha modo di distinguerli dai clienti promossi a top group dalla COALESCE.

E questo è il caso semplice, con tre livelli. Ho visto gerarchie a cinque livelli gestite con catene di COALESCE nidificati, CASE WHEN multipli, e una logica di report talmente contorta che nessuno osava più toccarla. Ogni nuova richiesta del business richiedeva una modifica a cascata su tutte le query.

Il problema di fondo è che la COALESCE è un cerotto applicato nel layer di presentazione. Non risolve il problema strutturale: la gerarchia è incompleta e il modello dimensionale non lo sa.

---

## La soluzione: self-parenting

Il principio è semplice: **chi non ha un padre diventa padre di sé stesso**. Questa tecnica si chiama {{< glossary term="self-parenting" >}}self-parenting{{< /glossary >}}.

Un Client senza Group? Quel client diventa il proprio Group. Un Group senza Top Group? Quel group diventa il proprio Top Group. In questo modo la gerarchia è sempre completa a tre livelli, senza buchi, senza NULL.

Non è un trucco. È una tecnica standard nel dimensional modeling, descritta da {{< glossary term="kimball" >}}Kimball{{< /glossary >}} e usata in produzione da decenni. L'idea è che la dimensione gerarchica nel DWH deve essere **bilanciata**: ogni record deve avere un valore valido per ogni livello della gerarchia. Se la sorgente non lo garantisce, lo garantisce l'{{< glossary term="etl" >}}ETL{{< /glossary >}}.

### La tabella dimensionale

``` sql
CREATE TABLE dim_client_hierarchy (
    client_key      NUMBER(10)    NOT NULL,
    client_id       NUMBER(10)    NOT NULL,
    client_name     VARCHAR2(100) NOT NULL,
    group_id        NUMBER(10)    NOT NULL,
    group_name      VARCHAR2(100) NOT NULL,
    top_group_id    NUMBER(10)    NOT NULL,
    top_group_name  VARCHAR2(100) NOT NULL,
    region          VARCHAR2(50),
    is_direct_client  CHAR(1)     DEFAULT 'N',
    is_standalone_group CHAR(1)   DEFAULT 'N',
    CONSTRAINT pk_dim_client_hier PRIMARY KEY (client_key)
);
```

Notate due cose. Primo: nessuna colonna è nullable. Group e Top Group sono `NOT NULL`. Secondo: ho aggiunto due flag — `is_direct_client` e `is_standalone_group` — che permettono di distinguere i record bilanciati artificialmente da quelli che hanno una gerarchia naturale. Questo è importante: il business deve poter filtrare i "veri" top group dai clienti promossi.

### La logica ETL

``` sql
INSERT INTO dim_client_hierarchy (
    client_key, client_id, client_name,
    group_id, group_name,
    top_group_id, top_group_name,
    region, is_direct_client, is_standalone_group
)
SELECT
    client_id AS client_key,
    client_id,
    client_name,
    -- Se non ha un group, il client diventa group di sé stesso
    COALESCE(group_id, client_id)          AS group_id,
    COALESCE(group_name, client_name)      AS group_name,
    -- Se non ha un top group, il group (o il client) diventa top group di sé stesso
    COALESCE(top_group_id, group_id, client_id)       AS top_group_id,
    COALESCE(top_group_name, group_name, client_name)  AS top_group_name,
    region,
    CASE WHEN group_id IS NULL THEN 'Y' ELSE 'N' END  AS is_direct_client,
    CASE WHEN group_id IS NOT NULL AND top_group_id IS NULL
         THEN 'Y' ELSE 'N' END                        AS is_standalone_group
FROM stg_clienti;
```

Guardate la cascata di COALESCE nella trasformazione. La logica è:

- `group_id`: se il client ha un group, usa quello; altrimenti usa il client stesso
- `top_group_id`: se c'è un top group, usa quello; se non c'è ma c'è un group, usa il group; se non c'è neanche il group, usa il client

Ogni livello "mancante" viene riempito dal livello immediatamente inferiore. Il risultato è una gerarchia sempre completa.

### Il risultato dopo il bilanciamento

``` sql
SELECT client_key, client_name, group_name, top_group_name,
       is_direct_client, is_standalone_group
FROM   dim_client_hierarchy
ORDER BY top_group_id, group_id, client_id;
```

``` text
KEY   CLIENT_NAME           GROUP_NAME        TOP_GROUP_NAME      DIRECT  STANDALONE
----  --------------------  ----------------  ------------------  ------  ----------
1001  Rossi Energia Srl     Consorzio Nord    Holding Nazionale   N       N
1002  Bianchi Gas SpA       Consorzio Nord    Holding Nazionale   N       N
1003  Verdi Distribuzione   Gruppo Centro     Holding Nazionale   N       N
1004  Neri Servizi          Gruppo Centro     Gruppo Centro       N       Y
1007  Viola Gas Srl         Rete Sud          Rete Sud            N       Y
1008  Arancio Distribuzione Rete Sud          Rete Sud            N       Y
1005  Gialli Utilities      Gialli Utilities  Gialli Utilities    Y       N
1006  Blu Energia           Blu Energia       Blu Energia         Y       N
```

Otto righe, zero NULL. Ogni client ha un group e un top group. I flag dicono la verità: Gialli e Blu sono clienti diretti (self-parented a tutti i livelli), Gruppo Centro e Rete Sud sono gruppi standalone (self-parented al livello top group).

---

## I report dopo il bilanciamento

La stessa query di aggregazione che prima produceva risultati spezzati:

``` sql
SELECT top_group_name,
       group_name,
       COUNT(*)        AS num_clienti,
       SUM(f.revenue)  AS fatturato_totale
FROM   dim_client_hierarchy d
JOIN   stg_clienti f ON d.client_id = f.client_id
GROUP BY top_group_name, group_name
ORDER BY top_group_name, group_name;
```

``` text
TOP_GROUP_NAME      GROUP_NAME          NUM_CLIENTI  FATTURATO_TOTALE
------------------  ------------------  -----------  ----------------
Blu Energia         Blu Energia                   1         52000.00
Gialli Utilities    Gialli Utilities              1         38000.00
Gruppo Centro       Gruppo Centro                 1         45000.00
Holding Nazionale   Consorzio Nord                2        214000.00
Holding Nazionale   Gruppo Centro                 1         67000.00
Rete Sud            Rete Sud                      2        104000.00
```

Nessun NULL. Ogni riga ha un top group e un group identificabili. I totali tornano.

E se il business vuole solo i "veri" top group, escludendo i clienti promossi:

``` sql
SELECT top_group_name,
       COUNT(*)        AS num_clienti,
       SUM(f.revenue)  AS fatturato_totale
FROM   dim_client_hierarchy d
JOIN   stg_clienti f ON d.client_id = f.client_id
WHERE  d.is_direct_client = 'N'
AND    d.is_standalone_group = 'N'
GROUP BY top_group_name
ORDER BY fatturato_totale DESC;
```

``` text
TOP_GROUP_NAME      NUM_CLIENTI  FATTURATO_TOTALE
------------------  -----------  ----------------
Holding Nazionale             3        281000.00
```

I flag rendono tutto filtrabile. Nessuna logica condizionale nel report, nessun CASE WHEN, nessuna COALESCE. Il modello dimensionale contiene già tutta l'informazione necessaria.

---

## Il drill-down completo

Il vero test di una gerarchia bilanciata è il drill-down: dal livello più alto al più basso, senza sorprese.

``` sql
-- Livello 1: totale per Top Group
SELECT top_group_name,
       COUNT(DISTINCT group_id) AS num_gruppi,
       COUNT(*)                 AS num_clienti,
       SUM(f.revenue)           AS fatturato
FROM   dim_client_hierarchy d
JOIN   stg_clienti f ON d.client_id = f.client_id
GROUP BY top_group_name
ORDER BY fatturato DESC;
```

``` text
TOP_GROUP_NAME      NUM_GRUPPI  NUM_CLIENTI  FATTURATO
------------------  ----------  -----------  ----------
Holding Nazionale            2            3   281000.00
Rete Sud                     1            2   104000.00
Blu Energia                  1            1    52000.00
Gruppo Centro                1            1    45000.00
Gialli Utilities             1            1    38000.00
```

``` sql
-- Livello 2: drill-down dentro "Holding Nazionale"
SELECT group_name,
       COUNT(*)       AS num_clienti,
       SUM(f.revenue) AS fatturato
FROM   dim_client_hierarchy d
JOIN   stg_clienti f ON d.client_id = f.client_id
WHERE  d.top_group_name = 'Holding Nazionale'
GROUP BY group_name
ORDER BY fatturato DESC;
```

``` text
GROUP_NAME        NUM_CLIENTI  FATTURATO
----------------  -----------  ----------
Consorzio Nord              2   214000.00
Gruppo Centro               1    67000.00
```

``` sql
-- Livello 3: drill-down dentro "Consorzio Nord"
SELECT client_name, f.revenue
FROM   dim_client_hierarchy d
JOIN   stg_clienti f ON d.client_id = f.client_id
WHERE  d.group_name = 'Consorzio Nord'
ORDER BY f.revenue DESC;
```

``` text
CLIENT_NAME          REVENUE
-------------------  ----------
Rossi Energia Srl     125000.00
Bianchi Gas SpA        89000.00
```

Tre livelli di drill-down, zero NULL, zero logica condizionale. La gerarchia è bilanciata e i numeri tornano a ogni livello.

---

## Perché non basta la COALESCE nel report

Qualcuno potrebbe obiettare: "Ma la COALESCE nel report fa la stessa cosa, senza bisogno di modificare il modello."

No. Fa qualcosa di simile, ma con tre differenze fondamentali.

**Primo: la COALESCE va ripetuta ovunque.** Ogni query, ogni report, ogni dashboard, ogni estrazione. Se hai venti report che usano la gerarchia, devi ricordarti di applicare la COALESCE in tutti e venti. E quando arriva il ventunesimo, devi ricordarti di nuovo. Il self-parenting nel modello dimensionale lo fai una volta nell'ETL e basta.

**Secondo: la COALESCE non distingue.** Non sai se "Gialli Utilities" nel campo top_group è un vero top group o un client promosso. Con i flag nel modello dimensionale hai l'informazione per filtrare. Senza flag, il business è cieco.

**Terzo: le performance.** Un GROUP BY con COALESCE su colonne nullable è meno efficiente di un GROUP BY su colonne NOT NULL. L'optimizer di Oracle gestisce meglio le colonne con vincoli NOT NULL — può eliminare controlli sui NULL, usare indici in modo più aggressivo, e produrre piani di esecuzione più semplici. Su una tabella dimensionale con milioni di righe, la differenza si vede.

---

## Quando usare il self-parenting (e quando no)

Il self-parenting funziona bene quando:

- La gerarchia ha un **numero fisso di livelli** (tipicamente 2-5)
- Il caso d'uso principale è l'**aggregazione e il drill-down** nei report
- Il modello è un **data warehouse** o un cubo OLAP
- I livelli mancanti sono l'eccezione, non la regola

Non funziona bene quando:

- La gerarchia è **ricorsiva** con profondità variabile (es. organigrammi aziendali a N livelli)
- Serve navigare il **grafo** delle relazioni (es. reti sociali, supply chain)
- Il modello è **OLTP** e il self-parenting creerebbe ambiguità nelle logiche applicative
- I livelli della gerarchia cambiano frequentemente nel tempo

Per le gerarchie ricorsive a profondità variabile, l'approccio corretto è diverso: tabelle di bridge, closure table o modelli parent-child con CTE ricorsive. Sono strumenti potenti ma risolvono un problema diverso.

Il self-parenting risolve un problema specifico — gerarchie a livelli fissi con rami incompleti — e lo risolve nel modo più semplice possibile: bilanciando la struttura a monte, nel modello, anziché a valle, nei report.

---

## La regola che mi guida

Ho progettato decine di dimensioni gerarchiche in vent'anni di data warehouse. La regola che mi porto dietro è sempre la stessa:

**Se il report ha bisogno di logica condizionale per gestire la gerarchia, il problema è nel modello, non nel report.**

Un report dovrebbe fare GROUP BY e JOIN. Se deve anche decidere come gestire i livelli mancanti, sta facendo il lavoro dell'ETL. E un report che fa il lavoro dell'ETL è un report che prima o poi si rompe.

Il self-parenting non è elegante. Non è sofisticato. È una soluzione che un informatico appena laureato potrebbe trovare brutta. Ma funziona, è manutenibile, e trasforma un problema che infesta ogni singolo report in un problema che si risolve una volta sola, in un punto solo, e non torna più.

A volte la soluzione migliore è la più semplice. Questa è una di quelle volte.

---

## Glossario

**[COALESCE](/it/glossary/coalesce/)** — Funzione SQL che restituisce il primo valore non NULL da una lista di espressioni. Spesso usata come workaround per le gerarchie incomplete nei report, ma non risolve il problema strutturale nel modello dimensionale.

**[Drill-down](/it/glossary/drill-down/)** — Navigazione nei report dal livello aggregato al livello di dettaglio (es. da Top Group a Group a Client). Richiede una gerarchia completa e bilanciata per funzionare correttamente senza NULL o righe mancanti.

**[OLAP](/it/glossary/olap/)** — Online Analytical Processing — elaborazione orientata all'analisi multidimensionale dei dati, tipica dei data warehouse e dei cubi di analisi. Contrapposta all'OLTP (Online Transaction Processing) dei sistemi transazionali.

**[Ragged hierarchy](/it/glossary/ragged-hierarchy/)** — Gerarchia in cui non tutti i rami raggiungono la stessa profondità: alcuni livelli intermedi sono assenti. Tipica nelle anagrafiche clienti, prodotti e strutture organizzative dove non tutte le entità hanno la stessa struttura gerarchica.

**[Self-parenting](/it/glossary/self-parenting/)** — Tecnica di bilanciamento delle gerarchie sbilanciate: chi non ha un padre diventa padre di sé stesso. Il livello mancante viene riempito con i dati del livello inferiore, eliminando i NULL dalla dimensione e garantendo un drill-down corretto.
