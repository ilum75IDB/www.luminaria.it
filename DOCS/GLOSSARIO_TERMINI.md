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
| AWR | Automatic Workload Repository — strumento diagnostico integrato in Oracle Database per la raccolta e l'analisi delle statistiche di performance | oracle-awr-ash |
| default_statistics_target | Parametro PostgreSQL che definisce quanti campioni raccogliere per colonna durante l'ANALYZE. Il default è 100; su colonne con distribuzione asimmetrica conviene alzarlo a 500-1000 | explain-analyze-postgresql |
| Chiave surrogata | Identificativo numerico generato dal data warehouse, distinto dalla chiave naturale del sistema sorgente. Nella SCD Tipo 2 è indispensabile perché lo stesso record può avere più versioni | scd-tipo-2 |
| ETL | Extract, Transform, Load — processo di estrazione, trasformazione e caricamento dati dai sistemi sorgente al data warehouse | scd-tipo-2, ragged-hierarchies |
| Execution Plan | Sequenza di operazioni (scan, join, sort) che il database sceglie per risolvere una query SQL. Si visualizza con EXPLAIN e EXPLAIN ANALYZE | explain-analyze-postgresql |
| Fact table | Tabella centrale dello star schema che contiene le misure numeriche (importi, quantità, conteggi) e le chiavi esterne verso le tabelle dimensionali | scd-tipo-2 |
| Hash Join | Strategia di join che costruisce una hash table dalla tabella più piccola e poi scansiona la più grande cercando corrispondenze con lookup O(1). Efficiente su grandi volumi senza indici | explain-analyze-postgresql |
| Kimball | Ralph Kimball — metodologia di progettazione data warehouse basata su dimensional modeling, star schema e processi ETL bottom-up. Riferimento standard per la classificazione delle SCD | scd-tipo-2 |
| MERGE | Istruzione SQL che combina INSERT e UPDATE in un'unica operazione: se il record esiste lo aggiorna, se non esiste lo inserisce. In Oracle anche nota come "upsert" | scd-tipo-2 |
| Nested Loop | Strategia di join che per ogni riga della tabella esterna cerca le corrispondenze nella tabella interna. Ideale per poche righe, disastrosa su grandi volumi | explain-analyze-postgresql |
| SCD | Slowly Changing Dimension — tecnica di data warehouse per tracciare le variazioni nel tempo dei dati nelle tabelle dimensionali | scd-tipo-2 |
| Star schema | Modello di dati tipico del data warehouse: una fact table al centro collegata a più tabelle dimensionali tramite chiavi esterne. Semplifica le query analitiche e ottimizza le performance | scd-tipo-2 |

---

**Ultimo aggiornamento**: 2026-03-18
**Totale termini**: 13
**Totale articoli con glossario**: 2
