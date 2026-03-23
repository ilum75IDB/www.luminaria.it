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
| Anonymous User | Utente MySQL/MariaDB senza nome creato automaticamente durante l'installazione, rappresenta un rischio di sicurezza in produzione | mysql-users-and-hosts |
| Authentication Plugin | Modulo MySQL/MariaDB che gestisce il metodo di verifica delle credenziali durante la connessione | mysql-users-and-hosts |
| Additive Measure | Misura numerica in una fact table che può essere sommata lungo tutte le dimensioni (importi, quantità, conteggi). Le misure semi-additive e non-additive hanno regole di aggregazione diverse | fatto-grana-sbagliata |
| AI Manager | Ruolo professionale che governa l'impatto dell'intelligenza artificiale su architetture, processi e persone all'interno di un progetto | ai-manager-project-management |
| ANALYZE | Comando PostgreSQL che raccoglie statistiche sulla distribuzione dei dati nelle tabelle, usate dall'optimizer per scegliere il piano di esecuzione | explain-analyze-postgresql, pg-stat-statements |
| ASH | Active Session History — componente Oracle che campiona lo stato di ogni sessione attiva una volta al secondo, usato per la diagnosi puntuale dei problemi di performance | oracle-awr-ash |
| BYOL | Bring Your Own License — programma Oracle che permette di riutilizzare le licenze on-premises nel cloud OCI senza costi aggiuntivi di licensing | oracle-cloud-migration |
| Binary log | Registro binario sequenziale di MySQL che traccia tutte le modifiche ai dati (INSERT, UPDATE, DELETE, DDL), usato per la replica e il point-in-time recovery | binary-log-mysql |
| Brompton | Bicicletta pieghevole britannica considerata il riferimento mondiale per qualità costruttiva, compattezza e praticità nel pendolarismo urbano | bici-vs-auto-roma |
| Branch | Ramo di sviluppo indipendente in un sistema di version control, permette di lavorare su modifiche isolate senza influenzare il codice principale | ai-github-project-management |
| Bicicletta Pieghevole | Bicicletta che si ripiega in 10-20 secondi diventando un bagaglio trasportabile in ufficio, in metro o in treno | bici-vs-auto-roma |
| Bloat | Spazio morto accumulato in una tabella o indice PostgreSQL a causa di dead tuples non rimossi, che gonfia la dimensione su disco | vacuum-autovacuum-postgresql |
| B-Tree | Struttura dati ad albero bilanciato, tipo di indice predefinito nei database relazionali. Efficiente per ricerche di uguaglianza e range, inadatto per LIKE con wildcard iniziale | like-optimization-postgresql |
| Autovacuum | Daemon PostgreSQL che esegue automaticamente VACUUM e ANALYZE sulle tabelle quando il numero di dead tuples supera una soglia configurabile | vacuum-autovacuum-postgresql |
| AWR | Automatic Workload Repository — strumento diagnostico integrato in Oracle Database per la raccolta e l'analisi delle statistiche di performance | oracle-awr-ash, oracle-cloud-migration |
| default_statistics_target | Parametro PostgreSQL che definisce quanti campioni raccogliere per colonna durante l'ANALYZE. Il default è 100; su colonne con distribuzione asimmetrica conviene alzarlo a 500-1000 | explain-analyze-postgresql |
| Chiave surrogata | Identificativo numerico generato dal data warehouse, distinto dalla chiave naturale del sistema sorgente. Nella SCD Tipo 2 è indispensabile perché lo stesso record può avere più versioni | scd-tipo-2 |
| Cutover | Momento critico di una migrazione in cui il sistema di produzione viene spostato definitivamente dalla vecchia alla nuova infrastruttura | oracle-cloud-migration |
| CTAS | Create Table As Select — tecnica Oracle per creare una nuova tabella popolandola con i risultati di una query, usata per migrazioni di tabelle di grandi dimensioni | oracle-partitioning |
| Churn | Misura di quanto una tabella database cambia dopo l'inserimento iniziale dei dati (UPDATE/DELETE). Determina il costo di manutenzione degli indici GIN | like-optimization-postgresql |
| Carbon Footprint | Quantità totale di gas serra emessi da un'attività — un'auto nel traffico romano produce 120-150 g di CO₂ per km, una bici zero | bici-vs-auto-roma |
| CDC | Change Data Capture — tecnica per intercettare e propagare le modifiche ai dati in tempo reale, spesso basata sulla lettura dei log delle transazioni | binary-log-mysql |
| Compliance | Conformità alle normative, regolamenti e standard applicabili — nel contesto AI include GDPR, regolamenti di settore e policy interne | ai-manager-project-management |
| Code Review | Pratica di revisione del codice da parte di un collega prima del merge, per catturare bug e condividere conoscenza nel team | ai-github-project-management |
| COALESCE | Funzione SQL che restituisce il primo valore non NULL da una lista di espressioni. Spesso usata come workaround per le gerarchie incomplete, ma non risolve il problema strutturale nel modello | ragged-hierarchies |
| Dead Tuple | Riga obsoleta in una tabella PostgreSQL, marcata come non più visibile dopo un UPDATE o DELETE ma non ancora rimossa fisicamente | vacuum-autovacuum-postgresql |
| Daily Standup | Riunione quotidiana di massimo 15 minuti in cui ogni membro del team risponde a tre domande: cosa ho fatto ieri, cosa farò oggi, cosa mi blocca | standup-meeting-15-minuti |
| Data Governance | Insieme di politiche, processi e standard che garantiscono la qualità, la sicurezza e la conformità dei dati in un'organizzazione | ai-manager-project-management |
| Data Guard | Tecnologia Oracle per la replica in tempo reale di un database su uno o più server standby, garantendo alta disponibilità e disaster recovery | oracle-data-guard, oracle-cloud-migration |
| DEFAULT PRIVILEGES | Meccanismo PostgreSQL che definisce automaticamente i privilegi da assegnare a tutti gli oggetti futuri creati in uno schema | postgresql_roles_and_users |
| Direttiva 2011/7/UE | Direttiva europea sui ritardi di pagamento che fissa il termine standard a 30 giorni, il massimo a 60, con interessi di mora automatici | pagamenti-60-90-120-giorni |
| Drill-down | Navigazione nei report dal livello aggregato al livello di dettaglio. Richiede una gerarchia completa e bilanciata per funzionare correttamente | ragged-hierarchies, fatto-grana-sbagliata |
| Data Warehouse | Sistema centralizzato di raccolta e storicizzazione dati provenienti da fonti diverse, progettato per l'analisi e il supporto alle decisioni aziendali | 4-milioni-nessun-software |
| DSO | Days Sales Outstanding — numero medio di giorni per incassare i crediti commerciali. In Italia la media è 80 giorni, quasi tre volte la media europea | pagamenti-60-90-120-giorni |
| ETL | Extract, Transform, Load — processo di estrazione, trasformazione e caricamento dati dai sistemi sorgente al data warehouse | scd-tipo-2, ragged-hierarchies, 4-milioni-nessun-software, fatto-grana-sbagliata, partitioning-dwh |
| Exchange Partition | Operazione DDL Oracle che scambia istantaneamente i segmenti dati tra una tabella non partizionata e una partizione, senza spostare fisicamente i dati | partitioning-dwh |
| Float Finanziario | Liquidità a costo zero generata dalla differenza tra i tempi di incasso dai clienti e i tempi di pagamento ai fornitori | pagamenti-60-90-120-giorni |
| FLUSH PRIVILEGES | Comando MySQL/MariaDB che ricarica le tabelle dei grant in memoria, rendendo effettive le modifiche manuali ai privilegi | mysql-users-and-hosts |
| Execution Plan | Sequenza di operazioni (scan, join, sort) che il database sceglie per risolvere una query SQL. Si visualizza con EXPLAIN e EXPLAIN ANALYZE | explain-analyze-postgresql, like-optimization-postgresql, pg-stat-statements |
| Facilitatore | Persona responsabile di guidare una riunione mantenendo il focus, rispettando il timebox e garantendo che tutti abbiano voce | standup-meeting-15-minuti |
| Fact table | Tabella centrale dello star schema che contiene le misure numeriche (importi, quantità, conteggi) e le chiavi esterne verso le tabelle dimensionali | scd-tipo-2, fatto-grana-sbagliata, partitioning-dwh |
| Least Privilege | Principio di sicurezza che prevede l'assegnazione a ogni utente solo dei permessi strettamente necessari per svolgere la propria funzione | mysql-users-and-hosts, oracle-roles-privileges, postgresql_roles_and_users |
| Lift-and-Shift | Strategia di migrazione che sposta un sistema da un ambiente a un altro senza modificarne l'architettura, il codice o la configurazione | tecnica-si-e-yes-and |
| Local Index | Indice Oracle partizionato con la stessa chiave della tabella, dove ogni partizione della tabella ha la sua partizione di indice corrispondente | oracle-partitioning, partitioning-dwh |
| GRANT | Comando SQL per assegnare privilegi specifici a un utente o ruolo su database, tabelle o colonne | mysql-users-and-hosts, postgresql_roles_and_users |
| GIN Index | Generalized Inverted Index — tipo di indice PostgreSQL ottimizzato per ricerche full-text, pattern matching con trigrammi e query su array e JSONB | like-optimization-postgresql |
| Grain | Livello di dettaglio (granularità) di una fact table nel data warehouse. Determina cosa rappresenta ciascuna riga e quali domande il modello dimensionale può soddisfare | fatto-grana-sbagliata |
| Full Table Scan | Operazione di lettura in cui il database legge tutti i blocchi di una tabella senza utilizzare indici. In Oracle si manifesta come wait event `db file scattered read` | oracle-awr-ash, partitioning-dwh |
| GTID | Global Transaction Identifier — identificativo univoco assegnato a ogni transazione in MySQL, che semplifica la gestione della replica e il tracking delle transazioni tra master e slave | mysqldump-mysqlpump-mydumper |
| Hash Join | Strategia di join che costruisce una hash table dalla tabella più piccola e poi scansiona la più grande cercando corrispondenze con lookup O(1). Efficiente su grandi volumi senza indici | explain-analyze-postgresql |
| Hot Desk | Modello di organizzazione degli spazi ufficio in cui le postazioni non sono assegnate: chi viene in ufficio occupa una scrivania libera | smartworking-consulenza-it |
| Huge Pages | Pagine di memoria da 2 MB che riducono drasticamente la pressione sulla MMU e sul TLB, migliorando le performance di Oracle su Linux | oracle-linux-kernel |
| Issue Tracker | Sistema di tracciamento integrato per bug, richieste evolutive e task di progetto, su GitHub integrato direttamente nel repository | ai-github-project-management |
| Interessi di Mora | Interessi automatici previsti dalla legge (tasso BCE + 8%) che maturano su ogni fattura pagata in ritardo | pagamenti-60-90-120-giorni |
| INTO OUTFILE | Clausola SQL di MySQL per esportare il risultato di una query direttamente in un file sul filesystem del server, soggetta alle restrizioni di secure-file-priv | mysql-multi-istanza-secure-file-priv |
| I/O Scheduler | Componente del kernel Linux che decide l'ordine in cui le richieste di I/O vengono inviate al disco, con impatto diretto sulle performance del database | oracle-linux-kernel |
| IST | Incremental State Transfer — meccanismo di Galera Cluster per trasferire solo le transazioni mancanti a un nodo che rientra nel cluster | galera-cluster-3-nodi |
| Kimball | Ralph Kimball — metodologia di progettazione data warehouse basata su dimensional modeling, star schema e processi ETL bottom-up. Riferimento standard per la classificazione delle SCD | scd-tipo-2 |
| Knowledge Transfer | Processo di trasferimento della conoscenza tra persone, team o sistemi, critico nei progetti IT dove la perdita di know-how compromette la continuità | ai-manager-project-management |
| KPI | Key Performance Indicator — metrica misurabile che valuta l'efficacia di un'attività rispetto a un obiettivo definito | smartworking-consulenza-it |
| MERGE | Istruzione SQL che combina INSERT e UPDATE in un'unica operazione: se il record esiste lo aggiorna, se non esiste lo inserisce. In Oracle anche nota come "upsert" | scd-tipo-2 |
| Mobilità Sostenibile | Approccio ai trasporti urbani che privilegia mezzi a basso impatto ambientale, riducendo emissioni, traffico e costi | bici-vs-auto-roma |
| MVCC | Multi-Version Concurrency Control — modello di concorrenza di PostgreSQL che mantiene più versioni delle righe per garantire isolamento transazionale | vacuum-autovacuum-postgresql |
| mydumper | Tool open source di backup logico per MySQL/MariaDB con parallelismo reale a livello di chunk. Divide le tabelle grandi in pezzi e li esporta con thread multipli, con restore parallelo tramite myloader | mysqldump-mysqlpump-mydumper |
| mysqlbinlog | Utility da riga di comando di MySQL per leggere, filtrare e riapplicare il contenuto dei file binary log. Indispensabile per il point-in-time recovery e il debug della replica | binary-log-mysql |
| mysqldump | Utility di backup logico inclusa in ogni installazione MySQL. Produce un file SQL sequenziale con tutte le istruzioni per ricreare schema e dati. Single-threaded, affidabile ma lenta su database grandi | mysqldump-mysqlpump-mydumper |
| mysqlpump | Evoluzione di mysqldump introdotta in MySQL 5.7, con supporto per il parallelismo a livello di tabella e compressione nativa. Deprecato da Oracle in MySQL 8.0.34 per problemi di consistenza | mysqldump-mysqlpump-mydumper |
| Nested Loop | Strategia di join che per ogni riga della tabella esterna cerca le corrispondenze nella tabella interna. Ideale per poche righe, disastrosa su grandi volumi | explain-analyze-postgresql |
| Object Privilege | Privilegio Oracle che autorizza operazioni su un oggetto specifico del database come SELECT, INSERT o EXECUTE su una tabella, vista o procedura | oracle-roles-privileges |
| Pull Request | Meccanismo di proposta e revisione delle modifiche al codice su GitHub, con code review e approvazione prima del merge nel branch principale | ai-github-project-management |
| Parking Lot | Lista visibile di argomenti emersi durante una riunione che meritano approfondimento ma vengono rinviati a dopo per rispettare il timebox | standup-meeting-15-minuti |
| Pedalata Assistita | Sistema di propulsione elettrica che amplifica la forza della pedalata, eliminando il problema delle salite e del sudore nei tragitti urbani | bici-vs-auto-roma |
| Pendolarismo | Spostamento quotidiano casa-lavoro e ritorno, che nelle grandi città può assorbire 2-4 ore al giorno e centinaia di euro al mese | smartworking-consulenza-it, bici-vs-auto-roma |
| Presenteismo | Cultura organizzativa che equipara la presenza fisica in ufficio alla produttività, indipendentemente dai risultati prodotti | smartworking-consulenza-it |
| Partita IVA | Regime fiscale italiano per lavoratori autonomi e liberi professionisti, che nella consulenza IT implica l'assunzione diretta del rischio di credito | pagamenti-60-90-120-giorni |
| pg_stat_statements | Estensione PostgreSQL che raccoglie statistiche di esecuzione per tutte le query SQL: tempi, conteggi, righe restituite e blocchi letti. Strumento fondamentale per la diagnostica delle performance | pg-stat-statements |
| Partition Pruning | Meccanismo automatico di Oracle che esclude le partizioni non rilevanti durante l'esecuzione di una query, leggendo solo quelle corrispondenti al predicato | oracle-partitioning, partitioning-dwh |
| ROI | Return on Investment — rapporto tra il beneficio ottenuto e il costo sostenuto per un investimento | ai-manager-project-management |
| ROLE (PostgreSQL) | Entità fondamentale di PostgreSQL che unifica il concetto di utente e gruppo di permessi: con LOGIN è un utente, senza LOGIN è un contenitore di privilegi | postgresql_roles_and_users |
| pg_trgm | Estensione PostgreSQL che fornisce funzioni e operatori per la ricerca di similarità basata su trigrammi, abilitando l'uso di indici GIN per LIKE con wildcard | like-optimization-postgresql |
| NOLOGGING | Modalità Oracle che sopprime la generazione di redo log durante operazioni bulk, velocizzando le operazioni ma richiedendo un backup RMAN immediato | oracle-partitioning |
| OCI | Oracle Cloud Infrastructure — piattaforma cloud di Oracle con vantaggi significativi di licensing per i database Oracle grazie al programma BYOL e al rapporto 1:1 delle OCPU | oracle-cloud-migration |
| Outsourcing | Esternalizzazione di attività o progetti IT a fornitori esterni, con rischi significativi di perdita di know-how e vendor lock-in se non gestita correttamente | 4-milioni-nessun-software |
| OLAP | Online Analytical Processing — elaborazione orientata all'analisi multidimensionale dei dati, tipica dei data warehouse. Contrapposta all'OLTP dei sistemi transazionali | ragged-hierarchies |
| PITR | Point-in-Time Recovery — tecnica di ripristino che combina backup e binary log per riportare un database a un qualsiasi momento nel tempo | binary-log-mysql, mysqldump-mysqlpump-mydumper |
| Quorum | Meccanismo di consenso basato sulla maggioranza dei nodi, usato nei cluster database per prevenire lo split-brain e garantire la consistenza dei dati | galera-cluster-3-nodi |
| RAC | Real Application Clusters — tecnologia Oracle che permette a più istanze di accedere contemporaneamente allo stesso database, garantendo alta disponibilità e scalabilità orizzontale | oracle-cloud-migration |
| Ragged hierarchy | Gerarchia in cui non tutti i rami raggiungono la stessa profondità: alcuni livelli intermedi sono assenti. Tipica nelle anagrafiche clienti e strutture organizzative | ragged-hierarchies |
| Range Partitioning | Strategia di partizionamento che divide una tabella in segmenti basati su intervalli di valori di una colonna (tipicamente una data). Ogni partizione contiene le righe il cui valore cade nell'intervallo definito | partitioning-dwh, oracle-partitioning |
| Redo Log | File di log in cui Oracle registra ogni modifica ai dati prima di scriverla nei datafile, garantendo il recovery e la replica Data Guard | oracle-data-guard |
| REVOKE | Comando SQL per rimuovere privilegi o ruoli precedentemente assegnati a un utente o ruolo, complementare al comando GRANT | oracle-roles-privileges |
| Relay log | File di log intermedio sullo slave MySQL che riceve gli eventi dal binary log del master prima che vengano eseguiti localmente | binary-log-mysql |
| RMAN | Recovery Manager — strumento nativo Oracle per backup, restore e recovery del database, inclusa la creazione di database standby per Data Guard | oracle-data-guard |
| RPO | Recovery Point Objective — la quantità massima di dati che un'organizzazione può permettersi di perdere in caso di disastro, misurata in tempo | oracle-data-guard |
| RTO | Recovery Time Objective — il tempo massimo accettabile per ripristinare un servizio dopo un guasto o un disastro | oracle-data-guard |
| Schema | Namespace logico all'interno di un database che raggruppa tabelle, viste, funzioni e altri oggetti, permettendo organizzazione e separazione dei permessi | postgresql_roles_and_users |
| Scrum | Framework agile per la gestione di progetti che organizza il lavoro in sprint a durata fissa, con ruoli definiti e cerimonie strutturate | standup-meeting-15-minuti |
| Scope | Perimetro di un progetto che definisce cosa è incluso e cosa è escluso: funzionalità, deliverable, vincoli e confini concordati con gli stakeholder | tecnica-si-e-yes-and |
| shared_buffers | Area di memoria condivisa di PostgreSQL che funge da cache per i blocchi dati letti dal disco. Il parametro più importante per il tuning della memoria, con un default di 128 MB quasi sempre inadeguato | pg-stat-statements |
| SGA | System Global Area — area di memoria condivisa di Oracle Database che contiene buffer cache, shared pool, redo log buffer e altre strutture critiche | oracle-linux-kernel |
| SCAN Listener | Single Client Access Name — componente Oracle RAC che fornisce un unico punto di accesso al cluster, distribuendo automaticamente le connessioni tra i nodi disponibili | oracle-cloud-migration |
| Smart Working | Modello di lavoro flessibile che combina lavoro da remoto e presenza in ufficio, basato su obiettivi misurabili invece che su presenza fisica | smartworking-consulenza-it |
| Scope Creep | Espansione incontrollata dei requisiti di progetto oltre il perimetro iniziale, che porta a ritardi, aumento dei costi e spesso al fallimento del progetto | 4-milioni-nessun-software |
| Stakeholder | Persona o gruppo con un interesse diretto nel risultato di un progetto: committente, utente finale, sponsor, team tecnico | tecnica-si-e-yes-and |
| SCD | Slowly Changing Dimension — tecnica di data warehouse per tracciare le variazioni nel tempo dei dati nelle tabelle dimensionali | scd-tipo-2 |
| secure-file-priv | Direttiva di sicurezza MySQL che limita le directory in cui il server può leggere e scrivere file, proteggendo il filesystem da operazioni non autorizzate | mysql-multi-istanza-secure-file-priv |
| Split-brain | Condizione critica in un cluster database dove due o più parti operano indipendentemente, accettando scritture divergenti sugli stessi dati | galera-cluster-3-nodi |
| SQL Injection | Tecnica di attacco che inserisce codice SQL malevolo negli input di un'applicazione per manipolare le query eseguite dal database | mysql-multi-istanza-secure-file-priv |
| SST | State Snapshot Transfer — meccanismo di Galera Cluster per trasferire una copia completa dei dati a un nodo che si unisce al cluster | galera-cluster-3-nodi |
| Sequential Scan | Operazione di lettura in cui PostgreSQL legge tutti i blocchi di una tabella senza utilizzare indici, efficiente su tabelle piccole ma problematica su tabelle grandi | pg-stat-statements |
| Self-parenting | Tecnica di bilanciamento delle gerarchie sbilanciate: chi non ha un padre diventa padre di sé stesso, eliminando i NULL dalla dimensione | ragged-hierarchies |
| Snapshot (Oracle) | Istantanea delle statistiche di performance catturata periodicamente da AWR (di default ogni 60 minuti) e usata per generare report diagnostici comparativi | oracle-awr-ash |
| Star schema | Modello di dati tipico del data warehouse: una fact table al centro collegata a più tabelle dimensionali tramite chiavi esterne. Semplifica le query analitiche e ottimizza le performance | scd-tipo-2, fatto-grana-sbagliata |
| Tablespace | Unità logica di storage in Oracle che raggruppa uno o più datafile fisici, usata per organizzare e gestire lo spazio su disco per tabelle, indici e partizioni | oracle-partitioning |
| THP | Transparent Huge Pages — funzione del kernel Linux che promuove automaticamente le pagine normali a pagine grandi, causa latenze imprevedibili e deve essere disabilitata per Oracle | oracle-linux-kernel |
| Timeboxing | Tecnica di gestione del tempo che assegna un intervallo fisso e non negoziabile a un'attività, forzando la conclusione entro il limite stabilito | tecnica-si-e-yes-and, standup-meeting-15-minuti |
| Transport Lag | Ritardo nella trasmissione dei redo log dal database primary allo standby in una configurazione Data Guard, indicatore critico della salute della replica | oracle-cloud-migration |
| Swappiness | Parametro kernel Linux (vm.swappiness) che controlla la propensione del sistema a spostare pagine di memoria nello swap | oracle-linux-kernel |
| Switchover | Operazione pianificata di Data Guard che inverte i ruoli tra primary e standby senza perdita di dati, reversibile e controllata | oracle-cloud-migration |
| System Privilege | Privilegio Oracle che autorizza operazioni globali sul database come CREATE TABLE, CREATE SESSION o ALTER SYSTEM, indipendenti da qualsiasi oggetto specifico | oracle-roles-privileges |
| systemd | Sistema di init e gestore dei servizi su Linux, usato per gestire istanze multiple di MySQL/MariaDB sullo stesso server tramite unit file separati | mysql-multi-istanza-secure-file-priv |
| Version Control | Sistema che traccia ogni modifica al codice sorgente, permettendo cronologia, annullamento e collaborazione. Git è lo standard attuale | ai-github-project-management |
| VACUUM | Comando PostgreSQL che recupera lo spazio occupato dai dead tuples, rendendolo riutilizzabile per nuovi inserimenti | vacuum-autovacuum-postgresql |
| Vendor Lock-in | Dipendenza strutturale da un fornitore esterno che rende difficile o costoso cambiare provider, spesso causata dalla perdita di know-how e dalla proprietà del codice | 4-milioni-nessun-software |
| Unified Audit | Sistema di audit centralizzato introdotto in Oracle 12c che unifica tutti i tipi di audit in un'unica infrastruttura, sostituendo il vecchio audit tradizionale | oracle-roles-privileges |
| Unix Socket | Meccanismo di comunicazione inter-processo locale su sistemi Unix/Linux, usato da MySQL per connessioni più veloci rispetto a TCP quando client e server sono sullo stesso host | mysql-multi-istanza-secure-file-priv |
| Wait Event | Evento di attesa registrato da Oracle ogni volta che una sessione non può procedere e deve attendere una risorsa (I/O, lock, CPU, rete). L'analisi dei wait event è la base della metodologia diagnostica Oracle | oracle-awr-ash |
| WSREP | Write Set Replication — API e protocollo di replica sincrona usato da Galera Cluster per mantenere i nodi del cluster allineati in tempo reale | galera-cluster-3-nodi |
| Yes-And | Tecnica di comunicazione nata nel teatro di improvvisazione che sostituisce il "No, però..." con "Sì, e...", trasformando le discussioni in costruzione collaborativa | tecnica-si-e-yes-and |
| ZDM | Zero Downtime Migration — strumento Oracle per automatizzare le migrazioni verso OCI combinando Data Guard e Data Pump sotto un layer di orchestrazione | oracle-cloud-migration |

---

**Ultimo aggiornamento**: 2026-03-23
**Totale termini**: 131
**Totale articoli con glossario**: 28
