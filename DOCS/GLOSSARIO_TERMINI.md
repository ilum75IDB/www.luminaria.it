# Glossario Termini — Database Strategy Blog

Tabella centralizzata di tutti i termini tecnici e acronimi presenti nelle sezioni Glossario degli articoli del blog.

## Come usare questo file

- Ogni volta che si scrive un nuovo articolo con la sezione Glossario, aggiungere qui i termini nuovi
- Se un termine esiste gia, aggiornare solo la colonna **Contenuto in** aggiungendo il nuovo articolo
- I termini sono ordinati alfabeticamente
- La colonna **Contenuto in** elenca gli slug degli articoli (link relativi alla sezione)

## Tabella Termini

| Termine | Descrizione | Contenuto in |
|---------|-------------|--------------|
| ANALYZE | Comando PostgreSQL che raccoglie statistiche sulla distribuzione dei dati nelle tabelle, usate dall'optimizer per scegliere il piano di esecuzione | explain-analyze-postgresql |
| ASH | Active Session History — componente Oracle che campiona lo stato di ogni sessione attiva una volta al secondo, usato per la diagnosi puntuale dei problemi di performance | oracle-awr-ash |
| Binary log | Registro binario sequenziale di MySQL che traccia tutte le modifiche ai dati (INSERT, UPDATE, DELETE, DDL), usato per la replica e il point-in-time recovery | binary-log-mysql |
| AWR | Automatic Workload Repository — strumento diagnostico integrato in Oracle Database per la raccolta e l'analisi delle statistiche di performance | oracle-awr-ash |
| default_statistics_target | Parametro PostgreSQL che definisce quanti campioni raccogliere per colonna durante l'ANALYZE. Il default è 100; su colonne con distribuzione asimmetrica conviene alzarlo a 500-1000 | explain-analyze-postgresql |
| Chiave surrogata | Identificativo numerico generato dal data warehouse, distinto dalla chiave naturale del sistema sorgente. Nella SCD Tipo 2 è indispensabile perché lo stesso record può avere più versioni | scd-tipo-2 |
| CDC | Change Data Capture — tecnica per intercettare e propagare le modifiche ai dati in tempo reale, spesso basata sulla lettura dei log delle transazioni | binary-log-mysql |
| COALESCE | Funzione SQL che restituisce il primo valore non NULL da una lista di espressioni. Spesso usata come workaround per le gerarchie incomplete, ma non risolve il problema strutturale nel modello | ragged-hierarchies |
| Data Guard | Tecnologia Oracle per la replica in tempo reale di un database su uno o più server standby, garantendo alta disponibilità e disaster recovery | oracle-data-guard |
| Drill-down | Navigazione nei report dal livello aggregato al livello di dettaglio. Richiede una gerarchia completa e bilanciata per funzionare correttamente | ragged-hierarchies |
| ETL | Extract, Transform, Load — processo di estrazione, trasformazione e caricamento dati dai sistemi sorgente al data warehouse | scd-tipo-2, ragged-hierarchies |
| Execution Plan | Sequenza di operazioni (scan, join, sort) che il database sceglie per risolvere una query SQL. Si visualizza con EXPLAIN e EXPLAIN ANALYZE | explain-analyze-postgresql |
| Fact table | Tabella centrale dello star schema che contiene le misure numeriche (importi, quantità, conteggi) e le chiavi esterne verso le tabelle dimensionali | scd-tipo-2 |
| Full Table Scan | Operazione di lettura in cui il database legge tutti i blocchi di una tabella senza utilizzare indici. In Oracle si manifesta come wait event `db file scattered read` | oracle-awr-ash |
| Hash Join | Strategia di join che costruisce una hash table dalla tabella più piccola e poi scansiona la più grande cercando corrispondenze con lookup O(1). Efficiente su grandi volumi senza indici | explain-analyze-postgresql |
| Kimball | Ralph Kimball — metodologia di progettazione data warehouse basata su dimensional modeling, star schema e processi ETL bottom-up. Riferimento standard per la classificazione delle SCD | scd-tipo-2 |
| MERGE | Istruzione SQL che combina INSERT e UPDATE in un'unica operazione: se il record esiste lo aggiorna, se non esiste lo inserisce. In Oracle anche nota come "upsert" | scd-tipo-2 |
| mysqlbinlog | Utility da riga di comando di MySQL per leggere, filtrare e riapplicare il contenuto dei file binary log. Indispensabile per il point-in-time recovery e il debug della replica | binary-log-mysql |
| Nested Loop | Strategia di join che per ogni riga della tabella esterna cerca le corrispondenze nella tabella interna. Ideale per poche righe, disastrosa su grandi volumi | explain-analyze-postgresql |
| OLAP | Online Analytical Processing — elaborazione orientata all'analisi multidimensionale dei dati, tipica dei data warehouse. Contrapposta all'OLTP dei sistemi transazionali | ragged-hierarchies |
| PITR | Point-in-Time Recovery — tecnica di ripristino che combina backup e binary log per riportare un database a un qualsiasi momento nel tempo | binary-log-mysql |
| Ragged hierarchy | Gerarchia in cui non tutti i rami raggiungono la stessa profondità: alcuni livelli intermedi sono assenti. Tipica nelle anagrafiche clienti e strutture organizzative | ragged-hierarchies |
| Redo Log | File di log in cui Oracle registra ogni modifica ai dati prima di scriverla nei datafile, garantendo il recovery e la replica Data Guard | oracle-data-guard |
| Relay log | File di log intermedio sullo slave MySQL che riceve gli eventi dal binary log del master prima che vengano eseguiti localmente | binary-log-mysql |
| RMAN | Recovery Manager — strumento nativo Oracle per backup, restore e recovery del database, inclusa la creazione di database standby per Data Guard | oracle-data-guard |
| RPO | Recovery Point Objective — la quantità massima di dati che un'organizzazione può permettersi di perdere in caso di disastro, misurata in tempo | oracle-data-guard |
| RTO | Recovery Time Objective — il tempo massimo accettabile per ripristinare un servizio dopo un guasto o un disastro | oracle-data-guard |
| SCD | Slowly Changing Dimension — tecnica di data warehouse per tracciare le variazioni nel tempo dei dati nelle tabelle dimensionali | scd-tipo-2 |
| Self-parenting | Tecnica di bilanciamento delle gerarchie sbilanciate: chi non ha un padre diventa padre di sé stesso, eliminando i NULL dalla dimensione | ragged-hierarchies |
| Snapshot (Oracle) | Istantanea delle statistiche di performance catturata periodicamente da AWR (di default ogni 60 minuti) e usata per generare report diagnostici comparativi | oracle-awr-ash |
| Star schema | Modello di dati tipico del data warehouse: una fact table al centro collegata a più tabelle dimensionali tramite chiavi esterne. Semplifica le query analitiche e ottimizza le performance | scd-tipo-2 |
| Wait Event | Evento di attesa registrato da Oracle ogni volta che una sessione non può procedere e deve attendere una risorsa (I/O, lock, CPU, rete). L'analisi dei wait event è la base della metodologia diagnostica Oracle | oracle-awr-ash |

---

**Ultimo aggiornamento**: 2026-03-18
**Totale termini**: 32
**Totale articoli con glossario**: 6
