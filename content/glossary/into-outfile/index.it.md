---
title: "INTO OUTFILE"
description: "Clausola SQL di MySQL che permette di scrivere il risultato di una SELECT direttamente su un file nel filesystem del server."
translationKey: "glossary_into-outfile"
articles:
  - "/posts/mysql/mysql-multi-istanza-secure-file-priv"
---

**INTO OUTFILE** è una clausola SQL di MySQL che permette di esportare il risultato di una query direttamente in un file sul filesystem del server database. È il metodo nativo per generare file CSV, TSV o con separatori personalizzati.

## Come funziona

La clausola si aggiunge alla fine di una `SELECT` e specifica il percorso del file di destinazione. I parametri `FIELDS TERMINATED BY`, `ENCLOSED BY` e `LINES TERMINATED BY` controllano il formato dell'output. Il file viene creato dall'utente di sistema MySQL (non dall'utente che esegue la query), quindi deve trovarsi in una directory con i permessi corretti.

## A cosa serve

`INTO OUTFILE` è utile per export massivi di dati dal database in file di testo strutturati. È il complemento di `LOAD DATA INFILE`, che fa l'operazione inversa (importa dati da file). Insieme formano il meccanismo nativo di MySQL per bulk import/export.

## Quando si usa

L'uso è vincolato alla direttiva `secure-file-priv`: il file di destinazione deve trovarsi nella directory autorizzata. Quando `secure-file-priv` blocca il percorso desiderato, l'alternativa è usare il client mysql da shell con `-B -e` e redirigere l'output, che non è soggetto alle stesse restrizioni.
