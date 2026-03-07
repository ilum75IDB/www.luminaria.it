---
title: "4 million euros, two multinationals, zero software: the true story of a failure foretold"
description: "An insurance client spent over 4 million euros hiring two IT consulting giants to build a custom management system. The result: nothing. By contrast, a data warehouse built by two people in 3 years with 60,000 lines of code that runs every single day."
date: "2026-02-16T10:00:00+01:00"
draft: false
translationKey: "consulenza_it_milioni_sprecati"
tags: ["Project Management", "IT Consulting", "Data Warehouse", "Insurance", "Oracle", "Outsourcing"]
categories: ["Project Management", "Consulting"]
image: "4-milioni-nessun-software.cover.jpg"
---

The story I'm about to tell is true. I won't name names — not out of diplomacy, but because names don't matter. What matters is understanding the mechanism. Because this mechanism repeats itself, identically, in dozens of companies. And it costs millions.

------------------------------------------------------------------------

## 🏢 The client: an insurance group with a legitimate ambition

A solid company in the insurance sector. Operations in Italy, France, Northern European countries, Spain. Thousands of employees, millions of policies under management, a growing business.

At some point, the board makes a reasonable decision: **we need a custom management system**. A system that reflects our processes, our business rules, the regulatory specificities of every country where we operate.

A legitimate decision. Sensible. Even strategic.

The problem isn't the decision.\
The problem is **who they entrust it to**.

------------------------------------------------------------------------

## 💰 Act one: the big multinational (2013–2018)

One of the Big Names in global IT consulting is brought in. A name everyone knows. Thousands of consultants, offices on every continent, PowerPoint presentations that could bring you to tears.

The project kicks off. Requirements are defined. The budget is estimated. Contracts are signed.

Months pass. Then years.

Deliverables arrive — on paper. But the software doesn't work as expected. Specifications change. Costs balloon. Consultants rotate: the one who understood the domain leaves, a new one arrives and starts from scratch. The classic pattern of fixed-scope consulting that becomes, in practice, open-ended engagement.

**From 2013 to 2018: over 2.5 million euros spent.**\
Result: incomplete, unstable software that no one internally knew how to maintain.

Because *they* had written the code. With *their* conventions. With *their* architecture. And when they left, they took the knowledge with them.

------------------------------------------------------------------------

## 🔄 Act two: let's change supplier (2018–2022)

Management, burned but not defeated, decides to switch. "The problem was the supplier," they think. "Let's get a better one."

Enter another multinational. Equally famous. Equally large. Equally expensive.

New kickoff. New requirements analysis — because obviously they can't build on the previous supplier's work. New slides. New promises.

And history repeats itself.

Same problems, different actors. Consultant turnover. Loss of know-how. Timelines stretching. Budgets exploding. Endless meetings discussing milestones that never arrive.

**From 2018 to 2022: another 1.5 million euros.**\
Result: another piece of software that fails to meet business needs.

Total invested over nearly a decade: **over 4 million euros**.\
Working software: **zero**.

------------------------------------------------------------------------

## 📊 Let's tally up the disaster

| Period | Supplier | Investment | Result |
|---|---|---|---|
| 2013 – 2018 | Multinational A | ~€2,500,000 | Incomplete software, abandoned |
| 2018 – 2022 | Multinational B | ~€1,500,000 | Inadequate software, abandoned |
| **Total** | | **~€4,000,000+** | **No software in production** |

Four million euros. Nearly ten years of project work. Two of the most prestigious names in global IT consulting.

And in the end, the company finds itself exactly where it started.

It's not bad luck. It's a pattern.\
And anyone who's worked in this industry for thirty years, like me, recognises it at first glance.

------------------------------------------------------------------------

## 🧠 Why it happens: the anatomy of failure

This kind of failure isn't an accident. It's the predictable outcome of a business model with a structural flaw.

**1. The incentive is wrong.**\
A large consulting firm makes money by selling man-days. The longer the project lasts, the more it bills. There's no real incentive to finish the project quickly and well. There's an incentive to keep it alive as long as possible.

**2. Turnover is endemic.**\
Major consulting multinationals have annual turnover rates of 15–25%. In a project lasting five years, the team is completely renewed at least twice. Each time you start over: new learning curve, new interpretation of requirements, new mistakes.

**3. Know-how walks out the door.**\
When the supplier finishes (or gets fired), the system knowledge leaves with them. The client is left with software they don't understand, can't maintain, and can't evolve.

**4. Specifications become a weapon.**\
In a custom project of this scale, specifications are always incomplete — because the business is complex and evolving. This becomes the perfect alibi: "the software doesn't work because the specifications changed." And it's always someone else's fault.

------------------------------------------------------------------------

## ✅ The turning point: buy, don't build

In the end, after nearly a decade and over 4 million burned, the company makes the decision it should have made from the start:

**Buy an existing market software product and adapt it internally to their needs.**

A commercial insurance product, battle-tested, with a stable codebase and a support community. And an internal team — people who know the business, who stay with the company, who accumulate knowledge instead of dispersing it — tasked with customising and evolving it.

Cost? A fraction of what was spent in the previous ten years.\
Result? A system that works. That evolves. That the company truly owns.

The lesson is brutal in its simplicity:

> Not everything needs to be built from scratch. And above all, not everything should be delegated to those who have no interest in finishing.

------------------------------------------------------------------------

## 🏗️ The comparison that hurts: our Data Warehouse

And here's the part of the story I know from the inside. Because for the same company, during the same period, a colleague and I built something that works. Every single day.

A complete **Data Warehouse**. Designed, developed, deployed to production, and maintained **by two people**.

Not a demo. Not a prototype. A production system that:

- **Loads data every day** — the entire ETL cycle runs in **one and a half hours**
- **Integrates 4 different source systems** — each with its own format, protocol, and quirks
- **Collects data from 4 geographic areas**: Italy, France, Northern European countries, Spain
- **Comprises approximately 60,000 lines of code** written by four hands
- **The architecture was designed by me** — from the data model to the loading strategy, from error handling to historicisation

| | Custom management software | Data Warehouse |
|---|---|---|
| Team | Two multinationals (dozens of consultants) | **2 people** |
| Project duration | ~10 years (and counting) | **3 years** |
| Budget | €4,000,000+ | A fraction |
| Lines of code | Unknown (and abandoned) | **~60,000** (documented, maintained) |
| Result | No software in production | **System in daily production** |
| Processing time | — | **1h 30min / day** |
| Geographic coverage | — | 4 countries, 4 source systems |
| Know-how | Lost with every supplier change | **Internal, stable, documented** |

Two people. Three years. A system that wakes up every morning, gathers data from four corners of Europe, transforms it, loads it, and makes it available for business decisions. In an hour and a half.

Sixty thousand lines of code. Each one thought through, tested, maintained by those who wrote it.

No PowerPoint. No kickoff. No consultant walking away with the knowledge.\
Just competence, solid architecture, and work done right.

------------------------------------------------------------------------

## 🎯 The lesson

When I talk to companies about to embark on a major IT project, I always say the same thing:

**Don't pay for a brand. Pay for the people.**

A small team of professionals who know the domain, who stay on the project, who are accountable for the result — is worth more than a hundred rotating consultants billing days.

Software isn't built with slides. It's built with hands in the code, architecture in the mind, and responsibility on the shoulders.

Four million euros up in smoke teach one thing:

> The highest cost isn't the wrong supplier you pick.\
> It's the time you waste before realising the solution was simpler than what they sold you.

------------------------------------------------------------------------

## 💬 To those about to sign that contract

If your company is about to entrust a critical project to a large consulting firm, pause for a moment.

Ask yourself:

- Who will write the code? Will they still be with the company in two years?
- If the supplier leaves tomorrow, could we maintain the system?
- Is there a market product that covers 80% of our needs?
- Can we build a small, competent, stable internal team?

The answers to these questions are worth more than any commercial proposal.\
Because the difference between a project that works and one that burns millions isn't about technology.

It's about the people. The continuity. The accountability.

And the ability to say "no" to those who sell you complexity when the solution is simple.
