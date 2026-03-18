---
title: "Quorum"
description: "Mecanism de consens bazat pe majoritatea nodurilor, folosit în clusterele de baze de date pentru prevenirea split-brain-ului și garantarea consistenței datelor."
translationKey: "glossary_quorum"
articles:
  - "/posts/mysql/galera-cluster-3-nodi"
---

**Quorum-ul** este numărul minim de noduri care trebuie să fie de acord pentru ca clusterul să fie considerat operațional. Într-un cluster cu 3 noduri, quorum-ul este 2 (majoritatea). Dacă un nod se deconectează, celelalte două recunosc că sunt majoritatea și continuă să funcționeze normal.

## Cum funcționează

Galera Cluster folosește un protocol de comunicare de grup care verifică continuu câte noduri sunt accesibile. Calculul este simplu: quorum = (număr total noduri / 2) + 1. Cu 3 noduri quorum-ul este 2, cu 5 noduri este 3. Nodurile care pierd quorum-ul trec în starea Non-Primary și refuză scrierile pentru a evita inconsistențele.

## La ce folosește

Quorum-ul previne **split-brain-ul**: situația în care două părți ale clusterului operează independent, acceptând scrieri diferite pe aceleași date. Fără quorum, o întrerupere de rețea ar putea duce la date divergente imposibil de reconciliat automat.

## Când se folosește

Quorum-ul este activ automat în orice cluster Galera. Din acest motiv, **trei noduri este minimul în producție**: cu două noduri, pierderea unuia lasă supraviețuitorul fără quorum și prin urmare blocat.
