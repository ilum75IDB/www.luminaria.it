---
title: "SCD Tip 2: istoria pe care business-ul nu știa că o vrea"
description: "Un director comercial întreabă câți clienți avea regiunea Nord în iunie trecut. DWH-ul nu poate răspunde pentru că fiecare actualizare suprascrie datele anterioare. Cum am implementat o SCD Tip 2 cu chei surogat și date de valabilitate pentru a reda business-ului memoria istorică."
date: "2025-11-11T10:00:00+01:00"
draft: false
translationKey: "scd_tipo_2"
tags: ["scd", "dimensional-modeling", "etl", "kimball", "data-warehouse"]
categories: ["data-warehouse"]
image: "scd-tipo-2.cover.jpg"
---

Directorul comercial apare la ședința de luni dimineața cu o întrebare simplă: "Câți clienți aveam în regiunea Nord în iunie trecut?"

Răspunsul DWH-ului: liniște.

Nu pentru că sistemul era căzut, sau pentru că lipsea tabela. Datele erau acolo, tehnic vorbind. Dar erau greșite. DWH-ul returna clienții care *astăzi* sunt în regiunea Nord — nu pe cei care erau acolo în iunie. Pentru că în fiecare noapte, procesul de încărcare suprascria datele master ale clienților cu valorile curente, ștergând orice urmă a ceea ce fusese înainte.

Un client care în iunie era în regiunea Nord și în septembrie s-a mutat în regiunea Centru? Pentru DWH, acel client fusese întotdeauna în regiunea Centru. Istoria nu exista.

---

## Proiectul și modelul original

Contextul era un data warehouse în sectorul asigurărilor — gestionarea daunelor și portofoliului de clienți. Sistemul sursă conținea o înregistrare master pentru fiecare client: nume, regiune, agent alocat, clasă de risc, tip de poliță.

Dimensiunea din DWH era modelată astfel:

``` sql
CREATE TABLE dim_client (
    client_id       NUMBER(10)    NOT NULL,
    nume            VARCHAR2(100) NOT NULL,
    regiune         VARCHAR2(50)  NOT NULL,
    agent           VARCHAR2(100),
    clasa_risc      VARCHAR2(20),
    tip_polita      VARCHAR2(50),
    CONSTRAINT pk_dim_client PRIMARY KEY (client_id)
);
```

ETL-ul de noapte era un simplu MERGE: dacă clientul există, actualizează toate câmpurile; dacă nu există, inserează.

``` sql
MERGE INTO dim_client d
USING stg_client s ON (d.client_id = s.client_id)
WHEN MATCHED THEN UPDATE SET
    d.nume       = s.nume,
    d.regiune    = s.regiune,
    d.agent      = s.agent,
    d.clasa_risc = s.clasa_risc,
    d.tip_polita = s.tip_polita
WHEN NOT MATCHED THEN INSERT (
    client_id, nume, regiune, agent, clasa_risc, tip_polita
) VALUES (
    s.client_id, s.nume, s.regiune, s.agent, s.clasa_risc, s.tip_polita
);
```

Simplu, curat, rapid. Și complet greșit pentru un data warehouse.

Aceasta este ceea ce Kimball numește **SCD Tip 1** — Slowly Changing Dimension de Tip 1. Suprascrii valoarea veche cu cea nouă. Fără istorie, fără versionare. Valoarea actuală șterge valoarea anterioară.

Pentru un sistem OLTP este perfect: vrei întotdeauna adresa curentă a clientului, telefonul actualizat, email-ul valid. Dar un data warehouse nu este un sistem tranzacțional. Un data warehouse este o mașină a timpului. Și o mașină a timpului care suprascrie trecutul este inutilă.

---

## Ce se pierde cu Tipul 1

Directorul comercial nu era singurul care punea întrebări la care DWH-ul nu putea răspunde. Iată un eșantion din solicitările acumulate în trei luni:

- *"Câți clienți au trecut de la clasa de risc Înaltă la Scăzută în ultimul an?"* — Imposibil. Clasa anterioară nu mai există.
- *"Agentul Rossi a pierdut clienți față de trimestrul trecut?"* — Imposibil. Dacă un client a fost reatribuit agentului Bianchi, nu există nicio urmă că ar fi aparținut vreodată lui Rossi.
- *"Cifra de afaceri a regiunii Sud a scăzut sau clienții s-au mutat?"* — Imposibil de distins. Dacă un client de 200K s-a mutat din regiunea Sud în regiunea Centru, cifra de afaceri a Sudului scade dar nu pentru că business-ul merge prost — pur și simplu clientul și-a schimbat adresa.

De fiecare dată răspunsul era același: "Sistemul nu păstrează istoria." Ceea ce tradus în limbaj de business înseamnă: "Nu știm."

La un moment dat, CFO-ul a cerut un raport de analiză trimestrială care să compare compoziția portofoliului de clienți între Q1 și Q2. Echipa de BI a încercat să-l construiască. Le-a luat trei zile. Rezultatul nu era fiabil pentru că datele din Q1 nu mai existau — fuseseră suprascrise cu datele din Q2. Raportul compara Q2 cu Q2 deghizat în Q1.

Acela a fost momentul care a declanșat proiectul de restructurare.

---

## SCD Tip 2: principiul

Tipul 2 nu suprascrie. Versionează.

Când un atribut se schimbă, înregistrarea curentă este închisă — i se atribuie o dată de sfârșit de valabilitate — și se inserează o nouă înregistrare cu valorile actualizate și o nouă dată de început de valabilitate. Înregistrarea veche rămâne în baza de date, intactă, cu toate valorile pe care le avea când era curentă.

Pentru ca acest lucru să funcționeze sunt necesare trei elemente suplimentare în tabela dimensională:

1. **O cheie surogat** — un identificator generat de DWH, distinct de cheia naturală a sistemului sursă. Este necesar deoarece același client va avea mai multe înregistrări (câte una pentru fiecare versiune), deci cheia naturală nu mai este unică.
2. **Date de valabilitate** — `valid_from` și `valid_to` — care definesc intervalul temporal în care fiecare versiune a înregistrării era curentă.
3. **Un flag de versiune curentă** — `is_current` — care permite recuperarea rapidă a versiunii active fără a filtra pe date.

### Noua tabelă dimensională

``` sql
CREATE TABLE dim_client (
    client_key      NUMBER(10)    NOT NULL,
    client_id       NUMBER(10)    NOT NULL,
    nume            VARCHAR2(100) NOT NULL,
    regiune         VARCHAR2(50)  NOT NULL,
    agent           VARCHAR2(100),
    clasa_risc      VARCHAR2(20),
    tip_polita      VARCHAR2(50),
    valid_from      DATE          NOT NULL,
    valid_to        DATE          NOT NULL,
    is_current      CHAR(1)       DEFAULT 'Y' NOT NULL,
    CONSTRAINT pk_dim_client PRIMARY KEY (client_key)
);

CREATE INDEX idx_dim_client_natural ON dim_client (client_id, is_current);
CREATE INDEX idx_dim_client_validity ON dim_client (client_id, valid_from, valid_to);

CREATE SEQUENCE seq_dim_client START WITH 1 INCREMENT BY 1;
```

`client_key` este cheia surogat — generată de secvență, niciodată preluată din sistemul sursă. `client_id` este cheia naturală — servește pentru a lega diferitele versiuni ale aceluiași client.

`valid_to` pentru înregistrarea curentă o fixez la `DATE '9999-12-31'` — o convenție standard care simplifică interogările temporale. Când cauți înregistrarea validă la o anumită dată, filtrul `WHERE data_referinta BETWEEN valid_from AND valid_to` funcționează fără cazuri speciale.

---

## Logica ETL

ETL-ul Tipului 2 are două faze: mai întâi închizi înregistrările care s-au schimbat, apoi inserezi noile versiuni. Ordinea contează — dacă inserezi înainte de a închide, există un moment în care coexistă două versiuni "curente" ale aceluiași client.

### Faza 1: identificarea și închiderea înregistrărilor modificate

``` sql
MERGE INTO dim_client d
USING (
    SELECT s.client_id,
           s.nume,
           s.regiune,
           s.agent,
           s.clasa_risc,
           s.tip_polita
    FROM   stg_client s
    JOIN   dim_client d
           ON  s.client_id = d.client_id
           AND d.is_current = 'Y'
    WHERE  (s.regiune    != d.regiune
         OR s.agent      != d.agent
         OR s.clasa_risc != d.clasa_risc
         OR s.tip_polita != d.tip_polita
         OR s.nume       != d.nume)
) changed ON (d.client_id = changed.client_id AND d.is_current = 'Y')
WHEN MATCHED THEN UPDATE SET
    d.valid_to   = TRUNC(SYSDATE) - 1,
    d.is_current = 'N';
```

WHERE-ul compară fiecare atribut urmărit. Dacă măcar unul este diferit, înregistrarea curentă este închisă: `valid_to` se setează pe ziua de ieri și `is_current` devine 'N'.

O notă practică: compararea cu `!=` nu gestionează NULL-urile. Dacă `agent` poate fi NULL, ai nevoie de funcții de comparare NULL-safe. În Oracle folosesc `DECODE`:

``` sql
WHERE DECODE(s.regiune, d.regiune, 0, 1) = 1
   OR DECODE(s.agent, d.agent, 0, 1) = 1
   OR DECODE(s.clasa_risc, d.clasa_risc, 0, 1) = 1
   -- ...
```

`DECODE` tratează două NULL-uri ca egale — exact comportamentul de care ai nevoie.

### Faza 2: inserarea noilor versiuni

``` sql
INSERT INTO dim_client (
    client_key, client_id, nume, regiune, agent,
    clasa_risc, tip_polita, valid_from, valid_to, is_current
)
SELECT seq_dim_client.NEXTVAL,
       s.client_id,
       s.nume,
       s.regiune,
       s.agent,
       s.clasa_risc,
       s.tip_polita,
       TRUNC(SYSDATE),
       DATE '9999-12-31',
       'Y'
FROM   stg_client s
WHERE  NOT EXISTS (
    SELECT 1
    FROM   dim_client d
    WHERE  d.client_id = s.client_id
    AND    d.is_current = 'Y'
);
```

Acest INSERT acoperă două cazuri: clienți complet noi (care nu există în dim_client) și clienți a căror versiune curentă tocmai a fost închisă în Faza 1 (care prin urmare nu mai au o înregistrare cu `is_current = 'Y'`).

`valid_from` este data de astăzi. `valid_to` este "sfârșitul timpului" — `9999-12-31`. `client_key` este generată de secvență.

---

## Datele: înainte și după

Să vedem un exemplu concret. Clientul 2001 — "Alfa Asigurări Srl" — este în regiunea Nord, alocat agentului Rossi, clasă de risc Medie.

În iulie clientul este reatribuit agentului Bianchi. În octombrie clasa de risc se schimbă de la Medie la Înaltă.

**Cu Tipul 1** (modelul anterior), în octombrie dim_client conține o singură linie:

``` text
CLIENT_ID  NUME                  REGIUNE  AGENT    CLASA_RISC
---------  --------------------  -------  -------  ----------
2001       Alfa Asigurări Srl    Nord     Bianchi  Inalta
```

Nicio urmă de Rossi. Nicio urmă de clasa Medie. Pentru DWH, acest client a aparținut întotdeauna agentului Bianchi cu clasă Înaltă.

**Cu Tipul 2**, în octombrie dim_client conține trei linii:

``` text
KEY   CLIENT_ID  NUME                  REGIUNE  AGENT    CLASA   VALID_FROM  VALID_TO    CURRENT
----  ---------  --------------------  -------  -------  ------  ----------  ----------  -------
1001  2001       Alfa Asigurări Srl    Nord     Rossi    Medie   2025-01-15  2025-07-09  N
1002  2001       Alfa Asigurări Srl    Nord     Bianchi  Medie   2025-07-10  2025-10-04  N
1003  2001       Alfa Asigurări Srl    Nord     Bianchi  Inalta  2025-10-05  9999-12-31  Y
```

Trei versiuni ale aceluiași client. Fiecare versiune spune o parte din poveste: cine era agentul, care era clasa de risc și în ce perioadă. Datele nu se suprapun. Flag-ul `is_current` identifică versiunea activă.

---

## Interogările temporale

Acum directorul comercial poate primi răspunsul său.

### Câți clienți în regiunea Nord în iunie?

``` sql
SELECT COUNT(DISTINCT client_id) AS clienti_nord_iunie
FROM   dim_client
WHERE  regiune = 'Nord'
AND    DATE '2025-06-15' BETWEEN valid_from AND valid_to;
```

Interogarea este directă: ia toate înregistrările care erau valide pe 15 iunie 2025 și filtrează pe regiune. Fără CASE WHEN, fără logică condițională, fără aproximări.

### Clienți care au schimbat clasa de risc în ultimul an

``` sql
SELECT c1.client_id,
       c1.nume,
       c1.clasa_risc    AS clasa_anterioara,
       c2.clasa_risc    AS clasa_actuala,
       c1.valid_to + 1  AS data_schimbare
FROM   dim_client c1
JOIN   dim_client c2
       ON  c1.client_id = c2.client_id
       AND c1.valid_to + 1 = c2.valid_from
WHERE  c1.clasa_risc != c2.clasa_risc
AND    c1.valid_to >= ADD_MONTHS(TRUNC(SYSDATE), -12)
ORDER BY data_schimbare DESC;
```

Două versiuni consecutive ale aceluiași client, unite prin data de tranziție. Dacă clasa de risc diferă între cele două versiuni, clientul a schimbat clasa. Data schimbării este ziua de după închiderea versiunii anterioare.

### Comparație portofoliu Q1 vs Q2

``` sql
SELECT regiune,
       COUNT(DISTINCT CASE
           WHEN DATE '2025-03-31' BETWEEN valid_from AND valid_to
           THEN client_id END) AS clienti_q1,
       COUNT(DISTINCT CASE
           WHEN DATE '2025-06-30' BETWEEN valid_from AND valid_to
           THEN client_id END) AS clienti_q2
FROM   dim_client
WHERE  DATE '2025-03-31' BETWEEN valid_from AND valid_to
   OR  DATE '2025-06-30' BETWEEN valid_from AND valid_to
GROUP BY regiune
ORDER BY regiune;
```

Un singur scan al tabelei, două contorizări distincte filtrate pe dată. CFO-ul are raportul trimestrial — cel real, nu cel care compara Q2 cu el însuși.

---

## Tabela de fapte și cheile surogat

Un punct adesea subestimat: tabela de fapte trebuie să folosească **cheia surogat**, nu cheia naturală.

``` sql
CREATE TABLE fact_dauna (
    dauna_key       NUMBER(10)    NOT NULL,
    client_key      NUMBER(10)    NOT NULL,  -- FK la versiunea specifică
    data_key        NUMBER(8)     NOT NULL,
    suma            NUMBER(15,2),
    tip_dauna       VARCHAR2(50),
    CONSTRAINT pk_fact_dauna PRIMARY KEY (dauna_key),
    CONSTRAINT fk_fact_client FOREIGN KEY (client_key)
        REFERENCES dim_client (client_key)
);
```

`client_key` din tabela de fapte indică spre *versiunea* clientului care era curentă în momentul daunei. Dacă o daună survine în mai, când clientul aparținea încă agentului Rossi, faptul indică spre versiunea cu agentul Rossi. Dacă altă daună survine în septembrie, cu clientul deja sub agentul Bianchi, faptul indică spre versiunea cu agentul Bianchi.

Rezultatul este că fiecare fapt este asociat cu contextul dimensional corect pentru momentul în care s-a produs. Interoghezi daunele din mai și vezi agentul Rossi. Interoghezi pe cele din septembrie și vezi agentul Bianchi. Fără nicio logică temporală în interogare — JOIN-ul direct între tabela de fapte și dimensiune returnează contextul corect.

``` sql
-- Daune pe agent, cu contextul corect din momentul daunei
SELECT d.agent,
       COUNT(*)     AS num_daune,
       SUM(f.suma)  AS suma_totala
FROM   fact_dauna f
JOIN   dim_client d ON f.client_key = d.client_key
GROUP BY d.agent
ORDER BY suma_totala DESC;
```

Fără clauză temporală. JOIN-ul pe cheia surogat face toată munca.

---

## Dimensiunile Tipului 2

Costul Tipului 2 este creșterea tabelei dimensionale. Cu Tipul 1, fiecare client este o linie. Cu Tipul 2, fiecare client poate avea N linii — câte una pentru fiecare schimbare de atribut urmărit.

În proiectul de asigurări numerele arătau astfel:

| Metrică | Valoare |
|---------|---------|
| Clienți activi | ~120.000 |
| Atribute urmărite | 4 (regiune, agent, clasă risc, tip poliță) |
| Rata medie de schimbare | ~8% din clienți/an |
| Linii dim_client după 1 an | ~140.000 |
| Linii dim_client după 3 ani | ~180.000 |
| Linii dim_client după 5 ani | ~220.000 |

De la 120K la 220K în cinci ani. O creștere de 83% — care pare mult în procente dar este neglijabilă în termeni absoluți. 220K linii sunt nimic pentru Oracle. Interogarea cu index pe cheia surogat rămâne în ordinul milisecundelor.

Problema apare când ai milioane de clienți cu rate mari de schimbare. În acel caz monitorizezi creșterea, consideri partiționarea dimensiunii, și mai ales alegi cu grijă *care* atribute să le urmărești. Nu toate atributele merită Tip 2. Telefonul clientului? Tip 1, suprascriere. Regiunea comercială? Tip 2, pentru că impactează analiza cifrei de afaceri.

Alegerea atributelor de urmărit cu Tip 2 este o decizie de business, nu tehnică. Întreabă business-ul: "Dacă acest câmp se schimbă, aveți nevoie să știți care era valoarea anterioară?" Dacă răspunsul este da, este Tip 2. Dacă este nu, este Tip 1.

---

## Când nu e nevoie de Tipul 2

Nu toate dimensiunile au nevoie de istorie. Am văzut proiecte în care fiecare dimensiune era Tip 2 "pentru siguranță" — rezultatul era un model inutil de complex, ETL-uri lente, și nimeni care să fi interogat vreodată istoria dimensiunii "tip_plată" sau "canal_vânzare".

Tipul 2 are un cost: complexitatea ETL-ului, creșterea tabelei, necesitatea de a gestiona cheile surogat în tabela de fapte. Este un cost care merită plătit când business-ul are nevoie de istorie. Dacă nu are, Tipul 1 este alegerea corectă.

Sunt și cazuri în care Tipul 2 nu este suficient. Dacă trebuie să știi nu doar *ce* s-a schimbat ci și *cine* a făcut schimbarea și *de ce*, atunci ai nevoie de un audit trail — o tabelă separată cu un log complet al modificărilor. Tipul 2 urmărește versiunile, nu cauzele.

Iar pentru dimensiuni cu schimbări foarte frecvente — prețuri care se schimbă zilnic, scoruri care se actualizează orar — Tipul 2 poate genera o creștere nesustenabilă. În acele cazuri se evaluează **Tipul 6** (o combinație de Tipurile 1, 2 și 3) sau abordări de mini-dimensiune.

Dar pentru cazul cel mai frecvent — date master de clienți, produse, angajați, puncte de vânzare — Tipul 2 este instrumentul potrivit. Suficient de simplu pentru a fi implementat fără framework-uri exotice, suficient de puternic pentru a reda business-ului dimensiunea care-i lipsea: timpul.

---

## Ce am învățat

Directorul comercial nu știa că are nevoie de istorie până când a avut nevoie de ea. Și când a avut nevoie, DWH-ul nu o avea.

Acesta este punctul. Nu implementezi Tipul 2 pentru că "e best practice" sau pentru că "Kimball spune așa în capitolul 5". Îl implementezi pentru că un data warehouse fără istorie este o bază de date operațională cu o star schema lipită deasupra. Funcționează pentru rapoartele lunii curente, dar nu răspunde la întrebarea pe care mai devreme sau mai târziu cineva o va pune: "Cum era înainte?"

Întrebarea vine întotdeauna. Singura întrebare este dacă DWH-ul tău este pregătit să răspundă.
