---
title: "Range Partitioning"
description: "Strategia di partizionamento che divide una tabella in segmenti basati su intervalli di valori di una colonna, tipicamente una data."
translationKey: "glossary_range-partitioning"
aka: "Partizionamento per intervallo"
articles:
  - "/posts/data-warehouse/partitioning-dwh"
  - "/posts/oracle/oracle-partitioning"
---

Il **Range Partitioning** (partizionamento per intervallo) è una strategia di partizionamento delle tabelle in cui le righe vengono distribuite in partizioni diverse in base al valore di una colonna rispetto a intervalli predefiniti. La colonna di partizionamento è quasi sempre una data nei data warehouse.

## Come funziona

Ogni partizione è definita con una clausola `VALUES LESS THAN` che stabilisce il limite superiore dell'intervallo. Oracle assegna automaticamente ogni riga alla partizione corretta in base al valore della colonna di partizionamento. Se una riga ha `data_vendita = '2025-03-15'`, viene inserita nella partizione il cui intervallo include quella data.

## Quando si usa

Il range partitioning è la scelta naturale quando i dati hanno una dimensione temporale dominante — fact table nei data warehouse, tabelle di log, tabelle di transazioni. La granularità della partizione (giornaliera, mensile, trimestrale) dipende dal volume di inserimento e dalla tipologia delle query: partizioni troppo piccole generano overhead di gestione, troppo grandi riducono l'efficacia del partition pruning.

## Vantaggi operativi

Oltre alle performance delle query, il range partitioning abilita operazioni di gestione del ciclo di vita dei dati impossibili su tabelle monolitiche: drop istantaneo di una partizione (senza DELETE), compressione selettiva delle partizioni storiche, spostamento su storage diversi (ILM — Information Lifecycle Management), e exchange partition per caricamenti bulk a impatto zero.
