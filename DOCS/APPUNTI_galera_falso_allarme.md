# Appunti — Galera Cluster: falso allarme (2026-03-10)

## Contesto
- Cluster Galera a 3 nodi, MariaDB 10.6.11
- Nodi: pglsdb01c, pglsdb03c, pglsdb04c
- pglsdb04c e' il nodo che ha fatto il bootstrap (`--wsrep-new-cluster`)

## Cosa e' successo
- Il nodo **pglsdb01c** e' andato giu' brevemente e si e' riavviato alle 10:51 del 10 marzo 2026
- Ha fatto scattare l'allarme di monitoring
- Il nodo e' rientrato da solo nel cluster (probabilmente IST, assenza breve)
- Il cluster non ha mai perso il quorum (2 su 3 = maggioranza)

## Diagnosi eseguita

### Comandi usati
1. `systemctl status mariadb.service` su tutti e 3 i nodi
2. `SHOW STATUS LIKE 'wsrep_cluster_status'` — Primary su tutti
3. `SHOW STATUS LIKE 'wsrep_cluster_size'` — 3 su tutti
4. `SHOW STATUS LIKE 'wsrep_connected'` — ON su tutti
5. `SHOW STATUS LIKE 'wsrep_cluster_state_uuid'` — stesso UUID su tutti (`ea048e66-c12a-11f0-85dc-cbebc6fbb9a6`)
6. `SHOW STATUS LIKE 'wsrep_cluster_conf_id'` — 13 su tutti

### Risultato
Tutto OK. Cluster sano, tutti i nodi sincronizzati, stessa vista del cluster.

## Dettaglio nodi al momento della diagnosi

| Nodo       | Uptime da         | Tasks | Memoria | Note                          |
|------------|-------------------|-------|---------|-------------------------------|
| pglsdb04c  | 27/02 15:34       | 58    | 45.0G   | Bootstrap node                |
| pglsdb03c  | 27/02 15:35       | 16    | 37.1G   | Meno traffico? Da verificare  |
| pglsdb01c  | 10/03 10:51       | 48    | 12.6G   | Riavviato oggi, rientrato OK  |

## Osservazione
pglsdb03c ha significativamente meno task (16 vs 58/48) — potrebbe non ricevere traffico dal load balancer, oppure semplicemente meno connessioni attive in quel momento.

## Spunti per il post
- Galera gestisce il rientro automatico dei nodi senza intervento manuale
- Il quorum a maggioranza (2/3) garantisce continuita' di servizio
- Importanza di avere una checklist di comandi wsrep_ pronti per la diagnosi rapida
- IST vs SST: se il nodo rientra velocemente, il gcache basta per il delta (IST)
- Il `wsrep_cluster_conf_id` uguale su tutti i nodi conferma che non c'e' stato split-brain
- Falso allarme ≠ tempo perso: la verifica sistematica e' comunque buona pratica
