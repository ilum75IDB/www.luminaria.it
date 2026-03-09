---
title: "Data Warehouse"
date: "2026-03-10T10:00:00+01:00"
description: "Data Warehouse architecture in practice: dimensional modeling, hierarchies, ETL and loading strategies. When data is not just meant to work, but to drive decisions."
image: "data-warehouse.cover.jpg"
layout: "list"
---
A data warehouse is not a database with bigger tables.<br>
It is a different way of thinking about data — oriented towards analysis, history, decisions.<br>

The difference between a DWH that works and one that becomes a problem almost always lies in the model. Fact tables with the wrong granularity, poorly designed dimensions, hierarchies that cannot support aggregation queries. Problems that are invisible during development but explode when the business asks for reports the model cannot deliver.<br>

In this section I share real cases of data warehouse design and restructuring: dimensional modeling, balanced hierarchies, slowly changing dimensions, loading strategies. Not Kimball textbook theory, but solutions applied in production on systems that serve real business decisions.<br>

Because a data warehouse is not built to contain data.<br>
It is built to answer questions.
