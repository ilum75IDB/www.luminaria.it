---
title: "Performance & Tuning"
date: "2026-03-10T10:00:00+01:00"
description: "Oracle performance in real environments: partitioning, execution plans, query optimization and tuning strategies. When data grows and queries can no longer keep up."
layout: "list"
---
In Oracle, performance is not measured by gut feeling.<br>
It is measured with numbers: response time, consistent gets, physical reads, wait events.<br>

The problem is that when a database is small, everything works. Queries respond in milliseconds, indexes are enough, a full table scan does not hurt. Then data grows — millions, hundreds of millions, billions of rows — and what used to work stops working.<br>

Not because the database is broken. Because it was not designed for that scale.<br>

In this section I share real optimization cases: tables with billions of rows, queries that go from hours to seconds, execution plans that hide invisible traps. Not textbook theory, but solutions applied in production on real systems.<br>

Because in Oracle, tuning is not a mystic art.<br>
It is engineering. And like all engineering, it is based on measurements, not on intuition.
