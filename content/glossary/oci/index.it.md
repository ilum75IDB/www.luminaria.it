---
title: "OCI"
description: "Oracle Cloud Infrastructure — la piattaforma cloud di Oracle, con vantaggi significativi di licensing per i database Oracle grazie al programma BYOL."
translationKey: "glossary_oci"
aka: "Oracle Cloud Infrastructure"
articles:
  - "/posts/oracle/oracle-cloud-migration"
---

**OCI** (Oracle Cloud Infrastructure) è la piattaforma cloud di Oracle, lanciata nella sua seconda generazione nel 2018. A differenza di altri cloud provider, OCI è progettata nativamente per i workload Oracle Database e offre vantaggi significativi in termini di licensing e performance.

## Perché OCI per Oracle Database

Il vantaggio principale riguarda il licensing. Su OCI, Oracle riconosce le proprie OCPU (Oracle CPU) con un rapporto 1:1 ai fini del conteggio delle licenze. Su altri cloud provider come AWS o Azure, il rapporto vCPU-licenze è meno favorevole e il rischio di audit è concreto.

Il programma **BYOL** (Bring Your Own License) permette di riutilizzare le licenze on-premises esistenti su OCI senza costi aggiuntivi — un fattore decisivo per le aziende che hanno già investito in licenze Enterprise Edition.

## Servizi principali per i DBA

- **Bare Metal DB Systems**: server fisici dedicati con Oracle Database preinstallato
- **VM DB Systems**: istanze virtuali con configurazione flessibile (Flex shapes)
- **Exadata Cloud Service**: Exadata completo gestito in cloud
- **Autonomous Database**: database completamente gestito con tuning automatico

## Networking e connettività

OCI offre **FastConnect** per connessioni dedicate ad alta banda tra data center on-premises e la cloud region, oltre a VPN site-to-site per scenari con requisiti di banda inferiori. La latenza e la bandwidth del collegamento sono fattori critici nelle migrazioni con Data Guard cross-site.
