---
title: "Split-brain"
description: "Condiție critică într-un cluster de baze de date unde două sau mai multe părți operează independent, acceptând scrieri divergente pe aceleași date."
translationKey: "glossary_split-brain"
articles:
  - "/posts/mysql/galera-cluster-3-nodi"
---

**Split-brain-ul** este o condiție critică care apare când un cluster de baze de date se împarte în două sau mai multe partiții care nu pot comunica între ele, iar fiecare partiție continuă să accepte scrieri independent. Rezultatul sunt date divergente imposibil de reconciliat automat.

## Cum funcționează

Într-un cluster cu 3 noduri, dacă rețeaua dintre Nodul 1 și Nodurile 2-3 se întrerupe, fără protecția quorum-ului ambele părți ar putea continua să accepte scrieri. Când rețeaua se restabilește, clusterul s-ar găsi cu două versiuni diferite ale acelorași date. Mecanismul de quorum previne acest scenariu: doar partiția cu majoritatea nodurilor (quorum) poate continua să funcționeze.

## La ce folosește

Înțelegerea split-brain-ului este fundamentală pentru proiectarea de clustere de baze de date fiabile. Este motivul principal pentru care Galera Cluster necesită un număr impar de noduri (3, 5, 7) și implementează mecanismul de quorum. Cu un număr par de noduri, o partiție de rețea poate împărți clusterul în două jumătăți egale, niciuna dintre ele neavând quorum.

## Când se folosește

Termenul split-brain descrie un risc de evitat, nu o funcționalitate de activat. În Galera, protecția este automată: nodurile care pierd quorum-ul trec în starea Non-Primary și refuză scrierile. Parametrul `pc.ignore_quorum` dezactivează această protecție, dar folosirea lui în producție este puternic descurajată.
