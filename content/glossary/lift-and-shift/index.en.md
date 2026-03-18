---
title: "Lift-and-Shift"
description: "Migration strategy that moves a system from one environment to another without modifying its architecture, code, or configuration."
translationKey: "glossary_lift-and-shift"
aka: "Rehosting"
articles:
  - "/posts/project-management/tecnica-si-e-yes-and"
---

**Lift-and-Shift** (rehosting) is a migration strategy that consists of moving a system from one environment to another — typically from on-premises to cloud — without modifying its architecture, application code, or configuration. The system is taken as-is and "lifted and shifted."

## How it works

The infrastructure is replicated in the target environment: same virtual machines, same databases, same middleware. The advantage is speed: no code rewriting, no architectural redesign. The risk is carrying over all the problems from the original environment, including inefficiencies and technical debt.

## When to use it

When the priority is exiting a datacenter quickly (contract expiration, hardware decommission), when the budget does not allow rearchitecture, or as the first phase of an incremental migration where components are then modernized one by one.

## What can go wrong

A lift-and-shift to cloud without optimization can cost more than the original on-premises infrastructure. Applications not designed for cloud do not leverage elasticity, auto-scaling, and managed services. The result is often a private datacenter rebuilt in the cloud at a higher price.
