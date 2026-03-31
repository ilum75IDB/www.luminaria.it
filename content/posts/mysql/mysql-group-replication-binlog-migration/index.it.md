---
title: "Disco pieno su un cluster MySQL: binary log, Group Replication e la migrazione che non ammette errori"
description: "Filesystem al 92% su un cluster MySQL Group Replication a 3 nodi. La causa? Binary log accumulati sul volume principale. Dall'allarme alla migrazione su volume dedicato, nodo per nodo, senza perdere il quorum."
date: "2025-10-14T08:03:00+01:00"
draft: false
translationKey: "mysql_group_replication_binlog_migration"
tags: ["group-replication", "binary-log", "disk-space", "cluster", "innodb-cluster"]
categories: ["mysql"]
image: "mysql-group-replication-binlog-migration.cover.jpg"
---

La notifica è arrivata un lunedì mattina, in mezzo a tre riunioni e un caffè ancora caldo. "Filesystem /mysql all'85% sul nodo primario." Su un altro nodo era al 66%, sul terzo al 25%. In un cluster, quando i numeri non tornano tra i nodi, c'è sempre qualcosa sotto.

La prima domanda che ti viene in mente è "quanto spazio serve?". Ma è la domanda sbagliata. Quella giusta è: "perché si sta riempiendo?"

---

## La causa: binary log sul volume sbagliato

Controllare è stato rapido:

```sql
SHOW VARIABLES LIKE 'log_bin';
```

Risultato: `ON`. I binary log erano attivi — come ci si aspetta in un cluster. Ma la cosa che non andava era il path:

```sql
SHOW VARIABLES LIKE 'log_bin_basename';
```

```
/mysql/bin_log/binlog
```

I binlog stavano sullo stesso volume dei dati: `/mysql`. Un volume da circa 3 TB che su un nodo era già all'85%.

Ho verificato anche la retention:

```sql
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';
```

```
604800
```

Sette giorni. Non un valore assurdo, ma con tre nodi che scrivono binlog locali su un volume condiviso con i dati, sette giorni di binlog possono pesare parecchio — specialmente se il carico di scrittura è alto.

Il problema era chiaro: i binary log stavano mangiando lo spazio del filesystem principale. Non un bug, non una tabella che cresce a dismisura. Solo una scelta architetturale iniziale mai rivista.

---

## Ma che cluster è, esattamente?

Prima di toccare qualsiasi cosa su un server MySQL — prima ancora di pensare a spostare un file — devi sapere cosa hai davanti. "È un cluster" non basta. MySQL ha almeno quattro modi diversi di fare alta disponibilità, e ognuno ha le sue regole.

Ho iniziato con la replica classica:

```sql
SHOW SLAVE STATUS\G
```

Empty set su entrambi i nodi che ho controllato. Nessuna replica tradizionale attiva.

Poi ho provato con `SHOW REPLICA STATUS` — ma su MySQL 8.0.20 quel comando non esiste ancora. È stato introdotto nella 8.0.22. Un dettaglio che la documentazione online spesso dimentica di specificare, e che ti fa perdere cinque minuti a cercare un errore di sintassi che non è un errore.

Passaggio successivo — Group Replication:

```sql
SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
```

Ed ecco la risposta:

| MEMBER_HOST | MEMBER_STATE | MEMBER_ROLE |
|-------------|-------------|-------------|
| dbcluster01 | ONLINE | SECONDARY |
| dbcluster02 | ONLINE | SECONDARY |
| dbcluster03 | ONLINE | PRIMARY |

Tre nodi. Tutti ONLINE. Un primary, due secondary. Group Replication in modalità single-primary.

Conferma finale dai plugin:

```sql
SHOW PLUGINS;
```

In mezzo alla lista: `group_replication | ACTIVE | GROUP REPLICATION | group_replication.so`. E dalla configurazione:

```sql
SHOW VARIABLES LIKE 'group_replication_single_primary_mode';
```

```
ON
```

Ora sapevo cosa avevo davanti. Non una replica classica, non un Galera, non un cluster NDB. Un MySQL Group Replication single-primary con tre nodi, GTID abilitati, binlog in formato ROW. Il quadro era completo.

La tentazione è sempre quella di saltare questa fase. "Tanto so che è un cluster, muoviamoci." Ma saltare la diagnosi su un cluster è come operare senza la TAC: puoi avere fortuna, o puoi fare un disastro.

---

## La soluzione: un volume dedicato per i binary log

La strategia era semplice: i binlog devono stare su un volume separato. Non sullo stesso filesystem dei dati, non su un symlink improvvisato, non su una directory condivisa. Un volume dedicato, montato nello stesso path su tutti e tre i nodi.

Ho chiesto ai sistemisti di creare un nuovo volume da 600 GB con mount point `/mysql/binary_logs` su ciascuno dei tre nodi.

Quando il volume è stato pronto, ho verificato su tutti e tre:

```bash
df -h /mysql/binary_logs
```

| Nodo | /mysql | /mysql/binary_logs |
|------|--------|--------------------|
| dbcluster03 (PRIMARY) | 85% | 1% |
| dbcluster02 (SECONDARY) | 66% | 1% |
| dbcluster01 (SECONDARY) | 25% | 1% |

Spazio fresco e dedicato. Ogni volume su un disco locale della VM di competenza — tre dischi, tre volumi, stesso mountpoint per tutti e tre i nodi. I sistemisti avevano fatto un lavoro pulito.

---

## I check prima di toccare MySQL

Prima di fermare il primo nodo, ho fatto tre controlli che considero obbligatori.

**Permessi sulla directory.** MySQL non parte se non può scrivere nella directory dei binlog. Sembra banale, ma è una delle cause più frequenti di "perché non riparte dopo il cambio config".

```bash
ls -ld /mysql/binary_logs
```

Sui tre nodi i permessi erano 755. Funziona, ma non è il massimo lato sicurezza — i binlog possono contenere dati sensibili. Li ho portati a 750:

```bash
chmod 750 /mysql/binary_logs
```

Risultato: `drwxr-x--- mysql mysql`. Solo l'utente mysql può scrivere e leggere.

**Test di scrittura reale.** Prima di far scrivere MySQL, ho verificato che il filesystem rispondesse:

```bash
touch /mysql/binary_logs/testfile
ls -l /mysql/binary_logs/testfile
rm -f /mysql/binary_logs/testfile
```

Se il touch fallisce, il problema è dello storage o dei permessi — e meglio scoprirlo adesso che dopo un restart di MySQL.

**Stato del cluster.** L'ultimo controllo prima di procedere:

```sql
SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
```

Tre nodi ONLINE. Quorum intatto. Si può partire.

---

## La strategia: un nodo alla volta, il primary per ultimo

In un Group Replication a tre nodi, il quorum è due. Se fermi un nodo, gli altri due mantengono il gruppo. Se ne fermi due — hai perso il cluster.

La regola è semplice: **un nodo alla volta, aspettando che il precedente rientri nel gruppo prima di toccare il successivo**. E il primary si fa per ultimo.

Perché? Perché quando fermi il primary, succede una cosa importante: il cluster fa un'elezione automatica e uno dei secondary diventa il nuovo primary. Durante quei secondi — pochi, se tutto è sano — le connessioni attive possono essere droppate, le transazioni in corso possono fallire. È un disservizio breve, ma è un disservizio. Va comunicato.

L'ordine che ho seguito:

1. **dbcluster01** (SECONDARY)
2. **dbcluster02** (SECONDARY)
3. **dbcluster03** (PRIMARY)

---

## La procedura, nodo per nodo

Su ogni nodo la sequenza è identica:

**A. Verifica il ruolo del nodo.** Prima di fermarlo, conferma che sia quello che pensi:

```sql
SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
```

**B. Ferma MySQL:**

```bash
systemctl stop mysqld
```

**C. Modifica la configurazione.** Nel `my.cnf`, cambia il parametro `log_bin`:

Da:
```ini
log_bin=/mysql/bin_log/binlog
```

A:
```ini
log_bin=/mysql/binary_logs/mysql-bin
```

Una riga. Una sola modifica. Non toccare i parametri di Group Replication, non cambiare il `server_id`, non reinventare il motore a vapore mentre cambi una ruota.

**D. Avvia MySQL:**

```bash
systemctl start mysqld
```

**E. Verifica.** Tre cose da controllare:

Il nuovo path:
```sql
SHOW VARIABLES LIKE 'log_bin_basename';
```
Deve restituire `/mysql/binary_logs/mysql-bin`.

Il rientro nel gruppo:
```sql
SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
```
Il nodo deve tornare ONLINE.

I nuovi binlog sul nuovo path:
```bash
ls -lh /mysql/binary_logs/
```
Devono comparire i nuovi file `mysql-bin.000001`.

**Solo quando il nodo è ONLINE e il cluster mostra di nuovo tre nodi attivi, si passa al successivo.** Non prima.

Per il primary — dbcluster03 — la procedura è identica, ma prima di fermarlo ho verificato che entrambi i secondary fossero ONLINE e già migrati. Al momento dello stop, il cluster ha fatto l'elezione. Uno dei secondary è diventato primary. Breve disservizio, come previsto.

---

## Cosa non fare

Dalla mia esperienza, queste sono le trappole più comuni in questo tipo di intervento:

**Non copiare i vecchi binlog nel nuovo path.** In Group Replication non serve fare archeologia binaria. I nuovi binlog verranno creati nella nuova directory dopo il riavvio. Quelli vecchi servono solo se hai bisogno di point-in-time recovery — e in quel caso li sai già dove trovarli.

**Non toccare due nodi contemporaneamente.** Con tre nodi il quorum è sacro. Un nodo per volta, senza eccezioni. Se fermi due nodi insieme, stai giocando a Jenga bendato.

**Non partire dal primary.** Sempre i secondary prima, il primary per ultimo. Fare il contrario è il modo elegante di invitare il caos a cena.

**Non cancellare subito i vecchi binlog.** Dopo il cambio, il vecchio path `/mysql/bin_log/` non verrà più usato per i nuovi file. Ma non fare subito `rm -rf /mysql/bin_log/*`. Aspetta. Verifica che il cluster sia stabile, che i nuovi binlog vengano scritti sul nuovo mount, che non ci siano errori nel log di MySQL. Solo dopo qualche giorno di osservazione, ragiona sulla pulizia.

**Non fidarti solo del fatto che "MySQL è partito".** MySQL può partire ma non rientrare nel gruppo. Devi verificare tre cose: il `log_bin_basename` punta al nuovo path, il nodo è ONLINE nella tabella `replication_group_members`, e i file binlog vengono effettivamente scritti nella nuova directory.

---

## Quello che questa operazione insegna

Un filesystem al 92% non è un'emergenza — è un segnale. Il problema vero non era lo spazio disco, era una scelta architetturale fatta probabilmente al momento dell'installazione e mai più rivista: binlog e dati sullo stesso volume.

Separare i binary log su un volume dedicato non è solo un fix. È hardening dell'infrastruttura. È la differenza tra un sistema che "funziona" e un sistema che è progettato per funzionare anche quando le cose crescono.

E la parte più importante di tutto l'intervento non è stata la modifica del `my.cnf` — quella è una riga. La parte importante è stata la diagnosi: capire che tipo di cluster avevo davanti, verificare lo stato di ogni nodo, preparare lo storage, testare i permessi, pianificare l'ordine di esecuzione. Tutto prima di toccare un solo parametro.

Un DBA senior e un DBA junior conoscono entrambi il comando `systemctl stop mysqld`. La differenza è in tutto quello che succede prima.

------------------------------------------------------------------------

## Glossario

**[Group Replication](/it/glossary/group-replication/)** — Meccanismo nativo di MySQL per la replica sincrona multi-nodo con failover automatico e gestione del quorum. Supporta modalità single-primary e multi-primary.

**[Binary log](/it/glossary/binary-log/)** — Registro binario sequenziale di MySQL che traccia tutte le modifiche ai dati (INSERT, UPDATE, DELETE, DDL), usato per la replica e il point-in-time recovery.

**[GTID](/it/glossary/gtid/)** — Global Transaction Identifier — identificativo univoco assegnato a ogni transazione in MySQL, che semplifica la gestione della replica e il tracking delle transazioni tra i nodi del cluster.

**[Quorum](/it/glossary/quorum/)** — Numero minimo di nodi che devono essere attivi e in comunicazione perché un cluster possa continuare a operare. In un cluster a 3 nodi, il quorum è 2.

**[Single-primary](/it/glossary/single-primary/)** — Modalità di Group Replication in cui un solo nodo accetta scritture, mentre gli altri sono in sola lettura con failover automatico.
