---
title: "Granularitate greșită: când fact table nu răspunde la întrebările potrivite"
description: "O fact table construită pe totalul lunar al facturării părea perfectă. Apoi business-ul a cerut detalii pe produs, pe linie, pe client. Și data warehouse-ul a amuțit."
date: "2025-10-21T10:00:00+01:00"
draft: false
translationKey: "fatto_grana_sbagliata"
tags: ["data-warehouse", "fact-table", "granularity", "grain", "dimensional-modeling", "kimball"]
categories: ["Data Warehouse"]
image: "fatto-grana-sbagliata.cover.jpg"
---

Ședința începuse bine. Directorul comercial al unei companii de distribuție industrială — vreo șaizeci de milioane cifră de afaceri, trei mii de clienți activi, catalog cu douăsprezece mii de referințe — deschisese prezentarea noului data warehouse cu un zâmbet. Cifrele se potriveau, dashboard-urile arătau bine, totalurile lunare pe agent și pe zonă băteau cu contabilitatea.

Apoi cineva a pus întrebarea greșită. Sau mai bine zis, pe cea corectă.

*"Pot să văd cât a cumpărat clientul Bianchi în luna martie, linie cu linie, produs cu produs?"*

Liniște.

Responsabilul BI s-a uitat la mine. Eu m-am uitat la ecran. Ecranul arăta o {{< glossary term="fact-table" >}}fact table{{< /glossary >}} cu un rând per client per lună: sumă totală facturată, cantitate totală, număr de facturi. Niciun detaliu. Nicio linie de factură. Niciun produs.

Acea fact table răspundea la o singură întrebare: *cât a facturat fiecare client într-o lună dată?* Tot restul — pe produs, pe familie de produse, pe factură individuală — era în afara razei.

## 🔍 Grain-ul: decizia care determină totul

În {{< glossary term="star-schema" >}}modelarea dimensională{{< /glossary >}}, **{{< glossary term="grain" >}}grain-ul{{< /glossary >}}** (granularitatea) fact table-ului este prima decizie pe care o iei. Nu a doua, nu una dintre multe: prima. Kimball repetă asta în fiecare capitol, și are dreptate.

Grain-ul răspunde la întrebarea: *ce reprezintă un singur rând din fact table?*

În proiectul pe care l-am descris, cel care proiectase modelul alesese un grain lunar-client: un rând = un client într-o lună. Motivele păreau rezonabile: sistemul sursă exporta un sumar lunar, încărcarea era rapidă, tabelele erau mici, interogările simple.

Dar grain-ul determină întrebările la care data warehouse-ul poate răspunde. Dacă granularitatea este sumarul lunar per client, nu poți coborî sub acel nivel. Nu poți face {{< glossary term="drill-down" >}}drill-down{{< /glossary >}} pe produs. Nu poți ști dacă clientul Bianchi a cumpărat de zece ori același articol sau zece articole diferite. Nu poți compara marjele pe familie de produse.

Ai un total. Punct.

## 📊 Cifrele problemei

Fact table-ul original avea această structură:

```sql
CREATE TABLE fact_facturare_lunara (
    sk_client         INT NOT NULL,
    sk_timp           INT NOT NULL,  -- luna (YYYYMM)
    sk_agent          INT NOT NULL,
    sk_zona           INT NOT NULL,
    suma_totala       DECIMAL(15,2),
    cantitate_totala  INT,
    nr_facturi        INT,
    nr_linii          INT,
    FOREIGN KEY (sk_client) REFERENCES dim_client(sk_client),
    FOREIGN KEY (sk_timp)   REFERENCES dim_timp(sk_timp)
);
```

Rânduri pe an: aproximativ 180.000 (3.000 clienți × 12 luni × puțină variație). Mic, rapid, ușor de încărcat. {{< glossary term="etl" >}}ETL-ul{{< /glossary >}} rula în mai puțin de cinci minute.

Problema? {{< glossary term="additive-measure" >}}Măsurile aditive{{< /glossary >}} erau deja agregate. `suma_totala` era suma tuturor liniilor de factură ale lunii. Imposibil de reconstituit compoziția. Ca și cum ai avea totalul unui bon fără să știi ce ai cumpărat.

## 🏗️ Restructurarea: coborârea la linia de factură

Soluția era una singură: schimbarea grain-ului. Aducerea fact table-ului la cel mai jos nivel disponibil în sistemul sursă — linia individuală de factură.

```sql
CREATE TABLE fact_facturare_linie (
    sk_linie_factura  INT PRIMARY KEY,
    sk_factura        INT NOT NULL,
    sk_client         INT NOT NULL,
    sk_produs         INT NOT NULL,
    sk_timp           INT NOT NULL,  -- zi (YYYYMMDD)
    sk_agent          INT NOT NULL,
    sk_zona           INT NOT NULL,
    sk_familie        INT NOT NULL,
    cantitate         INT,
    pret_unitar       DECIMAL(12,4),
    suma_linie        DECIMAL(15,2),
    discount_pct      DECIMAL(5,2),
    suma_neta         DECIMAL(15,2),
    cost_produs       DECIMAL(15,2),
    marja             DECIMAL(15,2),
    FOREIGN KEY (sk_client) REFERENCES dim_client(sk_client),
    FOREIGN KEY (sk_produs) REFERENCES dim_produs(sk_produs),
    FOREIGN KEY (sk_timp)   REFERENCES dim_timp(sk_timp)
);
```

Rânduri pe an: aproximativ 2,4 milioane (3.000 clienți × ~800 linii/an în medie). Cu un ordin de mărime mai mult. Dar fiecare rând aducea cu sine detaliul complet: ce produs, ce factură, ce preț, ce discount, ce marjă.

## ⚡ Impactul asupra ETL-ului

Schimbarea grain-ului a avut un efect în cascadă asupra ETL-ului pe care nimeni nu-l anticipase — sau mai bine zis, pe care cel care alesese grain-ul agregat evitase să-l înfrunte.

**Dimensiuni noi necesare:**

| Dimensiune         | Cardinalitate | Note                                 |
|---------------------|-------------|--------------------------------------|
| `dim_produs`        | ~12.000     | Nu exista înainte: nu era necesară   |
| `dim_familie`       | ~180        | Ierarhie de produse pe 3 niveluri    |
| `dim_factura`       | ~45.000/an  | Antet factură cu date master         |

**Noua fereastră de încărcare:**

| Fază                | Înainte | După      |
|---------------------|---------|-----------|
| Extracție           | 40 sec  | 3 min     |
| Transformare        | 1 min   | 8 min     |
| Încărcare fact      | 30 sec  | 4 min     |
| **Total**           | **~2 min** | **~15 min** |

Cincisprezece minute versus două. Un preț acceptabil pentru un data warehouse care acum răspundea la întrebări reale.

## 🔬 Interogările care înainte erau imposibile

Cu noul grain, interogările pe care business-ul le dorea deveneau banale:

**Detalierea achizițiilor unui client pe produs:**

```sql
SELECT
    c.denumire_client,
    p.cod_produs,
    p.descriere,
    SUM(f.cantitate)     AS bucati,
    SUM(f.suma_neta)     AS facturat_net,
    SUM(f.marja)         AS marja_totala
FROM fact_facturare_linie f
JOIN dim_client c ON f.sk_client = c.sk_client
JOIN dim_produs p ON f.sk_produs = p.sk_produs
JOIN dim_timp   t ON f.sk_timp   = t.sk_timp
WHERE c.denumire_client = 'Bianchi Srl'
  AND t.an = 2024
  AND t.luna = 3
GROUP BY c.denumire_client, p.cod_produs, p.descriere
ORDER BY facturat_net DESC;
```

**Top 10 produse după marjă într-un trimestru:**

```sql
SELECT
    p.cod_produs,
    p.descriere,
    fm.desc_familie,
    SUM(f.suma_neta)  AS facturat,
    SUM(f.marja)      AS marja,
    ROUND(SUM(f.marja) / NULLIF(SUM(f.suma_neta), 0) * 100, 1) AS marja_pct
FROM fact_facturare_linie f
JOIN dim_produs  p  ON f.sk_produs  = p.sk_produs
JOIN dim_familie fm ON f.sk_familie = fm.sk_familie
JOIN dim_timp    t  ON f.sk_timp    = t.sk_timp
WHERE t.an = 2024
  AND t.trimestru = 1
GROUP BY p.cod_produs, p.descriere, fm.desc_familie
ORDER BY marja DESC
LIMIT 10;
```

**Comparație între agenți: facturare medie pe linie:**

```sql
SELECT
    a.nume_agent,
    COUNT(*)                      AS nr_linii,
    SUM(f.suma_neta)              AS facturat_total,
    ROUND(AVG(f.suma_neta), 2)    AS media_per_linie
FROM fact_facturare_linie f
JOIN dim_agent a ON f.sk_agent = a.sk_agent
JOIN dim_timp  t ON f.sk_timp  = t.sk_timp
WHERE t.an = 2024
GROUP BY a.nume_agent
ORDER BY facturat_total DESC;
```

Niciuna dintre aceste interogări nu era posibilă cu grain-ul lunar-client. Niciuna. Nu era o problemă de tuning sau de indexare — era o problemă structurală, scrisă în ADN-ul modelului.

## 📋 Regula Kimball pe care o ignoraserăm

Ralph Kimball o spune clar: *"modelează întotdeauna la cel mai fin nivel de detaliu disponibil în sistemul sursă."*

Nu e o sugestie. Nu e o opțiune între mai multe. E principiul fondator al modelării dimensionale. Și motivul e simplu: poți oricând agrega de la detaliu la total, dar nu poți niciodată dezagrega un total în detaliul său.

Agregarea este o operație ireversibilă. Ca amestecul culorilor: din roșu și galben obții portocaliu, dar din portocaliu nu te mai întorci niciodată la culorile originale.

În proiectul nostru, alegerea grain-ului agregat fusese dictată de lene în proiectare, nu de o constrângere tehnică. Sistemul sursă avea detaliul pe linie de factură — pur și simplu nimeni nu voise să înfrunte complexitatea modelării, gestionarea dimensiunilor suplimentare, extinderea ferestrei ETL.

Rezultatul? Un data warehouse care a trebuit reconstruit de la zero la șase luni după go-live.

## 🎯 Când grain-ul agregat are sens

Granularitatea fină nu e întotdeauna singura răspuns. Există cazuri legitime pentru fact table-uri agregate:

- **Tabele de agregare** (aggregate fact table) alături de tabela de detaliu, pentru a accelera interogările cele mai frecvente
- **Snapshot-uri periodice** unde business-ul gândește efectiv pe perioade (sold lunar al unui cont, stoc la sfârșit de săptămână)
- **Restricții de sursă** când sistemul upstream nu expune detaliul și nu există modalitate de a-l obține

Dar regula este: pornește de la detaliu, apoi agregă. Niciodată invers. Aggregate fact table-urile sunt o optimizare, nu un substitut pentru granularitatea fină.

În cazul nostru, după restructurare, am creat și o vedere materializată cu sumarul lunar per client — aceeași structură ca înainte — pentru dashboard-urile executive care nu aveau nevoie de detaliu. Ce-i mai bun din ambele lumi, fără a sacrifica nimic.

## Ce am învățat

Acel proiect m-a învățat ceva ce duc cu mine în fiecare angajament ulterior: prima jumătate de oră de proiectare a unui data warehouse, aceea în care se decide grain-ul, valorează mai mult decât toate optimizările care vor urma. Un ETL perfect, indexuri calibrate, hardware puternic — nimic din toate acestea nu compensează un grain greșit.

Dacă fact table-ul tău nu răspunde la întrebările business-ului, nu e vina interogărilor. E vina modelului. Și modelul se decide la grain.

------------------------------------------------------------------------

## Glosar

**[Grain](/ro/glossary/grain/)** — Nivelul de detaliu (granularitatea) al unei fact table în data warehouse. Determină ce reprezintă fiecare rând și la ce întrebări poate răspunde modelul. Este prima decizie în proiectarea dimensională.

**[Fact table](/ro/glossary/fact-table/)** — Tabela centrală a star schema-ului care conține măsurile numerice (sume, cantități, marje) și cheile externe către dimensiuni. Granularitatea sa determină nivelul de analiză posibil.

**[Additive Measure](/ro/glossary/additive-measure/)** — Măsură numerică ce poate fi sumată de-a lungul tuturor dimensiunilor (ex. sumă, cantitate). Odată agregată la nivel superior, detaliul original este pierdut ireversibil.

**[Drill-down](/ro/glossary/drill-down/)** — Navigare în rapoarte de la nivelul agregat la detaliu, de-a lungul unei ierarhii. Posibilă doar dacă fact table-ul conține date la un nivel de granularitate suficient.

**[Star Schema](/ro/glossary/star-schema/)** — Model de date cu o fact table centrală și tabele dimensionale legate. Cea mai utilizată structură în data warehouse pentru simplitatea interogărilor analitice.

**[ETL](/ro/glossary/etl/)** — Extract, Transform, Load: procesul de extracție, transformare și încărcare a datelor în data warehouse. O schimbare de grain impactează direct durata și complexitatea ETL-ului.
