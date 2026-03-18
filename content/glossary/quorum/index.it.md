---
title: "Quorum"
description: "Meccanismo di consenso basato sulla maggioranza dei nodi, usato nei cluster database per prevenire lo split-brain e garantire la consistenza dei dati."
translationKey: "glossary_quorum"
articles:
  - "/posts/mysql/galera-cluster-3-nodi"
---

Il **quorum** è il numero minimo di nodi che devono essere d'accordo per considerare il cluster operativo. In un cluster a 3 nodi, il quorum è 2 (la maggioranza). Se un nodo si disconnette, gli altri due riconoscono di essere la maggioranza e continuano a operare normalmente.

## Come funziona

Galera Cluster utilizza un protocollo di comunicazione di gruppo che verifica continuamente quanti nodi sono raggiungibili. Il calcolo è semplice: quorum = (numero totale nodi / 2) + 1. Con 3 nodi il quorum è 2, con 5 nodi è 3. I nodi che perdono il quorum passano allo stato Non-Primary e rifiutano le scritture per evitare inconsistenze.

## A cosa serve

Il quorum previene lo **split-brain**: la situazione in cui due parti del cluster operano indipendentemente, accettando scritture diverse sugli stessi dati. Senza quorum, un'interruzione di rete potrebbe portare a dati divergenti impossibili da riconciliare automaticamente.

## Quando si usa

Il quorum è attivo automaticamente in qualsiasi cluster Galera. Per questo motivo, **tre nodi è il minimo in produzione**: con due nodi, la perdita di uno lascia il superstite senza quorum e quindi bloccato.
