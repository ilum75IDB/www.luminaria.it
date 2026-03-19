---
title: "ETL"
description: "Extract, Transform, Load — processo di estrazione, trasformazione e caricamento dati dai sistemi sorgente al data warehouse."
translationKey: "glossary_etl"
aka: "Extract, Transform, Load"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
  - "/posts/data-warehouse/ragged-hierarchies"
  - "/posts/data-warehouse/fatto-grana-sbagliata"
---

**ETL** (Extract, Transform, Load) è il processo fondamentale attraverso cui i dati vengono spostati dai sistemi sorgente (database operazionali, file, API) al data warehouse.

## Le tre fasi

- **Extract**: estrazione dei dati dai sistemi sorgente. Può essere completa (full load) o incrementale (solo i dati nuovi o modificati)
- **Transform**: pulizia, validazione, standardizzazione e arricchimento dei dati. Qui si applicano le regole di business, le lookup sulle dimensioni, i calcoli derivati
- **Load**: caricamento dei dati trasformati nelle tabelle del data warehouse (fact e dimension)

## Perché è critico

L'ETL è la parte meno visibile ma più critica di un data warehouse. Se i dati vengono estratti in modo incompleto, trasformati con regole errate o caricati senza controlli, tutto ciò che sta sopra — report, dashboard, decisioni — sarà sbagliato.

Un ETL ben progettato è anche quello che determina la finestra di caricamento: quanto tempo serve per aggiornare il data warehouse. In ambienti reali, passare da 4 ore a 25 minuti può fare la differenza tra dati aggiornati alla mattina o al pomeriggio.

## ELT vs ETL

Con l'avvento dei data warehouse cloud e dei motori colonnari ad alte performance, si è diffuso il pattern **ELT** (Extract, Load, Transform): i dati vengono caricati grezzi nel warehouse e trasformati direttamente lì, sfruttando la potenza di calcolo del motore SQL. Il concetto di fondo resta lo stesso, cambia dove avviene la trasformazione.
