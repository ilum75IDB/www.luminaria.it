---
title: "Swappiness"
description: "Parametro kernel Linux (vm.swappiness) che controlla la propensione del sistema a spostare pagine di memoria nello swap, critico per i server database dove la SGA deve restare in RAM."
translationKey: "glossary_swappiness"
aka: "vm.swappiness"
articles:
  - "/posts/oracle/oracle-linux-kernel"
---

La **Swappiness** (`vm.swappiness`) è un parametro del kernel Linux che controlla quanto aggressivamente il sistema sposta pagine di memoria dalla RAM allo swap su disco. Il valore va da 0 (swap solo in caso estremo) a 100 (swap aggressivo). Il default è 60.

## Come funziona

Con il valore di default 60, Linux inizia a swappare quando la pressione sulla memoria è ancora relativamente bassa. Per un server database dedicato, questo è inaccettabile: l'SGA deve restare in RAM, sempre. Il valore raccomandato per Oracle è 1 — non 0, che disabiliterebbe completamente lo swap e potrebbe causare OOM killer.

## A cosa serve

Il valore 1 dice al kernel: "Swappa solo se non c'è davvero più alternativa." Questo garantisce che la SGA e le strutture critiche di Oracle restino in memoria fisica, evitando letture da swap (ordini di grandezza più lente della RAM) durante l'esecuzione delle query.

## Perché è critico

Con swappiness a 60, un server con 128 GB di RAM e 64 GB di SGA può iniziare a swappare parti della SGA anche con 20-30 GB di RAM libera. Il risultato sono performance degradate in modo imprevedibile, con picchi di latenza che sembrano problemi applicativi ma sono in realtà il sistema operativo che sposta memoria su disco.
