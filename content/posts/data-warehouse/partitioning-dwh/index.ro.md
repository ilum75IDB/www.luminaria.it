---
title: "Partitioning în DWH: când 3 ani de date sunt prea mulți"
description: "O fact table de 800 de milioane de rânduri fără partitioning, query-uri trimestriale care durau 12 minute și un business care voia răspunsuri în timp real. Cum am implementat range partitioning lunar și am adus timpii la 40 de secunde."
date: "2026-04-07T10:00:00+01:00"
draft: false
translationKey: "partitioning_dwh"
tags: ["partitioning", "performance", "oracle", "fact-table", "data-warehouse"]
categories: ["data-warehouse"]
image: "partitioning-dwh.cover.jpg"
---

Săptămâna trecută un coleg mi-a povestit despre un proiect unde query-urile pe data warehouse-ul nu mai returnau în timpi rezonabili. „Cât durează raportul trimestrial?" l-am întrebat. „Douăsprezece minute." „Și înainte?" „Un minut și jumătate."

Nu a trebuit să mai întreb nimic. Cunoșteam deja scenariul.

O {{< glossary term="fact-table" >}}fact table{{< /glossary >}} care pornește mică, crește în fiecare zi, și nimeni nu se preocupă de structura fizică până într-o zi când query-urile nu mai revin. Nu e un bug, nu e o eroare de cod. E greutatea datelor care în final se face simțită.

---

## Contextul: retail și trei ani de bonuri fiscale

Proiectul era în sectorul de retail alimentar — un lanț de supermarketuri cu aproximativ două sute de puncte de vânzare, în jur de o sută de milioane de euro cifră de afaceri anuală, și un data warehouse Oracle 19c care colecta totul: vânzări, returnări, mișcări de stoc, promoții.

Tabela din centrul problemei se numea `FACT_VENDITE`. Fiecare rând era o linie de bon — un bon mediu are opt linii, înmulțit cu treizeci de mii de bonuri pe zi pe două sute de magazine, asta înseamnă aproximativ 48 de milioane de rânduri pe lună. În trei ani se acumulaseră 800 de milioane de rânduri.

Structura era aceasta:

```sql
CREATE TABLE fact_vendite (
    vendita_id       NUMBER(18)    NOT NULL,
    data_vendita     DATE          NOT NULL,
    punto_vendita_id NUMBER(10)    NOT NULL,
    prodotto_id      NUMBER(10)    NOT NULL,
    cliente_id       NUMBER(10),
    quantita         NUMBER(10,2)  NOT NULL,
    importo          NUMBER(12,2)  NOT NULL,
    sconto           NUMBER(8,2)   DEFAULT 0,
    tipo_pagamento   VARCHAR2(20),
    CONSTRAINT pk_fact_vendite PRIMARY KEY (vendita_id)
);
```

Un singur index pe primary key, un index pe `data_vendita` și unul compozit pe `(punto_vendita_id, data_vendita)`. Niciun partiționism. Opt sute de milioane de rânduri într-o singură tabelă monolitică.

## 🔍 Simptomul: full table scan pe 800 de milioane de rânduri

Query-urile analitice ale DWH-ului lucrau aproape întotdeauna pe perioade. Vânzări pe ultimul trimestru pe punct de vânzare. Comparație an la an pe categorie de produs. Marje lunare pe regiune. Toate query-urile cu un filtru pe `data_vendita`.

Raportul trimestrial era acesta:

```sql
SELECT pv.regione,
       cat.famiglia,
       SUM(f.importo)    AS fatturato,
       SUM(f.quantita)   AS pezzi_venduti,
       SUM(f.sconto)     AS sconto_totale
FROM   fact_vendite f
JOIN   dim_punto_vendita pv  ON f.punto_vendita_id = pv.punto_vendita_id
JOIN   dim_prodotto cat      ON f.prodotto_id      = cat.prodotto_id
WHERE  f.data_vendita BETWEEN DATE '2025-10-01' AND DATE '2025-12-31'
GROUP BY pv.regione, cat.famiglia
ORDER BY fatturato DESC;
```

Predicatul pe `data_vendita` ar fi trebuit să folosească indexul. Și chiar o făcea — cu un an înainte, când tabela avea 500 de milioane de rânduri. Dar cu 800 de milioane, optimizer-ul decisese că indexul nu mai merita. Calculul era simplu: un trimestru = aproximativ 8% din totalul rândurilor. Cu un index range scan, Oracle ar fi avut nevoie de 64 de milioane de accesări aleatorii la blocuri. Un {{< glossary term="full-table-scan" >}}full table scan{{< /glossary >}} secvențial costa mai puțin.

Și asta a făcut: a citit 800 de milioane de rânduri pentru a returna 64 de milioane.

```
--------------------------------------------------------------
| Id | Operation          | Name         | Rows  | Bytes    |
--------------------------------------------------------------
|  0 | SELECT STATEMENT   |              |  1200 |    84K   |
|  1 |  SORT GROUP BY     |              |  1200 |    84K   |
|* 2 |   HASH JOIN        |              |   64M | 4480M    |
|  3 |    TABLE ACCESS FULL| DIM_PRODOTTO |  12K  |  168K    |
|* 4 |    HASH JOIN        |              |   64M | 3520M    |
|  5 |     TABLE ACCESS FULL| DIM_PUNTO_VENDITA | 200 | 4000 |
|* 6 |     TABLE ACCESS FULL| FACT_VENDITE| 800M  | 40G      |
--------------------------------------------------------------
```

Patruzeci de gigabytes de I/O pentru un query trimestrial. Într-un mediu unde buffer pool-ul era dimensionat la 16 GB, asta însemna citirea de mai mult de două ori a întregului cache de pe disc. Douăsprezece minute.

## 🏗️ Soluția: range partitioning lunar

Partiționalismul range pe dată este alegerea naturală pentru o fact table într-un data warehouse. Datele intră în ordine cronologică, query-urile filtrează pe perioadă, datele vechi devin reci și cele noi sunt fierbinți. Data este cheia de partiționare perfectă.

Am ales partiționarea lunară — 36 de partiții pentru trei ani de istoric, plus o partiție pentru datele curente. Fiecare partiție conținea aproximativ 48 de milioane de rânduri: un volum gestionabil pentru query-uri și pentru operațiunile de mentenanță.

```sql
CREATE TABLE fact_vendite_part (
    vendita_id       NUMBER(18)    NOT NULL,
    data_vendita     DATE          NOT NULL,
    punto_vendita_id NUMBER(10)    NOT NULL,
    prodotto_id      NUMBER(10)    NOT NULL,
    cliente_id       NUMBER(10),
    quantita         NUMBER(10,2)  NOT NULL,
    importo          NUMBER(12,2)  NOT NULL,
    sconto           NUMBER(8,2)   DEFAULT 0,
    tipo_pagamento   VARCHAR2(20)
)
PARTITION BY RANGE (data_vendita) (
    PARTITION p_2023_01 VALUES LESS THAN (DATE '2023-02-01'),
    PARTITION p_2023_02 VALUES LESS THAN (DATE '2023-03-01'),
    PARTITION p_2023_03 VALUES LESS THAN (DATE '2023-04-01'),
    -- ... 33 partiții intermediare ...
    PARTITION p_2025_12 VALUES LESS THAN (DATE '2026-01-01'),
    PARTITION p_max     VALUES LESS THAN (MAXVALUE)
);
```

Cu un {{< glossary term="local-index" >}}index local{{< /glossary >}} pe dată:

```sql
CREATE INDEX idx_vendite_data_local ON fact_vendite_part (data_vendita) LOCAL;
CREATE INDEX idx_vendite_pv_local   ON fact_vendite_part (punto_vendita_id, data_vendita) LOCAL;
```

Fiecare partiție are propriul segment de index. Când optimizer-ul elimină o partiție, elimină și segmentul de index corespunzător.

## 📦 Migrarea: de la monolit la partiții

Migrarea a 800 de milioane de rânduri nu este o operațiune care se face cu un simplu INSERT...SELECT. Este nevoie de o strategie.

Am folosit abordarea {{< glossary term="ctas" >}}CTAS{{< /glossary >}} (Create Table As Select) cu {{< glossary term="nologging" >}}NOLOGGING{{< /glossary >}} și paralelism. Procedura a fost:

1. Crearea tabelei partiționate goale cu structura definitivă
2. Popularea cu un INSERT direct din tabela originală
3. Reconstruirea indexurilor
4. Validarea numărului de rânduri
5. Redenumirea tabelelor (swap)
6. Executarea imediată a unui backup RMAN (NOLOGGING necesită backup)

```sql
ALTER SESSION ENABLE PARALLEL DML;

INSERT /*+ APPEND PARALLEL(8) */ INTO fact_vendite_part
SELECT * FROM fact_vendite;

COMMIT;
```

Cu 8 procese paralele și NOLOGGING, încărcarea a durat 47 de minute pentru 800 de milioane de rânduri. Destul de bine, având în vedere că fiecare rând trebuia distribuit în partiția corectă în funcție de dată.

Apoi faza de validare:

```sql
SELECT 'Original' AS sursa, COUNT(*) AS randuri FROM fact_vendite
UNION ALL
SELECT 'Partitionata', COUNT(*) FROM fact_vendite_part;
```

800.247.331 rânduri de ambele părți. Perfect.

```sql
ALTER TABLE fact_vendite RENAME TO fact_vendite_old;
ALTER TABLE fact_vendite_part RENAME TO fact_vendite;
```

Tabela originală am păstrat-o o săptămână ca plasă de siguranță, apoi am șters-o.

## ⚡ {{< glossary term="partition-pruning" >}}Partition pruning{{< /glossary >}} în acțiune

Cu partiționarea implementată, aceeași query trimestrială avea un plan de execuție complet diferit:

```sql
SELECT pv.regione,
       cat.famiglia,
       SUM(f.importo)    AS fatturato,
       SUM(f.quantita)   AS pezzi_venduti,
       SUM(f.sconto)     AS sconto_totale
FROM   fact_vendite f
JOIN   dim_punto_vendita pv  ON f.punto_vendita_id = pv.punto_vendita_id
JOIN   dim_prodotto cat      ON f.prodotto_id      = cat.prodotto_id
WHERE  f.data_vendita BETWEEN DATE '2025-10-01' AND DATE '2025-12-31'
GROUP BY pv.regione, cat.famiglia
ORDER BY fatturato DESC;
```

```
----------------------------------------------------------------------
| Id | Operation                   | Name         | Pstart | Pstop  |
----------------------------------------------------------------------
|  0 | SELECT STATEMENT            |              |        |        |
|  1 |  SORT GROUP BY              |              |        |        |
|* 2 |   HASH JOIN                 |              |        |        |
|  3 |    TABLE ACCESS FULL        | DIM_PRODOTTO |        |        |
|* 4 |    HASH JOIN                |              |        |        |
|  5 |     TABLE ACCESS FULL       | DIM_PUNTO_V  |        |        |
|  6 |     PARTITION RANGE ITERATOR|              |  34    |  36    |
|* 7 |      TABLE ACCESS FULL      | FACT_VENDITE |  34    |  36    |
----------------------------------------------------------------------
```

`Pstart: 34, Pstop: 36`. Optimizer-ul a citit doar trei partiții din 37 — octombrie, noiembrie și decembrie 2025. În loc de 800 de milioane de rânduri, a scanat 144 de milioane. În loc de 40 GB de I/O, aproximativ 7 GB.

Rezultatul? De la 12 minute la 40 de secunde.

Nu pentru că hardware-ul era mai rapid, nu pentru că rescrisesem query-urile. Doar pentru că baza de date știa acum unde *nu* trebuie să caute.

## 🔄 Exchange partition: încărcarea care nu costă nimic

Într-un data warehouse, datele sosesc cu o cadență regulată — în cazul nostru, un {{< glossary term="etl" >}}ETL{{< /glossary >}} nocturn care încărca vânzările zilei. Problema clasică a partiționării este: cum încarci datele noi în partiția corectă fără să impactezi query-urile?

Răspunsul se numește {{< glossary term="exchange-partition" >}}exchange partition{{< /glossary >}}.

Procesul funcționa astfel:

1. ETL-ul încarcă datele zilei într-o tabelă de staging nepartiționată
2. Se construiesc indexurile pe tabela de staging (aceeași structură ca indexurile locale)
3. Se execută exchange partition: tabela de staging și partiția țintă își schimbă segmentele

```sql
-- 1. Tabelă de staging cu aceeași structură ca o partiție
CREATE TABLE stg_vendite_daily (
    vendita_id       NUMBER(18)    NOT NULL,
    data_vendita     DATE          NOT NULL,
    punto_vendita_id NUMBER(10)    NOT NULL,
    prodotto_id      NUMBER(10)    NOT NULL,
    cliente_id       NUMBER(10),
    quantita         NUMBER(10,2)  NOT NULL,
    importo          NUMBER(12,2)  NOT NULL,
    sconto           NUMBER(8,2)   DEFAULT 0,
    tipo_pagamento   VARCHAR2(20)
);

-- 2. Încărcare ETL în staging (operațiune independentă)
INSERT /*+ APPEND */ INTO stg_vendite_daily
SELECT * FROM source_vendite WHERE data_vendita = TRUNC(SYSDATE - 1);

-- 3. Exchange: swap instantaneu al segmentelor
ALTER TABLE fact_vendite
EXCHANGE PARTITION p_2026_01 WITH TABLE stg_vendite_daily
INCLUDING INDEXES WITHOUT VALIDATION;
```

Exchange partition este o operațiune DDL care modifică doar data dictionary-ul — nu mută niciun byte de date. Durează mai puțin de o secundă, indiferent de volum. Și în timpul exchange-ului, query-urile pe celelalte partiții continuă să funcționeze fără întrerupere.

În cazul nostru, ETL-ul nocturn acumula datele zilei în staging, iar la sfârșitul lunii se făcea exchange-ul cu partiția lunii curente. Pe parcursul lunii, datele zilnice mergeau în partiția `p_max` (catch-all) și apoi erau consolidate cu un exchange lunar.

## 📊 Ciclul de viață al datelor

Cu partiționarea, gestionarea ciclului de viață devine banală. După trei ani, partiția cea mai veche poate fi:

- **comprimată**: `ALTER TABLE fact_vendite MODIFY PARTITION p_2023_01 COMPRESS FOR QUERY HIGH;`
- **mutată pe storage mai lent**: `ALTER TABLE fact_vendite MOVE PARTITION p_2023_01 TABLESPACE ts_archivio;`
- **eliminată direct**: `ALTER TABLE fact_vendite DROP PARTITION p_2023_01;`

Eliminarea unei partiții este instantanee — este o operațiune pe data dictionary, nu șterge rând cu rând. Compară asta cu un `DELETE FROM fact_vendite WHERE data_vendita < DATE '2023-02-01'` pe 48 de milioane de rânduri: minute de procesare, tone de redo log, și o tabelă plină de spațiu recuperabil care necesită un reorganize.

În proiectul de retail, politica era: 3 ani online comprimate, apoi drop. În fiecare prima zi a lunii, un job programat crea noua partiție și, dacă era necesar, elimina pe cea de acum 37 de luni. Complet automat.

## 🎯 Ce nu rezolvă partiționarea

Partiționarea nu este o baghetă magică. Nu înlocuiește indexurile — dacă query-ul nu filtrează pe cheia de partiționare, pruning-ul nu se activează și baza de date citește toate partițiile. Nu îmbunătățește query-urile care deja folosesc un index eficient pe puține rânduri. Și adaugă complexitate în administrare: partiții de creat, monitorizat, comprimat, eliminat.

Dar pentru o fact table într-un data warehouse — unde datele sunt cronologice, query-urile filtrează pe perioadă, și volumele cresc în fiecare zi — partiționarea range pe dată nu este o opțiune. Este o cerință arhitecturală.

Colegul cu raportul de 12 minute nu avea o problemă de hardware sau de query-uri prost scrise. Avea o tabelă care crescuse dincolo de punctul unde lipsa structurii fizice devine un blocaj. Partiționarea a pus lucrurile la locul lor: 40 de secunde, și niciun rând citit inutil.

------------------------------------------------------------------------

## Glosar

**[Range Partitioning](/ro/glossary/range-partitioning/)** — Strategie de partiționare care împarte o tabelă în segmente bazate pe intervale de valori ale unei coloane (de obicei o dată). Fiecare partiție conține rândurile a căror valoare se încadrează în intervalul definit.

**[Exchange Partition](/ro/glossary/exchange-partition/)** — Operațiune DDL Oracle care schimbă instantaneu segmentele de date între o tabelă nepartiționată și o partiție, fără a muta fizic niciun dat. Folosită în data warehouse-uri pentru încărcări masive cu impact zero.

**[Partition Pruning](/ro/glossary/partition-pruning/)** — Mecanism automat al optimizer-ului Oracle care exclude partițiile nerelevante în timpul execuției unui query, citind doar cele care corespund predicatului WHERE.

**[Fact table](/ro/glossary/fact-table/)** — Tabela centrală a star schema-ului care conține măsurile numerice ale afacerii (sume, cantități, contorizări) și cheile externe către tabelele dimensionale.

**[Full Table Scan](/ro/glossary/full-table-scan/)** — Operațiune de citire în care baza de date parcurge toate blocurile unei tabele fără a folosi indexuri. Eficientă pe volume mari când selectivitatea este scăzută, costisitoare când se caută puține rânduri.
