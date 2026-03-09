---
title: "Ierarhii dezechilibrate: când clientul nu are părinte și grupul nu are bunic"
description: "Un client cu o ierarhie pe trei niveluri — Top Group, Group, Client — unde nu toate ramurile sunt complete. Cum am echilibrat o ragged hierarchy cu self-parenting: cine nu are părinte devine propriul părinte."
date: "2026-01-20T10:00:00+01:00"
draft: false
translationKey: "ragged_hierarchies"
tags: ["hierarchies", "dimensional-modeling", "etl", "oracle", "olap", "reporting"]
categories: ["data-warehouse"]
image: "ragged-hierarchies.cover.jpg"
---

Trei niveluri. Top Group, Group, Client. Pare o structură banală — tipul de ierarhie pe care o desenezi pe o tablă în cinci minute și pe care orice instrument de BI ar trebui să o gestioneze fără probleme.

Apoi descoperi că nu toți clienții aparțin unui grup. Și că nu toate grupurile aparțin unui top group. Și că rapoartele de agregare pe care business-ul le solicită — cifra de afaceri per top group, număr de clienți per grup, drill-down de la vârf la frunză — produc rezultate greșite sau incomplete pentru că ierarhia are goluri.

În jargon tehnic se numește **ragged hierarchy**: o ierarhie în care nu toate ramurile ajung la aceeași adâncime. În lumea reală se numește „problema pe care nimeni nu o vede până nu deschide raportul și numerele nu se potrivesc."

---

## Clientul și modelul original

Proiectul era un data warehouse pentru o companie din sectorul energetic — distribuție de gaz și servicii conexe. Sistemul sursă gestiona un registru de clienți cu o structură ierarhică: clienții puteau fi grupați sub o entitate comercială (**Group**), iar grupurile puteau la rândul lor aparține unei entități superioare (**Top Group**).

Modelul din sursă era o singură tabelă cu referințele ierarhice:

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

Iată un eșantion de date:

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

Uitați-vă la date cu atenție. Există patru situații diferite:

- **Client 1001, 1002, 1003**: ierarhie completă — Client → Group → Top Group
- **Client 1004**: are un Group dar Group-ul nu are Top Group
- **Client 1005, 1006**: fără Group, fără Top Group — clienți direcți
- **Client 1007, 1008**: au un Group (Rete Sud) dar Group-ul nu are Top Group

Aceasta este o ragged hierarchy. Trei niveluri pe hârtie, dar în realitate ramurile au adâncimi diferite.

---

## Problema: rapoartele nu se potrivesc

Business-ul cerea un raport simplu: cifra de afaceri agregată per Top Group, cu posibilitate de drill-down per Group și apoi per Client. O cerere rezonabilă — tipul de lucru pe care îl aștepți de la orice DWH.

Interogarea cea mai naturală:

``` sql
SELECT top_group_name,
       group_name,
       COUNT(*)        AS num_clienti,
       SUM(revenue)    AS cifra_afaceri_totala
FROM   stg_clienti
GROUP BY top_group_name, group_name
ORDER BY top_group_name, group_name;
```

Rezultatul:

``` text
TOP_GROUP_NAME      GROUP_NAME        NUM_CLIENTI  CIFRA_AFACERI_TOTALA
------------------  ----------------  -----------  --------------------
Holding Nazionale   Consorzio Nord              2            214000.00
Holding Nazionale   Gruppo Centro               1             67000.00
(null)              Gruppo Centro               1             45000.00
(null)              Rete Sud                    2            104000.00
(null)              (null)                      2             90000.00
```

Cinci rânduri. Și cel puțin trei probleme.

Gruppo Centro apare de două ori: o dată sub „Holding Nazionale" (clientul 1003 care are top group) și o dată sub NULL (clientul 1004 al cărui top group este NULL). Același grup, despicat pe două rânduri, cu totaluri separate. Oricine se uită la acest raport va crede că Gruppo Centro are 67K cifră de afaceri sub holding și 45K undeva în altă parte. În realitate este un singur grup cu 112K total.

Clienții direcți (Gialli Utilities și Blu Energia) ajung într-un rând cu două NULL-uri. Managementul nu știe ce să facă cu un rând fără nume.

Totalul per Top Group este greșit pentru că lipsesc rândurile cu NULL. Dacă aduni doar rândurile cu top group, pierzi 239K din cifra de afaceri — 30% din total.

---

## Abordarea clasică: COALESCE și rugăciuni

Prima reacție, cea pe care o văd în 90% din cazuri, este să adaugi `COALESCE` în interogare:

``` sql
SELECT COALESCE(top_group_name, group_name, client_name) AS top_group_name,
       COALESCE(group_name, client_name)                 AS group_name,
       client_name,
       revenue
FROM   stg_clienti;
```

Funcționează? Într-un fel da — umple golurile. Dar introduce probleme noi.

Clientul „Gialli Utilities" acum apare ca Top Group, Group și Client simultan. Dacă business-ul vrea să numere câte Top Group-uri sunt, numărul este umflat. Dacă vrea să filtreze pentru „adevăratele" top group-uri, nu are cum să le distingă de clienții promovați de COALESCE.

Și acesta este cazul simplu, cu trei niveluri. Am văzut ierarhii pe cinci niveluri gestionate cu lanțuri de COALESCE îmbricate, multiple CASE WHEN, și o logică de raportare atât de încâlcită încât nimeni nu mai îndrăznea să o atingă. Fiecare nouă cerere de business necesita modificări în cascadă în toate interogările.

Problema de fond este că COALESCE este un plasture aplicat la nivelul de prezentare. Nu rezolvă problema structurală: ierarhia este incompletă și modelul dimensional nu știe acest lucru.

---

## Soluția: self-parenting

Principiul este simplu: **cine nu are părinte devine propriul părinte**.

Un Client fără Group? Acel client devine propriul Group. Un Group fără Top Group? Acel grup devine propriul Top Group. În acest mod ierarhia este întotdeauna completă pe trei niveluri, fără goluri, fără NULL.

Nu este un truc. Este o tehnică standard în modelarea dimensională, descrisă de Kimball și folosită în producție de decenii. Ideea este că dimensiunea ierarhică în DWH trebuie să fie **echilibrată**: fiecare înregistrare trebuie să aibă o valoare validă pentru fiecare nivel al ierarhiei. Dacă sursa nu garantează acest lucru, ETL-ul o face.

### Tabela dimensională

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

Observați două lucruri. Primul: nicio coloană nu este nullable. Group și Top Group sunt `NOT NULL`. Al doilea: am adăugat două flag-uri — `is_direct_client` și `is_standalone_group` — care permit distingerea înregistrărilor echilibrate artificial de cele cu o ierarhie naturală. Acest lucru este important: business-ul trebuie să poată filtra „adevăratele" top group-uri de clienții promovați.

### Logica ETL

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
    -- Dacă nu are group, clientul devine propriul group
    COALESCE(group_id, client_id)          AS group_id,
    COALESCE(group_name, client_name)      AS group_name,
    -- Dacă nu are top group, group-ul (sau clientul) devine propriul top group
    COALESCE(top_group_id, group_id, client_id)       AS top_group_id,
    COALESCE(top_group_name, group_name, client_name)  AS top_group_name,
    region,
    CASE WHEN group_id IS NULL THEN 'Y' ELSE 'N' END  AS is_direct_client,
    CASE WHEN group_id IS NOT NULL AND top_group_id IS NULL
         THEN 'Y' ELSE 'N' END                        AS is_standalone_group
FROM stg_clienti;
```

Uitați-vă la cascada de COALESCE din transformare. Logica este:

- `group_id`: dacă clientul are group, folosește-l; altfel folosește clientul însuși
- `top_group_id`: dacă există top group, folosește-l; dacă nu dar există group, folosește group-ul; dacă nu există nici group, folosește clientul

Fiecare nivel „lipsă" este completat de nivelul imediat inferior. Rezultatul este o ierarhie întotdeauna completă.

### Rezultatul după echilibrare

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

Opt rânduri, zero NULL. Fiecare client are un group și un top group. Flag-urile spun adevărul: Gialli și Blu sunt clienți direcți (self-parented la toate nivelurile), Gruppo Centro și Rete Sud sunt grupuri standalone (self-parented la nivelul top group).

---

## Rapoartele după echilibrare

Aceeași interogare de agregare care anterior producea rezultate sparte:

``` sql
SELECT top_group_name,
       group_name,
       COUNT(*)        AS num_clienti,
       SUM(f.revenue)  AS cifra_afaceri_totala
FROM   dim_client_hierarchy d
JOIN   stg_clienti f ON d.client_id = f.client_id
GROUP BY top_group_name, group_name
ORDER BY top_group_name, group_name;
```

``` text
TOP_GROUP_NAME      GROUP_NAME          NUM_CLIENTI  CIFRA_AFACERI_TOTALA
------------------  ------------------  -----------  --------------------
Blu Energia         Blu Energia                   1             52000.00
Gialli Utilities    Gialli Utilities              1             38000.00
Gruppo Centro       Gruppo Centro                 1             45000.00
Holding Nazionale   Consorzio Nord                2            214000.00
Holding Nazionale   Gruppo Centro                 1             67000.00
Rete Sud            Rete Sud                      2            104000.00
```

Niciun NULL. Fiecare rând are un top group și un group identificabile. Totalurile se potrivesc.

Și dacă business-ul vrea doar „adevăratele" top group-uri, excluzând clienții promovați:

``` sql
SELECT top_group_name,
       COUNT(*)        AS num_clienti,
       SUM(f.revenue)  AS cifra_afaceri_totala
FROM   dim_client_hierarchy d
JOIN   stg_clienti f ON d.client_id = f.client_id
WHERE  d.is_direct_client = 'N'
AND    d.is_standalone_group = 'N'
GROUP BY top_group_name
ORDER BY cifra_afaceri_totala DESC;
```

``` text
TOP_GROUP_NAME      NUM_CLIENTI  CIFRA_AFACERI_TOTALA
------------------  -----------  --------------------
Holding Nazionale             3            281000.00
```

Flag-urile fac totul filtrabil. Fără logică condițională în raport, fără CASE WHEN, fără COALESCE. Modelul dimensional conține deja toată informația necesară.

---

## Drill-down-ul complet

Adevăratul test al unei ierarhii echilibrate este drill-down-ul: de la nivelul cel mai înalt la cel mai de jos, fără surprize.

``` sql
-- Nivelul 1: total per Top Group
SELECT top_group_name,
       COUNT(DISTINCT group_id) AS num_grupuri,
       COUNT(*)                 AS num_clienti,
       SUM(f.revenue)           AS cifra_afaceri
FROM   dim_client_hierarchy d
JOIN   stg_clienti f ON d.client_id = f.client_id
GROUP BY top_group_name
ORDER BY cifra_afaceri DESC;
```

``` text
TOP_GROUP_NAME      NUM_GRUPURI  NUM_CLIENTI  CIFRA_AFACERI
------------------  -----------  -----------  -------------
Holding Nazionale             2            3     281000.00
Rete Sud                      1            2     104000.00
Blu Energia                   1            1      52000.00
Gruppo Centro                 1            1      45000.00
Gialli Utilities              1            1      38000.00
```

``` sql
-- Nivelul 2: drill-down în "Holding Nazionale"
SELECT group_name,
       COUNT(*)       AS num_clienti,
       SUM(f.revenue) AS cifra_afaceri
FROM   dim_client_hierarchy d
JOIN   stg_clienti f ON d.client_id = f.client_id
WHERE  d.top_group_name = 'Holding Nazionale'
GROUP BY group_name
ORDER BY cifra_afaceri DESC;
```

``` text
GROUP_NAME        NUM_CLIENTI  CIFRA_AFACERI
----------------  -----------  -------------
Consorzio Nord              2     214000.00
Gruppo Centro               1      67000.00
```

``` sql
-- Nivelul 3: drill-down în "Consorzio Nord"
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

Trei niveluri de drill-down, zero NULL, zero logică condițională. Ierarhia este echilibrată și numerele se potrivesc la fiecare nivel.

---

## De ce COALESCE în rapoarte nu este suficient

Cineva ar putea obiecta: „Dar COALESCE în raport face același lucru, fără să modifici modelul."

Nu. Face ceva asemănător, dar cu trei diferențe fundamentale.

**Primul: COALESCE trebuie repetat peste tot.** Fiecare interogare, fiecare raport, fiecare dashboard, fiecare extracție. Dacă ai douăzeci de rapoarte care folosesc ierarhia, trebuie să îți amintești să aplici COALESCE în toate douăzeci. Și când vine al douăzeci și unulea, trebuie să îți amintești din nou. Self-parenting-ul în modelul dimensional se face o dată în ETL și gata.

**Al doilea: COALESCE nu distinge.** Nu știi dacă „Gialli Utilities" în câmpul top_group este un adevărat top group sau un client promovat. Cu flag-urile din modelul dimensional ai informația pentru a filtra. Fără flag-uri, business-ul este orb.

**Al treilea: performanța.** Un GROUP BY cu COALESCE pe coloane nullable este mai puțin eficient decât un GROUP BY pe coloane NOT NULL. Optimizatorul Oracle gestionează mai bine coloanele cu constrângere NOT NULL — poate elimina verificările de NULL, poate folosi indecșii mai agresiv și poate produce planuri de execuție mai simple. Pe o tabelă dimensională cu milioane de rânduri, diferența se vede.

---

## Când să folosești self-parenting (și când nu)

Self-parenting-ul funcționează bine când:

- Ierarhia are un **număr fix de niveluri** (de obicei 2-5)
- Cazul de utilizare principal este **agregarea și drill-down-ul** în rapoarte
- Modelul este un **data warehouse** sau un cub OLAP
- Nivelurile lipsă sunt excepția, nu regula

Nu funcționează bine când:

- Ierarhia este **recursivă** cu adâncime variabilă (ex. organigrame cu N niveluri)
- Trebuie navigat **graful** relațiilor (ex. rețele sociale, lanțuri de aprovizionare)
- Modelul este **OLTP** și self-parenting-ul ar crea ambiguitate în logicile aplicative
- Nivelurile ierarhiei se schimbă frecvent în timp

Pentru ierarhiile recursive cu adâncime variabilă, abordarea corectă este diferită: tabele de bridge, closure tables sau modele parent-child cu CTE-uri recursive. Sunt instrumente puternice dar rezolvă o problemă diferită.

Self-parenting-ul rezolvă o problemă specifică — ierarhii cu niveluri fixe și ramuri incomplete — și o rezolvă în modul cel mai simplu posibil: echilibrând structura în amonte, în model, în loc de aval, în rapoarte.

---

## Regula care mă ghidează

Am proiectat zeci de dimensiuni ierarhice în douăzeci de ani de data warehousing. Regula pe care o port cu mine este întotdeauna aceeași:

**Dacă raportul are nevoie de logică condițională pentru a gestiona ierarhia, problema este în model, nu în raport.**

Un raport ar trebui să facă GROUP BY și JOIN. Dacă trebuie și să decidă cum să gestioneze nivelurile lipsă, face munca ETL-ului. Și un raport care face munca ETL-ului este un raport care mai devreme sau mai târziu se strică.

Self-parenting-ul nu este elegant. Nu este sofisticat. Este o soluție pe care un informatician proaspăt absolvent ar putea-o găsi urâtă. Dar funcționează, este mentenabil, și transformă o problemă care infestează fiecare raport individual într-o problemă care se rezolvă o dată, într-un singur loc, și nu mai revine.

Uneori cea mai bună soluție este cea mai simplă. Aceasta este una dintre acele dăți.
