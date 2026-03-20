---
title: "Exchange Partition"
description: "Operazione DDL Oracle che scambia istantaneamente i segmenti dati tra una tabella non partizionata e una partizione, senza spostare fisicamente i dati."
translationKey: "glossary_exchange-partition"
aka: "Scambio di partizione"
articles:
  - "/posts/data-warehouse/partitioning-dwh"
---

L'**Exchange Partition** è un'operazione DDL di Oracle che permette di scambiare istantaneamente il contenuto di una partizione con quello di una tabella non partizionata. Non viene spostato nemmeno un byte di dati — l'operazione modifica solo i puntatori nel data dictionary.

## Come funziona

Il comando `ALTER TABLE ... EXCHANGE PARTITION ... WITH TABLE ...` modifica i metadati nel data dictionary in modo che i segmenti fisici della partizione e della tabella di staging si scambino. La tabella di staging diventa la partizione e viceversa. L'operazione dura meno di un secondo indipendentemente dal volume dei dati, perché non implica alcun movimento fisico.

## A cosa serve

Nei data warehouse, l'exchange partition è lo strumento principale per il caricamento bulk dei dati. Il processo tipico è: l'ETL carica i dati in una tabella di staging, costruisce gli indici, valida i dati, e poi esegue l'exchange con la partizione target. Durante l'exchange, le query sulle altre partizioni continuano a funzionare senza interruzione.

## Cosa può andare storto

La clausola `WITHOUT VALIDATION` salta il controllo che i dati della staging table rientrino effettivamente nell'intervallo della partizione — velocizza l'operazione ma richiede che l'ETL garantisca la correttezza dei dati. Se i dati nella staging contengono date fuori intervallo, finiscono nella partizione sbagliata senza alcun errore. La clausola `INCLUDING INDEXES` richiede che la staging table abbia indici con la stessa struttura degli indici locali della tabella partizionata.
