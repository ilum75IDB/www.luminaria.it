---
title: "Single-primary"
description: "Modalità di MySQL Group Replication in cui un solo nodo accetta scritture, mentre gli altri sono in sola lettura con failover automatico."
translationKey: "glossary_single_primary"
aka: "Single-primary mode"
articles:
  - "/posts/mysql/mysql-group-replication-binlog-migration"
---

**Single-primary** è la modalità operativa più comune di MySQL Group Replication, in cui un solo nodo del cluster — il primary — accetta operazioni di scrittura. Gli altri nodi (secondary) sono in sola lettura (`read_only=ON`, `super_read_only=ON`) e ricevono le modifiche tramite la replica sincrona del gruppo.

## Come funziona

Il parametro `group_replication_single_primary_mode=ON` attiva questa modalità. Il primary è l'unico nodo con `read_only=OFF`. Se il primary viene fermato o diventa irraggiungibile, il cluster esegue un'elezione automatica e uno dei secondary diventa il nuovo primary in pochi secondi.

## Perché si usa

La modalità single-primary evita i conflitti di scrittura concorrente tipici del multi-primary. In produzione la maggior parte dei cluster MySQL usa questa modalità perché è più prevedibile: le applicazioni scrivono su un solo endpoint, la replica è lineare e il debugging è più semplice.

## Cosa può andare storto

Quando si ferma il primary per manutenzione, il cluster fa un failover automatico. Durante quei secondi le connessioni attive possono essere interrotte e le transazioni in corso possono fallire. È un disservizio breve ma va comunicato. La regola pratica: in un intervento di manutenzione su un cluster single-primary, i secondary si toccano prima, il primary per ultimo.
