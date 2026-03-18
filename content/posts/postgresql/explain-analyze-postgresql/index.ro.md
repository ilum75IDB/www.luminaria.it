---
title: "EXPLAIN ANALYZE nu e suficient: cum sa citesti cu adevarat un plan de executie PostgreSQL"
description: "Un caz real in care optimizatorul a ales un nested loop pe 2 milioane de randuri pentru ca statisticile erau vechi. Cum sa citesti un plan de executie, sa gasesti semnalele de alarma si sa intervii."
date: "2025-10-28T10:00:00+01:00"
draft: false
translationKey: "explain_analyze_postgresql"
tags: ["explain", "query-tuning", "execution-plan", "optimizer", "performance"]
categories: ["postgresql"]
image: "explain-analyze-postgresql.cover.jpg"
---

Zilele trecute un coleg imi trimite o captura de ecran pe Teams. Un query care ruleaza pe o tabela de 2 milioane de randuri, 45 de secunde timp de executie. Imi scrie:

> "Am facut EXPLAIN ANALYZE, dar nu inteleg ce e in neregula. Planul pare corect."

Spoiler: planul nu era deloc corect. Optimizatorul alesese un nested loop join unde era nevoie de un hash join, iar motivul era banal — statistici neactualizate. Dar ca sa ajung acolo a trebuit sa citesc planul rand cu rand, si atunci mi-am dat seama ca majoritatea DBA-ilor pe care ii cunosc folosesc EXPLAIN ANALYZE ca pe un oracol binar: daca timpul e mare, query-ul e lent. Sfarsitul analizei.

Nu. EXPLAIN ANALYZE e un instrument de diagnostic, nu un verdict. Trebuie sa stii sa-l citesti.

------------------------------------------------------------------------

## 🔧 EXPLAIN, EXPLAIN ANALYZE, EXPLAIN (ANALYZE, BUFFERS): trei lucruri diferite

Sa incepem cu bazele, pentru ca confuzia e mai raspandita decat ai crede.

**EXPLAIN** singur arata planul *estimat*. Optimizatorul decide ce ar face, dar nu executa nimic. Util pentru a intelege strategia, inutil pentru a intelege realitatea.

``` sql
EXPLAIN
SELECT * FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.created_at > '2025-01-01';
```

**EXPLAIN ANALYZE** executa query-ul si adauga timpii reali. Acum vezi cat a durat fiecare nod, cate randuri a returnat efectiv. Dar lipseste o piesa.

``` sql
EXPLAIN ANALYZE
SELECT * FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.created_at > '2025-01-01';
```

**EXPLAIN (ANALYZE, BUFFERS)** e ceea ce folosesc intotdeauna. Adauga informatii despre cate pagini de disc au fost citite, cate erau in cache (shared hit) si cate au trebuit incarcate de pe disc (shared read). Fara BUFFERS conduci noaptea fara faruri.

``` sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.created_at > '2025-01-01';
```

Regula personala: daca cineva imi trimite un EXPLAIN fara BUFFERS, i-l trimit inapoi.

------------------------------------------------------------------------

## 📖 Anatomia unui nod: ce sa citesti si in ce ordine

Un plan de executie e un arbore. Fiecare nod arata asa:

``` text
->  Hash Join  (cost=1234.56..5678.90 rows=50000 width=120)
      (actual time=12.345..89.012 rows=48750 loops=1)
      Buffers: shared hit=1200 read=3400
```

Iata ce sa urmaresti:

**cost** — sunt doua numere separate de `..`. Primul e costul de startup (cat inainte de a returna primul rand), al doilea e costul total estimat. Sunt unitati arbitrare ale optimizatorului, nu milisecunde. Servesc pentru a compara planuri alternative, nu pentru a masura performanta absoluta.

**rows** — randurile estimate de optimizator. Compara-le cu `actual rows`. Daca e un ordin de marime diferenta, ai gasit problema.

**actual time** — timp real in milisecunde. Din nou doua valori: startup si total. Atentie la campul `loops`: daca loops=10, timpul total trebuie inmultit cu 10.

**Buffers** — `shared hit` sunt paginile gasite in memorie, `shared read` cele citite de pe disc. Daca `read` domina, working set-ul tau nu incape in RAM.

------------------------------------------------------------------------

## 🚨 Semnalul de alarma numarul unu: randuri estimate vs randuri reale

Ma intorc la cazul colegului meu. Planul arata:

``` text
->  Nested Loop  (cost=0.87..45678.12 rows=150 width=200)
      (actual time=0.034..44890.123 rows=1950000 loops=1)
```

Optimizatorul estima 150 de randuri. In realitate au venit aproape 2 milioane.

Cand estimarea e gresita cu 4 ordine de marime, planul e inevitabil gresit. Optimizatorul a ales un nested loop pentru ca a crezut ca itereaza peste 150 de randuri. Un nested loop pe 150 de randuri e fulgerator de rapid. Pe 2 milioane e un dezastru.

Un hash join sau merge join ar fi fost alegerea corecta. Dar optimizatorul nu putea sti asta cu statisticile pe care le avea.

Regula practica: daca raportul dintre randurile estimate si cele reale depaseste 10x, ai o problema de statistici. Peste 100x, planul e aproape sigur gresit.

------------------------------------------------------------------------

## 🔍 De ce mint statisticile

PostgreSQL mentine statistici despre tabele in `pg_statistic` (citibile prin `pg_stats`). Aceste statistici includ:

- distributia valorilor (most common values)
- histograma valorilor
- numarul de valori distincte
- procentul de NULL

Optimizatorul foloseste aceste informatii pentru a estima selectivitatea fiecarei conditii WHERE si cardinalitatea fiecarui join.

Problema? Statisticile se actualizeaza cu `ANALYZE` — care poate fi manual sau gestionat de autovacuum. Dar autovacuum-ul lanseaza ANALYZE doar cand numarul de randuri modificate depaseste un prag:

``` text
threshold = autovacuum_analyze_threshold + autovacuum_analyze_scale_factor × n_live_tuples
```

Implicit: 50 de randuri + 10% din randurile vii. Pe o tabela de 2 milioane de randuri, sunt necesare 200.000 de modificari inainte ca ANALYZE-ul automat sa se declanseze.

In cazul colegului meu, tabela `orders` crescuse de la 500.000 la 2 milioane de randuri in trei saptamani — un import masiv dintr-un sistem legacy. Autovacuum-ul nu actualizase statisticile pentru ca 10% din 500.000 (dimensiunea cunoscuta) era 50.000, iar randurile fusesera inserate in loturi care individual nu depasisera pragul.

Rezultat: optimizatorul ragiona inca ca si cum tabela ar fi avut 500.000 de randuri cu vechea distributie a valorilor.

------------------------------------------------------------------------

## 🛠️ Actualizarea statisticilor: primul lucru de facut

Solutia imediata era evidenta:

``` sql
ANALYZE orders;
```

Dupa ANALYZE, am relansat query-ul cu EXPLAIN (ANALYZE, BUFFERS):

``` text
->  Hash Join  (cost=8500.00..32000.00 rows=1940000 width=200)
      (actual time=120.000..2800.000 rows=1950000 loops=1)
      Buffers: shared hit=28000 read=4500
```

De la 45 de secunde la sub 3 secunde. Optimizatorul alesese un hash join, estimarea randurilor era corecta, si planul era complet diferit.

Dar nu m-am oprit aici. Daca problema a aparut o data, va aparea din nou.

------------------------------------------------------------------------

## 📊 default_statistics_target: cand 100 nu e suficient

PostgreSQL colecteaza 100 de valori de esantion pe coloana implicit. Pentru tabele mici sau cu distributie uniforma, e suficient. Pentru tabele mari cu distributie neuniforma, 100 de esantioane pot da o reprezentare distorsionata.

In cazul tabelei `orders`, coloana `customer_id` avea o distributie foarte asimetrica: 5% din clienti generau 60% din comenzi. Cu 100 de esantioane, optimizatorul nu capta aceasta asimetrie.

Solutia:

``` sql
ALTER TABLE orders
ALTER COLUMN customer_id SET STATISTICS 500;

ANALYZE orders;
```

Dupa ce am crescut target-ul la 500, estimarile de cardinalitate ale optimizatorului pentru join-urile cu `customers` au devenit mult mai precise.

Regula: daca o coloana e folosita frecvent in WHERE sau JOIN si are distributie neuniforma, creste target-ul. 500 e un bun punct de plecare. Poti ajunge la 1000, dar peste aceasta valoare rareori ajuta si incetineste ANALYZE-ul insusi.

------------------------------------------------------------------------

## ⚠️ Cand sa fortezi plannerul: enable_nestloop si enable_hashjoin

Uneori, chiar si cu statistici actualizate, optimizatorul ia un drum gresit. Se intampla cu query-uri complexe, multe tabele in join, sau cand corelatia intre coloane inseala estimarile.

PostgreSQL ofera parametri pentru a dezactiva strategii specifice:

``` sql
SET enable_nestloop = off;
```

Asta forteaza optimizatorul sa nu foloseasca nested loop. Nu e o solutie, e un plasture de diagnostic. Daca dezactivezi nested loop-ul si query-ul trece de la 45 de secunde la 3 secunde, ai confirmat ca problema era alegerea join-ului. Dar nu poti lasa `enable_nestloop = off` in productie pentru ca exista o mie de query-uri unde nested loop-ul e alegerea corecta.

Folosesc acesti parametri doar in doua scenarii:

1. **Diagnostic**: pentru a confirma care strategie de join e problema
2. **Urgenta**: cand business-ul e blocat si trebuie sa repornesti un query critic in timp ce cauti solutia reala

Dupa diagnostic, fix-ul corect e intotdeauna pe statistici, indecsi sau rescrierea query-ului.

------------------------------------------------------------------------

## 📋 Workflow-ul meu cand un query e lent

Dupa treizeci de ani facand aceasta meserie, procesul meu a devenit aproape mecanic:

**1. EXPLAIN (ANALYZE, BUFFERS)** — intotdeauna cu BUFFERS. Salvez output-ul complet, nu doar ultimele randuri.

**2. Caut discrepanta de randuri** — compar `rows=` estimat cu `actual rows=` real pe fiecare nod. Incep de la nodurile frunza si urc spre radacina. Prima discrepanta semnificativa e aproape intotdeauna cauza.

**3. Verific statisticile** — ma uit la `pg_stats` pentru coloanele implicate. Verific `last_autoanalyze` si `last_analyze` in `pg_stat_user_tables`. Daca ultimul ANALYZE e vechi, il lansez si reevaluez.

**4. Evaluez BUFFERS** — daca `shared read` e foarte mare fata de `shared hit`, problema ar putea fi I/O, nu planul. In acel caz fix-ul e `shared_buffers` sau working set-ul pur si simplu nu incape in RAM.

**5. Testez alternative** — daca statisticile sunt actualizate dar planul e inca gresit, folosesc `enable_nestloop`, `enable_hashjoin`, `enable_mergejoin` pentru a intelege care strategie functioneaza mai bine. Apoi incerc sa ghidez optimizatorul spre acea strategie cu indecsi sau rescriere.

Nimic spectaculos. Niciun truc magic. Doar citirea sistematica a planului, un rand pe rand.

------------------------------------------------------------------------

## 💬 Lectia din acea zi

Colegul meu, dupa ce a vazut diferenta, mi-a zis: "Deci era de ajuns un ANALYZE?"

Da si nu. In acel caz specific, da. Dar ideea nu e comanda. Ideea e sa stii sa citesti planul ca sa intelegi *unde* sa te uiti. EXPLAIN ANALYZE iti da datele. Tu trebuie sa le interpretezi.

Am vazut DBA cu ani de experienta lansand EXPLAIN ANALYZE, uitandu-se la timpul total de la sfarsit, si zicand "query-ul e lent". E ca si cum ai lua temperatura unui pacient si ai zice "are febra". Da, dar de la ce?

Planul de executie iti spune de la ce. Fiecare nod e un organ. Randurile estimate fata de cele reale sunt valorile de laborator. Buffer-urile sunt radiografiile. Si ANALYZE-ul e antibioticul care rezolva 70% din cazuri.

Dar pentru acel 30% ramas, trebuie sa citesti. Rand cu rand. Nod cu nod. Nu exista scurtatura.

------------------------------------------------------------------------

## Glosar

**[Execution Plan](/ro/glossary/execution-plan/)** — secventa de operatii (scan, join, sort) pe care baza de date o alege pentru a rezolva o interogare SQL. Se vizualizeaza cu EXPLAIN si EXPLAIN ANALYZE.

**[Nested Loop](/ro/glossary/nested-loop/)** — strategie de join care pentru fiecare rand din tabelul extern cauta corespondentele in tabelul intern. Ideala pentru putine randuri, dezastruoasa pe volume mari cand este aleasa din greseala de optimizator.

**[Hash Join](/ro/glossary/hash-join/)** — strategie de join care construieste o hash table din tabelul mai mic si apoi scaneaza tabelul mai mare cautand corespondente cu lookup-uri O(1). Eficienta pe volume mari fara indexuri.

**[ANALYZE](/ro/glossary/postgresql-analyze/)** — comanda PostgreSQL care colecteaza statistici despre distributia datelor in tabele, folosite de optimizator pentru a estima cardinalitatea si a alege planul de executie.

**[default_statistics_target](/ro/glossary/postgresql-default-statistics-target/)** — parametrul PostgreSQL care defineste cate esantioane sa colecteze per coloana in timpul ANALYZE. Valoarea implicita este 100; pe coloane cu distributie asimetrica este recomandat sa fie crescut la 500-1000.
