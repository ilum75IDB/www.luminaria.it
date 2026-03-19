---
title: "Wrong grain: when the fact table can't answer the right questions"
description: "A fact table built on monthly invoice totals looked perfect. Then the business asked for product-level, line-level, customer-level detail. And the data warehouse went silent."
date: "2025-10-21T10:00:00+01:00"
draft: false
translationKey: "fatto_grana_sbagliata"
tags: ["data-warehouse", "fact-table", "granularity", "grain", "dimensional-modeling", "kimball"]
categories: ["Data Warehouse"]
image: "fatto-grana-sbagliata.cover.jpg"
---

The meeting had started well. The sales director of an industrial distribution company — around sixty million in revenue, three thousand active customers, a catalog of twelve thousand SKUs — had opened the new data warehouse presentation with a smile. The numbers matched, the dashboards were polished, the monthly totals by agent and territory reconciled with accounting.

Then someone asked the wrong question. Or rather, the right one.

*"Can I see what customer Bianchi purchased in March, line by line, product by product?"*

Silence.

The BI manager looked at me. I looked at the screen. The screen showed a {{< glossary term="fact-table" >}}fact table{{< /glossary >}} with one row per customer per month: total invoiced amount, total quantity, invoice count. No detail. No invoice line. No product.

That fact table answered one question only: *how much did each customer invoice in a given month?* Everything else — by product, by product family, by individual invoice — was out of reach.

## 🔍 The grain: the decision that determines everything

In {{< glossary term="star-schema" >}}dimensional modeling{{< /glossary >}}, the **{{< glossary term="grain" >}}grain{{< /glossary >}}** of the fact table is the first decision you make. Not the second, not one among many: the first. Kimball repeats it in every chapter, and he's right.

The grain answers the question: *what does a single row in the fact table represent?*

In the project I described, the original designer had chosen a monthly-customer grain: one row = one customer in one month. The reasons seemed sound: the source system exported a monthly summary, loading was fast, tables were small, queries were simple.

But the grain determines which questions the data warehouse can answer. If the grain is a monthly summary per customer, you can't go below that level. You can't {{< glossary term="drill-down" >}}drill down{{< /glossary >}} by product. You can't tell whether customer Bianchi bought the same item ten times or ten different items. You can't compare margins by product family.

You have a total. Period.

## 📊 The problem in numbers

The original fact table had this structure:

```sql
CREATE TABLE fact_monthly_revenue (
    sk_customer       INT NOT NULL,
    sk_time           INT NOT NULL,  -- month (YYYYMM)
    sk_agent          INT NOT NULL,
    sk_territory      INT NOT NULL,
    total_amount      DECIMAL(15,2),
    total_quantity    INT,
    num_invoices      INT,
    num_lines         INT,
    FOREIGN KEY (sk_customer) REFERENCES dim_customer(sk_customer),
    FOREIGN KEY (sk_time)     REFERENCES dim_time(sk_time)
);
```

Rows per year: about 180,000 (3,000 customers × 12 months × some variation). Small, fast, easy to load. The {{< glossary term="etl" >}}ETL{{< /glossary >}} ran in under five minutes.

The problem? The {{< glossary term="additive-measure" >}}additive measures{{< /glossary >}} were already aggregated. `total_amount` was the sum of all invoice lines for the month. No way to trace back to the composition. Like having a receipt total without knowing what you bought.

## 🏗️ The restructuring: going down to the invoice line

There was only one solution: change the grain. Bring the fact table down to the lowest level available in the source system — the individual invoice line.

```sql
CREATE TABLE fact_revenue_line (
    sk_invoice_line   INT PRIMARY KEY,
    sk_invoice        INT NOT NULL,
    sk_customer       INT NOT NULL,
    sk_product        INT NOT NULL,
    sk_time           INT NOT NULL,  -- day (YYYYMMDD)
    sk_agent          INT NOT NULL,
    sk_territory      INT NOT NULL,
    sk_family         INT NOT NULL,
    quantity          INT,
    unit_price        DECIMAL(12,4),
    line_amount       DECIMAL(15,2),
    discount_pct      DECIMAL(5,2),
    net_amount        DECIMAL(15,2),
    product_cost      DECIMAL(15,2),
    margin            DECIMAL(15,2),
    FOREIGN KEY (sk_customer) REFERENCES dim_customer(sk_customer),
    FOREIGN KEY (sk_product)  REFERENCES dim_product(sk_product),
    FOREIGN KEY (sk_time)     REFERENCES dim_time(sk_time)
);
```

Rows per year: about 2.4 million (3,000 customers × ~800 lines/year on average). An order of magnitude more. But every row carried the full detail: which product, which invoice, which price, which discount, which margin.

## ⚡ The ETL impact

The grain change had a cascading effect on the ETL that nobody had anticipated — or rather, that whoever chose the aggregated grain had avoided facing.

**New dimensions required:**

| Dimension          | Cardinality | Notes                                |
|---------------------|-------------|--------------------------------------|
| `dim_product`       | ~12,000     | Didn't exist before: wasn't needed   |
| `dim_family`        | ~180        | 3-level product hierarchy            |
| `dim_invoice`       | ~45,000/yr  | Invoice header with master data      |

**New loading window:**

| Phase               | Before  | After     |
|---------------------|---------|-----------|
| Extraction          | 40 sec  | 3 min     |
| Transformation      | 1 min   | 8 min     |
| Fact loading        | 30 sec  | 4 min     |
| **Total**           | **~2 min** | **~15 min** |

Fifteen minutes versus two. An acceptable price for a data warehouse that now answered real questions.

## 🔬 Queries that were previously impossible

With the new grain, the queries the business wanted became trivial:

**Customer purchases by product:**

```sql
SELECT
    c.company_name,
    p.product_code,
    p.description,
    SUM(f.quantity)       AS units,
    SUM(f.net_amount)     AS net_revenue,
    SUM(f.margin)         AS total_margin
FROM fact_revenue_line f
JOIN dim_customer c ON f.sk_customer = c.sk_customer
JOIN dim_product  p ON f.sk_product  = p.sk_product
JOIN dim_time     t ON f.sk_time     = t.sk_time
WHERE c.company_name = 'Bianchi Srl'
  AND t.year = 2024
  AND t.month = 3
GROUP BY c.company_name, p.product_code, p.description
ORDER BY net_revenue DESC;
```

**Top 10 products by margin in a quarter:**

```sql
SELECT
    p.product_code,
    p.description,
    fm.family_desc,
    SUM(f.net_amount)  AS revenue,
    SUM(f.margin)      AS margin,
    ROUND(SUM(f.margin) / NULLIF(SUM(f.net_amount), 0) * 100, 1) AS margin_pct
FROM fact_revenue_line f
JOIN dim_product  p  ON f.sk_product = p.sk_product
JOIN dim_family   fm ON f.sk_family  = fm.sk_family
JOIN dim_time     t  ON f.sk_time    = t.sk_time
WHERE t.year = 2024
  AND t.quarter = 1
GROUP BY p.product_code, p.description, fm.family_desc
ORDER BY margin DESC
LIMIT 10;
```

**Agent comparison: average revenue per invoice line:**

```sql
SELECT
    a.agent_name,
    COUNT(*)                       AS num_lines,
    SUM(f.net_amount)              AS total_revenue,
    ROUND(AVG(f.net_amount), 2)    AS avg_per_line
FROM fact_revenue_line f
JOIN dim_agent a ON f.sk_agent = a.sk_agent
JOIN dim_time  t ON f.sk_time  = t.sk_time
WHERE t.year = 2024
GROUP BY a.agent_name
ORDER BY total_revenue DESC;
```

None of these queries was possible with the monthly-customer grain. None. It wasn't a matter of tuning or indexing — it was a structural problem, written in the model's DNA.

## 📋 The Kimball rule we had ignored

Ralph Kimball puts it plainly: *"always model at the finest level of detail available in the source system."*

This isn't a suggestion. It's not one option among many. It's the founding principle of dimensional modeling. And the reason is simple: you can always aggregate from detail to total, but you can never disaggregate a total back into its detail.

Aggregation is an irreversible operation. Like mixing colors: from red and yellow you can get orange, but from orange you can never go back to the original colors.

In our project, the choice of an aggregated grain was driven by design laziness, not by a technical constraint. The source system had line-level detail — nobody had wanted to deal with the complexity of modeling it, managing the additional dimensions, extending the ETL window.

The result? A data warehouse that had to be rebuilt from scratch six months after go-live.

## 🎯 When an aggregated grain makes sense

A fine grain isn't always the only answer. There are legitimate cases for aggregated fact tables:

- **Aggregate fact tables** alongside the detail table, to speed up the most frequent queries
- **Periodic snapshots** where the business genuinely thinks in periods (monthly account balance, end-of-week inventory)
- **Source constraints** when the upstream system doesn't expose detail and there's no way to get it

But the rule is: start from detail, then aggregate. Never the other way around. Aggregate fact tables are an optimization, not a substitute for fine grain.

In our case, after the restructuring, we also created a materialized view with the monthly summary per customer — the same structure as before — for executive dashboards that didn't need the detail. The best of both worlds, without sacrificing anything.

## What I learned

That project taught me something I carry into every engagement since: the first half-hour of data warehouse design, the one where you decide the grain, is worth more than all the optimizations that come later. A flawless ETL, perfectly tuned indexes, powerful hardware — none of it compensates for the wrong grain.

If your fact table can't answer the business's questions, it's not the queries' fault. It's the model's fault. And the model is decided at the grain.

------------------------------------------------------------------------

## Glossary

**[Grain](/en/glossary/grain/)** — The level of detail of a fact table in a data warehouse. Determines what each row represents and which questions the model can answer. It's the first decision in dimensional design.

**[Fact table](/en/glossary/fact-table/)** — The central table in a star schema containing numeric measures (amounts, quantities, margins) and foreign keys to dimensions. Its grain determines the level of analysis possible.

**[Additive Measure](/en/glossary/additive-measure/)** — A numeric measure that can be summed across all dimensions (e.g., amount, quantity). Once aggregated to a higher level, the original detail is irreversibly lost.

**[Drill-down](/en/glossary/drill-down/)** — Navigation in reports from an aggregated level to detail, along a hierarchy. Only possible if the fact table contains data at a sufficient grain level.

**[Star Schema](/en/glossary/star-schema/)** — A data model with a central fact table and linked dimension tables. The most common structure in data warehouses for simple, fast analytical queries.

**[ETL](/en/glossary/etl/)** — Extract, Transform, Load: the process of extracting, transforming, and loading data into a data warehouse. A grain change directly impacts ETL duration and complexity.
