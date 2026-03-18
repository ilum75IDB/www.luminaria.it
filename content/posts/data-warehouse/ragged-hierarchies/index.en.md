---
title: "Ragged hierarchies: when the client has no parent and the group has no grandparent"
description: "A client with a three-level hierarchy — Top Group, Group, Client — where not all branches are complete. How I balanced a ragged hierarchy with self-parenting: whoever has no parent becomes their own parent."
date: "2026-01-20T10:00:00+01:00"
draft: false
translationKey: "ragged_hierarchies"
tags: ["hierarchies", "dimensional-modeling", "etl", "oracle", "olap", "reporting"]
categories: ["data-warehouse"]
image: "ragged-hierarchies.cover.jpg"
---

Three levels. Top Group, Group, Client. It looks like a trivial structure — the kind of hierarchy you draw on a whiteboard in five minutes and that any BI tool should handle without issues.

Then you discover that not all clients belong to a group. And that not all groups belong to a top group. And that the aggregation reports the business asks for — revenue by top group, client count by group, {{< glossary term="drill-down" >}}drill-down{{< /glossary >}} from the top to the leaf — produce wrong or incomplete results because the hierarchy has holes.

In technical jargon it is called a **{{< glossary term="ragged-hierarchy" >}}ragged hierarchy{{< /glossary >}}**: a hierarchy where not all branches reach the same depth. In the real world it is called "the problem nobody sees until they open the report and the numbers do not add up."

---

## The client and the original model

The project was a data warehouse for a company in the energy sector — gas distribution and related services. The source system managed a client master with a hierarchical structure: clients could be grouped under a commercial entity (the **Group**), and groups could in turn belong to a higher entity (the **Top Group**).

The model in the source was a single table with hierarchical references:

``` sql
CREATE TABLE stg_clienti (
    client_id       NUMBER(10)    NOT NULL,
    client_name     VARCHAR2(100) NOT NULL,
    group_id        NUMBER(10),
    group_name      VARCHAR2(100),
    top_group_id    NUMBER(10),
    top_group_name  VARCHAR2(100),
    revenue         NUMBER(15,2),
    region          VARCHAR2(50),
    CONSTRAINT pk_stg_clienti PRIMARY KEY (client_id)
);
```

Here is a data sample:

``` sql
INSERT INTO stg_clienti VALUES (1001, 'Rossi Energia Srl',     10, 'Consorzio Nord',      100, 'Holding Nazionale',  125000.00, 'Lombardia');
INSERT INTO stg_clienti VALUES (1002, 'Bianchi Gas SpA',       10, 'Consorzio Nord',      100, 'Holding Nazionale',   89000.00, 'Piemonte');
INSERT INTO stg_clienti VALUES (1003, 'Verdi Distribuzione',   20, 'Gruppo Centro',       100, 'Holding Nazionale',   67000.00, 'Toscana');
INSERT INTO stg_clienti VALUES (1004, 'Neri Servizi',          20, 'Gruppo Centro',       NULL, NULL,                  45000.00, 'Lazio');
INSERT INTO stg_clienti VALUES (1005, 'Gialli Utilities',      NULL, NULL,                NULL, NULL,                  38000.00, 'Sicilia');
INSERT INTO stg_clienti VALUES (1006, 'Blu Energia',           NULL, NULL,                NULL, NULL,                  52000.00, 'Sardegna');
INSERT INTO stg_clienti VALUES (1007, 'Viola Gas Srl',         30, 'Rete Sud',            NULL, NULL,                  71000.00, 'Campania');
INSERT INTO stg_clienti VALUES (1008, 'Arancio Distribuzione', 30, 'Rete Sud',            NULL, NULL,                  33000.00, 'Calabria');
```

Look at the data carefully. There are four different situations:

- **Client 1001, 1002, 1003**: complete hierarchy — Client → Group → Top Group
- **Client 1004**: has a Group but the Group has no Top Group
- **Client 1005, 1006**: no Group, no Top Group — direct clients
- **Client 1007, 1008**: have a Group (Rete Sud) but the Group has no Top Group

This is a ragged hierarchy. Three levels on paper, but in reality the branches have different depths.

---

## The problem: the reports do not add up

The business asked for a simple report: revenue aggregated by Top Group, with drill-down capability by Group and then by Client. A reasonable request — the kind of thing you expect from any DWH.

The most natural query:

``` sql
SELECT top_group_name,
       group_name,
       COUNT(*)        AS num_clients,
       SUM(revenue)    AS total_revenue
FROM   stg_clienti
GROUP BY top_group_name, group_name
ORDER BY top_group_name, group_name;
```

The result:

``` text
TOP_GROUP_NAME      GROUP_NAME        NUM_CLIENTS  TOTAL_REVENUE
------------------  ----------------  -----------  -------------
Holding Nazionale   Consorzio Nord              2      214000.00
Holding Nazionale   Gruppo Centro               1       67000.00
(null)              Gruppo Centro               1       45000.00
(null)              Rete Sud                    2      104000.00
(null)              (null)                      2       90000.00
```

Five rows. And at least three problems.

Gruppo Centro appears twice: once under "Holding Nazionale" (client 1003 which has a top group) and once under NULL (client 1004 whose top group is NULL). The same group, split across two rows, with separate totals. Anyone looking at this report will think Gruppo Centro has 67K revenue under the holding and 45K somewhere else. In reality it is a single group with 112K total.

The direct clients (Gialli Utilities and Blu Energia) end up in a row with two NULLs. Management does not know what to do with a nameless row.

The Top Group total is wrong because the NULL rows are missing. If you sum only the rows with a top group, you lose 239K in revenue — 30% of the total.

---

## The classic approach: COALESCE and prayers

The first reaction, the one I see in 90% of cases, is to add {{< glossary term="coalesce" >}}`COALESCE`{{< /glossary >}} to the query:

``` sql
SELECT COALESCE(top_group_name, group_name, client_name) AS top_group_name,
       COALESCE(group_name, client_name)                 AS group_name,
       client_name,
       revenue
FROM   stg_clienti;
```

Does it work? In a sense yes — it fills the holes. But it introduces new problems.

Client "Gialli Utilities" now appears as Top Group, Group and Client simultaneously. If the business wants to count how many Top Groups there are, the number is inflated. If they want to filter for "real" top groups, there is no way to distinguish them from clients promoted to top group by the COALESCE.

And this is the simple case, with three levels. I have seen five-level hierarchies managed with chains of nested COALESCE, multiple CASE WHEN expressions, and report logic so convoluted that nobody dared touch it anymore. Every new business request required cascading changes across all queries.

The root problem is that COALESCE is a patch applied at the presentation layer. It does not fix the structural issue: the hierarchy is incomplete and the dimensional model does not know it.

---

## The solution: self-parenting

The principle is simple: **whoever has no parent becomes their own parent**. This technique is called {{< glossary term="self-parenting" >}}self-parenting{{< /glossary >}}.

A Client without a Group? That client becomes its own Group. A Group without a Top Group? That group becomes its own Top Group. This way the hierarchy is always complete at three levels, with no holes, no NULLs.

It is not a trick. It is a standard technique in dimensional modeling, described by {{< glossary term="kimball" >}}Kimball{{< /glossary >}} and used in production for decades. The idea is that the hierarchical dimension in the DWH must be **balanced**: every record must have a valid value for every level of the hierarchy. If the source does not guarantee it, the {{< glossary term="etl" >}}ETL{{< /glossary >}} does.

### The dimensional table

``` sql
CREATE TABLE dim_client_hierarchy (
    client_key      NUMBER(10)    NOT NULL,
    client_id       NUMBER(10)    NOT NULL,
    client_name     VARCHAR2(100) NOT NULL,
    group_id        NUMBER(10)    NOT NULL,
    group_name      VARCHAR2(100) NOT NULL,
    top_group_id    NUMBER(10)    NOT NULL,
    top_group_name  VARCHAR2(100) NOT NULL,
    region          VARCHAR2(50),
    is_direct_client  CHAR(1)     DEFAULT 'N',
    is_standalone_group CHAR(1)   DEFAULT 'N',
    CONSTRAINT pk_dim_client_hier PRIMARY KEY (client_key)
);
```

Notice two things. First: no column is nullable. Group and Top Group are `NOT NULL`. Second: I added two flags — `is_direct_client` and `is_standalone_group` — that allow distinguishing artificially balanced records from those with a natural hierarchy. This is important: the business must be able to filter "real" top groups from promoted clients.

### The ETL logic

``` sql
INSERT INTO dim_client_hierarchy (
    client_key, client_id, client_name,
    group_id, group_name,
    top_group_id, top_group_name,
    region, is_direct_client, is_standalone_group
)
SELECT
    client_id AS client_key,
    client_id,
    client_name,
    -- If no group, the client becomes its own group
    COALESCE(group_id, client_id)          AS group_id,
    COALESCE(group_name, client_name)      AS group_name,
    -- If no top group, the group (or client) becomes its own top group
    COALESCE(top_group_id, group_id, client_id)       AS top_group_id,
    COALESCE(top_group_name, group_name, client_name)  AS top_group_name,
    region,
    CASE WHEN group_id IS NULL THEN 'Y' ELSE 'N' END  AS is_direct_client,
    CASE WHEN group_id IS NOT NULL AND top_group_id IS NULL
         THEN 'Y' ELSE 'N' END                        AS is_standalone_group
FROM stg_clienti;
```

Look at the COALESCE cascade in the transformation. The logic is:

- `group_id`: if the client has a group, use it; otherwise use the client itself
- `top_group_id`: if there is a top group, use it; if not but there is a group, use the group; if there is no group either, use the client

Every "missing" level is filled by the level immediately below. The result is a hierarchy that is always complete.

### The result after balancing

``` sql
SELECT client_key, client_name, group_name, top_group_name,
       is_direct_client, is_standalone_group
FROM   dim_client_hierarchy
ORDER BY top_group_id, group_id, client_id;
```

``` text
KEY   CLIENT_NAME           GROUP_NAME        TOP_GROUP_NAME      DIRECT  STANDALONE
----  --------------------  ----------------  ------------------  ------  ----------
1001  Rossi Energia Srl     Consorzio Nord    Holding Nazionale   N       N
1002  Bianchi Gas SpA       Consorzio Nord    Holding Nazionale   N       N
1003  Verdi Distribuzione   Gruppo Centro     Holding Nazionale   N       N
1004  Neri Servizi          Gruppo Centro     Gruppo Centro       N       Y
1007  Viola Gas Srl         Rete Sud          Rete Sud            N       Y
1008  Arancio Distribuzione Rete Sud          Rete Sud            N       Y
1005  Gialli Utilities      Gialli Utilities  Gialli Utilities    Y       N
1006  Blu Energia           Blu Energia       Blu Energia         Y       N
```

Eight rows, zero NULLs. Every client has a group and a top group. The flags tell the truth: Gialli and Blu are direct clients (self-parented at all levels), Gruppo Centro and Rete Sud are standalone groups (self-parented at the top group level).

---

## Reports after balancing

The same aggregation query that previously produced broken results:

``` sql
SELECT top_group_name,
       group_name,
       COUNT(*)        AS num_clients,
       SUM(f.revenue)  AS total_revenue
FROM   dim_client_hierarchy d
JOIN   stg_clienti f ON d.client_id = f.client_id
GROUP BY top_group_name, group_name
ORDER BY top_group_name, group_name;
```

``` text
TOP_GROUP_NAME      GROUP_NAME          NUM_CLIENTS  TOTAL_REVENUE
------------------  ------------------  -----------  -------------
Blu Energia         Blu Energia                   1      52000.00
Gialli Utilities    Gialli Utilities              1      38000.00
Gruppo Centro       Gruppo Centro                 1      45000.00
Holding Nazionale   Consorzio Nord                2     214000.00
Holding Nazionale   Gruppo Centro                 1      67000.00
Rete Sud            Rete Sud                      2     104000.00
```

No NULLs. Every row has an identifiable top group and group. The totals add up.

And if the business wants only "real" top groups, excluding promoted clients:

``` sql
SELECT top_group_name,
       COUNT(*)        AS num_clients,
       SUM(f.revenue)  AS total_revenue
FROM   dim_client_hierarchy d
JOIN   stg_clienti f ON d.client_id = f.client_id
WHERE  d.is_direct_client = 'N'
AND    d.is_standalone_group = 'N'
GROUP BY top_group_name
ORDER BY total_revenue DESC;
```

``` text
TOP_GROUP_NAME      NUM_CLIENTS  TOTAL_REVENUE
------------------  -----------  -------------
Holding Nazionale             3      281000.00
```

The flags make everything filterable. No conditional logic in the report, no CASE WHEN, no COALESCE. The dimensional model already contains all the information needed.

---

## The full drill-down

The real test of a balanced hierarchy is drill-down: from the highest level to the lowest, with no surprises.

``` sql
-- Level 1: total by Top Group
SELECT top_group_name,
       COUNT(DISTINCT group_id) AS num_groups,
       COUNT(*)                 AS num_clients,
       SUM(f.revenue)           AS revenue
FROM   dim_client_hierarchy d
JOIN   stg_clienti f ON d.client_id = f.client_id
GROUP BY top_group_name
ORDER BY revenue DESC;
```

``` text
TOP_GROUP_NAME      NUM_GROUPS  NUM_CLIENTS  REVENUE
------------------  ----------  -----------  ----------
Holding Nazionale            2            3   281000.00
Rete Sud                     1            2   104000.00
Blu Energia                  1            1    52000.00
Gruppo Centro                1            1    45000.00
Gialli Utilities             1            1    38000.00
```

``` sql
-- Level 2: drill-down into "Holding Nazionale"
SELECT group_name,
       COUNT(*)       AS num_clients,
       SUM(f.revenue) AS revenue
FROM   dim_client_hierarchy d
JOIN   stg_clienti f ON d.client_id = f.client_id
WHERE  d.top_group_name = 'Holding Nazionale'
GROUP BY group_name
ORDER BY revenue DESC;
```

``` text
GROUP_NAME        NUM_CLIENTS  REVENUE
----------------  -----------  ----------
Consorzio Nord              2   214000.00
Gruppo Centro               1    67000.00
```

``` sql
-- Level 3: drill-down into "Consorzio Nord"
SELECT client_name, f.revenue
FROM   dim_client_hierarchy d
JOIN   stg_clienti f ON d.client_id = f.client_id
WHERE  d.group_name = 'Consorzio Nord'
ORDER BY f.revenue DESC;
```

``` text
CLIENT_NAME          REVENUE
-------------------  ----------
Rossi Energia Srl     125000.00
Bianchi Gas SpA        89000.00
```

Three levels of drill-down, zero NULLs, zero conditional logic. The hierarchy is balanced and the numbers add up at every level.

---

## Why COALESCE in reports is not enough

Someone might object: "But COALESCE in the report does the same thing, without needing to change the model."

No. It does something similar, but with three fundamental differences.

**First: COALESCE must be repeated everywhere.** Every query, every report, every dashboard, every extract. If you have twenty reports using the hierarchy, you must remember to apply the COALESCE in all twenty. And when the twenty-first arrives, you must remember again. Self-parenting in the dimensional model is done once in the ETL and that is it.

**Second: COALESCE does not distinguish.** You cannot tell whether "Gialli Utilities" in the top_group field is a real top group or a promoted client. With flags in the dimensional model you have the information to filter. Without flags, the business is blind.

**Third: performance.** A GROUP BY with COALESCE on nullable columns is less efficient than a GROUP BY on NOT NULL columns. Oracle's optimizer handles NOT NULL constrained columns better — it can eliminate NULL checks, use indexes more aggressively, and produce simpler execution plans. On a dimensional table with millions of rows, the difference shows.

---

## When to use self-parenting (and when not to)

Self-parenting works well when:

- The hierarchy has a **fixed number of levels** (typically 2-5)
- The main use case is **aggregation and drill-down** in reports
- The model is a **data warehouse** or an {{< glossary term="olap" >}}OLAP{{< /glossary >}} cube
- Missing levels are the exception, not the rule

It does not work well when:

- The hierarchy is **recursive** with variable depth (e.g. org charts with N levels)
- You need to navigate the **graph** of relationships (e.g. social networks, supply chains)
- The model is **OLTP** and self-parenting would create ambiguity in application logic
- Hierarchy levels change frequently over time

For recursive hierarchies with variable depth, the right approach is different: bridge tables, closure tables or parent-child models with recursive CTEs. These are powerful tools but they solve a different problem.

Self-parenting solves a specific problem — fixed-level hierarchies with incomplete branches — and it solves it in the simplest way possible: balancing the structure upstream, in the model, rather than downstream, in the reports.

---

## The rule that guides me

I have designed dozens of hierarchical dimensions in twenty years of data warehousing. The rule I carry with me is always the same:

**If the report needs conditional logic to handle the hierarchy, the problem is in the model, not in the report.**

A report should do GROUP BY and JOIN. If it also has to decide how to handle missing levels, it is doing the ETL's job. And a report that does the ETL's job is a report that will break sooner or later.

Self-parenting is not elegant. It is not sophisticated. It is a solution that a freshly graduated computer scientist might find ugly. But it works, it is maintainable, and it transforms a problem that infests every single report into a problem that is solved once, in one place, and never comes back.

Sometimes the best solution is the simplest one. This is one of those times.

---

## Glossary

**[COALESCE](/en/glossary/coalesce/)** — A SQL function that returns the first non-NULL value from a list of expressions. Often used as a workaround for incomplete hierarchies in reports, but it doesn't solve the structural problem in the dimensional model.

**[Drill-down](/en/glossary/drill-down/)** — Navigation in reports from an aggregated level to a detail level (e.g. from Top Group to Group to Client). Requires a complete and balanced hierarchy to work correctly without NULLs or missing rows.

**[OLAP](/en/glossary/olap/)** — Online Analytical Processing — processing oriented to multidimensional data analysis, typical of data warehouses and analysis cubes. Contrasted with OLTP (Online Transaction Processing) used in transactional systems.

**[Ragged hierarchy](/en/glossary/ragged-hierarchy/)** — A hierarchy where not all branches reach the same depth: some intermediate levels are missing. Common in customer master data, products and organizational structures where not all entities share the same hierarchical structure.

**[Self-parenting](/en/glossary/self-parenting/)** — A technique for balancing ragged hierarchies: an entity without a parent becomes its own parent. The missing level is filled with data from the level below, eliminating NULLs from the dimension and ensuring correct drill-down behavior.
