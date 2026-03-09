---
title: "Access & Control"
date: "2026-03-10T10:00:00+01:00"
description: "Oracle security in practice: users, roles, privileges and audit. The most granular security model on the market — and the easiest to misconfigure."
layout: "list"
image: "security.cover.jpg"
---
In Oracle, security is not an option.<br>
It is a load-bearing structure of the database, present from day one.<br>

Oracle's security model is arguably the most comprehensive among relational databases: users separated from schemas (but linked), predefined and custom roles, system and object privileges, resource profiles, unified audit.<br>

Powerful? Enormously.<br>
Misconfigured in 90% of the installations I have seen? Also yes.<br>

The problem is not a lack of tools. It is the excess. Oracle offers so many security options that the temptation is to simplify with a `GRANT DBA` and move on to the next problem. I have seen it done by developers, system administrators, even DBAs with years of experience.<br>

In this section I share how to design Oracle security in real environments: role separation, the principle of least privilege, audit and the traps that even experienced professionals fall into.<br>

Because in Oracle, security is not something you add later.<br>
You design it first. Or you pay for it afterwards.
