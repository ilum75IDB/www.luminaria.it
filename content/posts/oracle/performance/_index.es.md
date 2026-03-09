---
title: "Performance & Tuning"
date: "2026-03-10T10:00:00+01:00"
description: "Rendimiento Oracle en entornos reales: partitioning, planes de ejecución, optimización de consultas y estrategias de tuning. Cuando los datos crecen y las consultas ya no pueden seguir el ritmo."
layout: "list"
---
En Oracle el rendimiento no se mide por intuición.<br>
Se mide con números: tiempo de respuesta, consistent gets, physical reads, wait events.<br>

El problema es que cuando una base de datos es pequeña, todo funciona. Las consultas responden en milisegundos, los índices son suficientes, un full table scan no asusta. Después los datos crecen — millones, cientos de millones, miles de millones de filas — y lo que funcionaba deja de funcionar.<br>

No porque la base de datos esté rota. Porque no fue diseñada para esa escala.<br>

En esta sección comparto casos reales de optimización: tablas con miles de millones de filas, consultas que pasan de horas a segundos, planes de ejecución que esconden trampas invisibles. No teoría de manual, sino soluciones aplicadas en producción sobre sistemas reales.<br>

Porque en Oracle el tuning no es un arte místico.<br>
Es ingeniería. Y como toda ingeniería, se basa en mediciones, no en intuiciones.
