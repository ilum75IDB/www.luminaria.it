---
title: "VACUUM și autovacuum: de ce PostgreSQL are nevoie ca cineva să facă curățenie"
description: "O bază de date PostgreSQL de 200 GB cu tabele umflate la triplul dimensiunii reale. Autovacuum era activ, dar prost configurat. Cum diagnostichezi bloat-ul, citești pg_stat_user_tables și faci tuning fără să dezactivezi nimic."
date: "2026-03-24T08:03:00+01:00"
lastmod: "2026-03-24T08:03:00+01:00"
draft: false
translationKey: "vacuum_autovacuum_postgresql"
tags: ["vacuum", "autovacuum", "mvcc", "performance", "bloat"]
categories: ["postgresql"]
image: "vacuum-autovacuum-postgresql.cover.jpg"
---

Acum câțiva ani mi s-a cerut să verific un PostgreSQL în producție care
"se încetinește în fiecare săptămână". Mereu același tipar: luni merge
bine, vineri e dezastru. În weekend cineva repornește serviciul și se
începe de la capăt.

Baza de date de aproximativ 200 GB. Tabelele principale ocupau aproape
triplul spațiului efectiv al datelor. Query-uri care cădeau în
sequential scan acolo unde nu ar fi trebuit. Timpii de răspuns creșteau
zi de zi.

Autovacuum era activ. Nimeni nu-l dezactivase. Dar nici nu-l
configurase.

------------------------------------------------------------------------

## 🧠 MVCC: de ce PostgreSQL generează "gunoi"

Ca să înțelegi problema, trebuie un pas înapoi. PostgreSQL folosește
MVCC — Multi-Version Concurrency Control. De fiecare dată când faci un
UPDATE, baza de date nu suprascrie rândul original. Creează o versiune
nouă și marchează pe cea veche ca "moartă".

La fel pentru DELETE: rândul nu este șters fizic. Este marcat ca
invizibil pentru tranzacțiile noi.

Aceste rânduri moarte se numesc **dead tuples**. Și rămân acolo, în
paginile de date, ocupând spațiu pe disc și încetinind scanările.

Este prețul pe care PostgreSQL îl plătește pentru izolarea
tranzacțională fără lock-uri exclusive pe citiri. Un preț rezonabil —
cu condiția ca cineva să facă curățenie.

------------------------------------------------------------------------

## 🔧 VACUUM: ce face de fapt

Comanda `VACUUM` face un lucru simplu: recuperează spațiul ocupat de
dead tuples și îl face reutilizabil pentru inserări noi.

Nu returnează spațiu sistemului de operare. Nu reorganizează tabela. Nu
compactează nimic. Marchează paginile ca rescriptibile.

``` sql
VACUUM reporting.transactions;
```

Atât e suficient în majoritatea cazurilor. VACUUM este ușor, nu
blochează scrierile și poate rula în paralel cu query-urile normale.

### Dar `VACUUM FULL`?

`VACUUM FULL` e altă poveste. Rescrie fizic întreaga tabelă, eliminând
tot spațiul mort. Returnează spațiu la filesystem.

Dar costul e brutal: **ia un lock exclusiv** pe tabelă pe toată durata
operației. Nimeni nu citește, nimeni nu scrie. Pe tabele mari vorbim de
minute sau ore.

``` sql
VACUUM FULL reporting.transactions;
```

În producție, `VACUUM FULL` trebuie folosit foarte rar. În urgențe. Și
întotdeauna în afara programului.

------------------------------------------------------------------------

## ⚙️ Autovacuum: paznicul tăcut

PostgreSQL are un daemon care rulează VACUUM automat: autovacuum.

Pornește când o tabelă acumulează destule dead tuples. Pragul se
calculează astfel:

```
vacuum threshold = autovacuum_vacuum_threshold
                 + autovacuum_vacuum_scale_factor × n_live_tup
```

Valorile implicite:

- `autovacuum_vacuum_threshold`: **50** dead tuples
- `autovacuum_vacuum_scale_factor`: **0.2** (20%)

Tradus: pe o tabelă cu 10 milioane de rânduri, autovacuum pornește când
dead tuples depășesc **2.000.050**. Două milioane de rânduri moarte
înainte ca cineva să facă curățenie.

Pentru o tabelă cu 500.000 de update-uri pe zi, asta înseamnă că
autovacuum se activează poate la fiecare 4 zile. Între timp bloat-ul
crește, scanările se încetinesc, indecșii se umflă.

De aceea luni totul mergea bine și vineri era dezastru.

------------------------------------------------------------------------

## 📊 Diagnostic: citirea pg_stat_user_tables

Primul lucru de făcut când suspectezi o problemă de vacuum este să
interoghezi `pg_stat_user_tables`:

``` sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
    last_vacuum,
    last_autovacuum,
    autovacuum_count,
    vacuum_count
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

În cazul clientului meu, situația arăta așa:

``` text
relname            | n_live_tup | n_dead_tup | dead_pct | last_autovacuum
-------------------+------------+------------+----------+------------------
transactions       | 12.400.000 |  3.800.000 |   23,5%  | acum 3 zile
order_lines        |  8.200.000 |  2.100.000 |   20,4%  | acum 4 zile
inventory_moves    |  5.600.000 |  1.900.000 |   25,3%  | acum 5 zile
```

Aproape un sfert din rânduri erau moarte. Autovacuum rula, dar mult prea
rar pentru a ține pasul.

------------------------------------------------------------------------

## 🎯 Tuning: adaptarea autovacuum la realitate

Trucul nu e să dezactivezi autovacuum. Niciodată. Trucul e să-l
configurezi pentru tabelele care au nevoie.

PostgreSQL permite setarea parametrilor de autovacuum **per tabelă
individuală**:

``` sql
ALTER TABLE reporting.transactions SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_vacuum_threshold = 1000
);
```

Cu această setare, autovacuum pornește după 1.000 + 1% din rândurile
vii de dead tuples. Pe 12 milioane de rânduri, se activează la ~121.000
dead tuples în loc de 2 milioane.

### cost_delay: nu sugruma vacuum-ul

Alt parametru critic este `autovacuum_vacuum_cost_delay`. Controlează
cât de mult vacuum-ul "se frânează singur" pentru a nu suprasolicita
I/O-ul.

Valoarea implicită e 2 milisecunde. Pe servere moderne cu SSD, e prea
conservator. Reducerea la 0 sau 1 ms permite vacuum-ului să termine mai
repede:

``` sql
ALTER TABLE reporting.transactions SET (
    autovacuum_vacuum_cost_delay = 0
);
```

### max_workers

Valoarea implicită e 3 workeri de autovacuum. Dacă ai zeci de tabele cu
trafic mare, 3 workeri nu sunt suficienți. Evaluează creșterea la 5–6,
monitorizând impactul asupra CPU și I/O:

``` text
-- în postgresql.conf
autovacuum_max_workers = 5
```

------------------------------------------------------------------------

## 📏 Măsurarea bloat-ului

Cum știi cât spațiu irosesc tabelele tale?

Query-ul clasic folosește `pgstattuple`:

``` sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT
    pg_size_pretty(pg_total_relation_size('reporting.transactions')) AS total_size,
    pg_size_pretty(pg_total_relation_size('reporting.transactions')
                   - pg_relation_size('reporting.transactions')) AS index_size,
    *
FROM pgstattuple('reporting.transactions');
```

Câmpurile cheie: `dead_tuple_percent` și `free_space`. Dacă dead_tuple
depășește 20–30%, tabela are o problemă serioasă.

O alternativă mai puțin precisă dar mai ușoară este estimarea bloat
ratio comparând `pg_class.relpages` cu rândurile estimate — există
query-uri consolidate în comunitate pentru asta (clasicul "bloat
estimation query" de la PostgreSQL Experts).

------------------------------------------------------------------------

## 🛠️ Când VACUUM nu e suficient: pg_repack

Dacă bloat-ul a scăpat de sub control — tabele la 50–70% spațiu mort —
VACUUM-ul normal nu recuperează totul. Eliberează dead tuples, dar
spațiul fragmentat rămâne.

`VACUUM FULL` funcționează, dar blochează totul.

Alternativa în producție este **pg_repack**: reconstruiește tabela
online, fără lock-uri exclusive prelungite.

``` bash
pg_repack -d mydb -t reporting.transactions
```

Nu e o soluție de folosit în fiecare săptămână. E cura de șoc pentru
când situația a degenerat deja. Soluția reală e să nu ajungi acolo, cu
un autovacuum bine configurat.

------------------------------------------------------------------------

## 💬 Principiul

Dezactivarea autovacuum este cel mai rău lucru pe care-l poți face unui
PostgreSQL în producție. Am văzut-o făcută "pentru că încetinește
query-urile în timpul zilei". Sigur, pentru că între timp bloat-ul îți
mănâncă baza de date din interior.

Autovacuum cu valorile implicite PostgreSQL este proiectat pentru o bază
de date generică. Nicio bază de date în producție nu e generică. Fiecare
tabelă are propriul tipar de scriere, propriul volum, propriul ritm.

Trei lucruri de reținut:

1. Verifică `pg_stat_user_tables` regulat. Dacă `n_dead_tup` crește
   mai repede decât poate curăța autovacuum, ai o problemă.

2. Configurează `scale_factor` și `threshold` pentru tabelele cu trafic
   mare. Nu există o configurare universală.

3. Nu aștepta ca bloat-ul să ajungă la 50% pentru a interveni. În acel
   punct opțiunile sunt puține și toate dureroase.

Bazele de date nu se întrețin singure. Nici cele care au un daemon care
încearcă.

------------------------------------------------------------------------

## Glosar

**[VACUUM](/ro/glossary/vacuum/)** — Comandă PostgreSQL care recuperează spațiul ocupat de dead tuples, făcându-l reutilizabil pentru inserări noi fără a-l returna sistemului de operare.

**[MVCC](/ro/glossary/mvcc/)** — Multi-Version Concurrency Control — modelul de concurență al PostgreSQL care menține mai multe versiuni ale rândurilor pentru a garanta izolarea tranzacțională fără lock-uri exclusive pe citiri.

**[Dead Tuple](/ro/glossary/dead-tuple/)** — Rând obsolet într-o tabelă PostgreSQL, marcat ca nevizibil după un UPDATE sau DELETE dar încă neșters fizic de pe disc.

**[Autovacuum](/ro/glossary/autovacuum/)** — Daemon PostgreSQL care rulează automat VACUUM și ANALYZE pe tabele când numărul de dead tuples depășește un prag configurabil.

**[Bloat](/ro/glossary/bloat/)** — Spațiu mort acumulat într-o tabelă sau index PostgreSQL din cauza dead tuple-urilor neșterse, care umflă dimensiunea pe disc și degradează performanțele.
