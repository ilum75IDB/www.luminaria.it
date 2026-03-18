---
title: "IST"
description: "Incremental State Transfer — mecanism Galera Cluster pentru transferul doar al tranzacțiilor lipsă către un nod care revine în cluster."
translationKey: "glossary_ist"
aka: "Incremental State Transfer"
articles:
  - "/posts/mysql/galera-cluster-3-nodi"
---

**IST** (Incremental State Transfer) este mecanismul prin care un nod Galera care revine în cluster după o absență scurtă primește doar tranzacțiile lipsă, fără a fi nevoie să descarce întregul set de date.

## Cum funcționează

Când un nod se reconectează la cluster, donatorul verifică dacă tranzacțiile lipsă sunt încă disponibile în gcache-ul său (Galera cache). Dacă gap-ul este acoperit de gcache, se execută un IST: doar tranzacțiile lipsă sunt trimise nodului, care le aplică și revine în starea Synced. Dacă gap-ul depășește gcache-ul, Galera recurge la un SST complet.

## La ce folosește

IST face ca revenirea unui nod în cluster să fie mult mai rapidă decât un SST complet. Un nod care a fost offline câteva minute sau ore poate redeveni operațional în câteva secunde, fără impact asupra performanței clusterului.

## Când se folosește

IST este declanșat automat când condițiile o permit. Dimensiunea gcache-ului (`gcache.size`) determină câte tranzacții poate păstra clusterul în memorie pentru a suporta IST. Un gcache mai mare permite perioade mai lungi de inactivitate ale unui nod fără necesitatea unui SST.
