---
title: "Când un LIKE '%valoare%' încetinește totul: un caz real de optimizare PostgreSQL"
description: "Un caz real de performanță în PostgreSQL în care un LIKE '%valoare%' a generat un full scan și a degradat timpii de răspuns. Analiza planului de execuție și o strategie de indexare scalabilă."
date: "2026-01-06T10:00:00+01:00"
lastmod: "2026-02-26T09:34:00+01:00"
draft: false
translationKey: "like_optimization_postgresql"
tags: ["query-tuning", "performance", "indexes", "pg_trgm"]
categories: ["postgresql"]
image: "like-optimization-postgresql.cover.jpg"
---

Cu câteva săptămâni în urmă, un client m-a contactat cu o problemă
foarte comună:

> "Căutarea din consola administrativă este lentă. Uneori durează câteva
> secunde. Am redus deja JOIN-urile, dar problema nu a dispărut."

Mediu: PostgreSQL în cloud managed.\
Tabela principală: `payment_report` (\~6 milioane de rânduri, 3 GB).\
Coloana căutată: `reference_code`.

Query problematic:

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

## 🧠 Prima observație: JOIN-urile nu erau problema

Am comparat:

-   Versiunea AS-IS (3 JOIN-uri pe aceeași tabelă)
-   Versiunea TO-BE (un singur JOIN)

Rezultatul?

Planul de execuție arăta în ambele cazuri:

``` text
Parallel Seq Scan on payment_report
Rows Removed by Filter: ~2,000,000
Buffers: shared read = sute de mii
Execution Time: 14–18 seconds
```

Reducerea JOIN-urilor a avut un impact marginal.

Problema reală era alta.

------------------------------------------------------------------------

## 📌 Vinovatul: `LIKE '%valoare%'` fără un index adecvat

O căutare cu wildcard inițial (`%valoare%`) face inutilizabil un index
B-Tree obișnuit.

PostgreSQL este forțat să execute un scan secvențial al întregii tabele.

În acest caz specific:

-   \~3 GB de date
-   sute de mii de pagini de 8KB citite
-   workload dominat de I/O
-   secunde de latență

Nu este o problemă de "SQL scris prost". Este o problemă de access path.

------------------------------------------------------------------------

## 🔬 Înainte de a crea un index: analiza riscului

Clientul a întrebat pe bună dreptate:

> "Dacă creăm un index trigram (GIN), riscăm să încetinim tranzacțiile
> de plată?"

Aici intervine un concept adesea ignorat: **churn**.

### Ce este churn-ul?

Reprezintă cât de mult se modifică o tabelă după inserarea rândurilor.

Frecvență mare de: - UPDATE - DELETE

→ churn ridicat\
→ cost mai mare de mentenanță al indexului\
→ posibilă degradare a scrierilor

În cazul nostru:

Tabela `payment_report`: - \~12k inserări/zi - 0 update-uri - 0
delete-uri - 0 dead tuples

Profil: **append-only**

Acesta este cel mai bun scenariu posibil pentru introducerea unui index
GIN.

------------------------------------------------------------------------

## 📊 Verificare esențială: sincron sau batch?

Tabela nu conținea timestamp de inserare.

Soluție: analiză indirectă.

Am corelat rândurile din `payment_report` cu timestamp-ul coșului
(`payment_cart.created_at`) și am analizat distribuția orară.

Rezultat:

-   model continuu 24/7
-   vârfuri în timpul zilei
-   scădere noaptea
-   corelație perfectă cu traficul coșurilor

Concluzie: populare near real-time, nu batch nocturn.

------------------------------------------------------------------------

## 🛠️ Soluția

``` sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE INDEX CONCURRENTLY idx_payment_report_reference_trgm
ON reporting.payment_report
USING gin (reference_code gin_trgm_ops);
```

Precauții:

-   Creare într-o fereastră off-peak
-   Utilizarea modului CONCURRENTLY
-   Monitorizarea I/O în timpul build-ului

------------------------------------------------------------------------

## 📈 Rezultatul așteptat

După crearea indexului:

-   Eliminarea Seq Scan
-   Utilizarea GIN index scan
-   Reducere drastică a I/O
-   Timp de execuție de la secunde → milisecunde

------------------------------------------------------------------------

## 🎯 Lecția cheie

Când o query este lentă:

1.  Nu te opri la numărul de JOIN-uri.
2.  Analizează planul de execuție.
3.  Identifică dacă blocajul este CPU sau I/O.
4.  Evaluează churn-ul înainte de a introduce un index GIN.
5.  Măsoară întotdeauna înainte de a decide.

De multe ori problema nu este „optimizarea query-ului".\
Este oferirea indexului corect planner-ului.

------------------------------------------------------------------------

## 💬 De ce împărtășesc acest caz?

Pentru că este un scenariu extrem de comun:

-   Tabele mari
-   Căutări de tip „conține"
-   Teama de a introduce indexuri GIN
-   Frica de degradarea performanței la scriere

Cu date concrete, decizia devine tehnică, nu emoțională.

Optimizarea nu este magie.\
Este măsurare, analiză de planuri și înțelegerea comportamentului real
al sistemului.
