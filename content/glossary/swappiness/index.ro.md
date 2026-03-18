---
title: "Swappiness"
description: "Parametru kernel Linux (vm.swappiness) care controlează propensiunea sistemului de a muta pagini de memorie în swap, critic pentru serverele de baze de date unde SGA trebuie să rămână în RAM."
translationKey: "glossary_swappiness"
aka: "vm.swappiness"
articles:
  - "/posts/oracle/oracle-linux-kernel"
---

**Swappiness** (`vm.swappiness`) este un parametru al kernelului Linux care controlează cât de agresiv mută sistemul pagini de memorie din RAM în swap pe disc. Valoarea variază de la 0 (swap doar în cazuri extreme) la 100 (swap agresiv). Valoarea implicită este 60.

## Cum funcționează

Cu valoarea implicită de 60, Linux începe să facă swap când presiunea pe memorie este încă relativ scăzută. Pentru un server de baze de date dedicat, acest lucru este inacceptabil: SGA trebuie să rămână în RAM, întotdeauna. Valoarea recomandată pentru Oracle este 1 — nu 0, care ar dezactiva complet swap-ul și ar putea declanșa OOM killer-ul.

## La ce servește

Valoarea 1 îi spune kernelului: "Fă swap doar dacă nu mai există cu adevărat altă alternativă." Acest lucru garantează că SGA și structurile critice ale Oracle rămân în memoria fizică, evitând citirile din swap (cu ordine de mărime mai lente decât RAM-ul) în timpul execuției interogărilor.

## De ce contează

Cu swappiness la 60, un server cu 128 GB de RAM și o SGA de 64 GB poate începe să facă swap la părți din SGA chiar și cu 20-30 GB de RAM liber. Rezultatul sunt performanțe degradate imprevizibil, cu vârfuri de latență care par probleme aplicative dar sunt de fapt sistemul de operare care mută memorie pe disc.
