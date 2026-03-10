---
title: "AI Manager and Project Management: when artificial intelligence enters the project"
description: "Managing AI in a project is not about using ChatGPT. It means governing the impact of artificial intelligence on architectures, processes and people. Reflections from nearly thirty years of mission-critical systems."
date: "2026-03-17T10:00:00+01:00"
draft: false
translationKey: "ai_manager_project_management"
tags: ["ai", "project-management", "ai-manager", "governance", "strategy"]
categories: ["Project Management"]
image: "ai-manager-project-management.cover.jpg"
---

A few months ago, during a meeting with a banking client, the CTO said something that stuck with me.

> "We need someone who manages AI. Not someone who uses it — someone who governs it."

I nodded without speaking. Because that sentence, in seven seconds, described a role the market is looking for without yet knowing what to call it.

------------------------------------------------------------------------

## 🧩 The fundamental misunderstanding

There is a widespread confusion, and I see it in every project where AI enters the picture.

The confusion is this: thinking that "adopting AI" means integrating a model, connecting an API, having an assistant generate text or code.

No. That is the technical side. The operational side. That is the work of a data scientist or an ML engineer. Important work, to be clear. But it is not the work of someone who governs.

Governing AI in a project means answering questions that no model can answer for you:

- Where does AI create real value and where does it only create enthusiasm?
- How much does it cost to maintain, not just to implement?
- What happens when the model gets it wrong — and who is accountable?
- How does it integrate with existing architectures without compromising stability and security?
- How do you ensure alignment between data governance, compliance and automation?

If you do not have answers to these questions, you are not governing AI. You are being governed by it.

------------------------------------------------------------------------

## 🏗️ It is not a new role. It is a role that did not have a name yet

When I think about it, I realize I have been doing this work long before anyone coined the label "AI Manager".

Thirty years of data architectures. Mission-critical systems in Telco, Banking, Insurance, Public Administration. Environments where data is not an asset to monetize — it is infrastructure to protect.

In those contexts I have always done the same thing: connecting strategy to technical reality. Translating business needs into solutions that actually work, not on a slide but in production. Mediating between those who want everything now and those who know that certain things require time and architecture.

AI has not changed this pattern. It has made it more visible.

Because AI, unlike a database or an ETL pipeline, is a topic that excites boards and scares engineers. Everyone wants a piece of it, few know where to put it. And the role of the person in between — between management enthusiasm and infrastructure caution — becomes crucial.

------------------------------------------------------------------------

## 📍 Where AI creates real value (and where it does not)

I have learned something over the past three years, working with AI in real project contexts: the value of AI is almost never where people think it is.

It is not in automatic code generation. Not in the chatbot answering customers. Not in the report that writes itself.

Real value lies in three places:

**1. Accelerating analysis**

AI is devastating when it needs to analyze context. Reading thousands of lines of code, correlating logs, spotting patterns. What costs a senior engineer two hours, AI does in seconds. Not better — faster. And speed, in a project with deadlines, is money.

**2. Reducing decision noise**

In every complex project there is a moment when information is too much and the team no longer knows what is urgent versus what is important. AI can do triage. It can classify, prioritize, highlight anomalies. It does not decide for you — it presents data in a way that makes the decision clearer.

**3. Documentation and knowledge transfer**

Nobody likes documenting. Nobody. AI can generate documentation from code, commits, issues. Not perfect, but enough to avoid losing knowledge when someone leaves the project. And anyone who has managed projects knows how much that costs.

Everything else — the shiny demos, the presentations with bold percentages, the vendors promising triple-digit ROI — is noise. The AI Manager is the one who separates the signal from the noise.

------------------------------------------------------------------------

## ⚖️ The triangle the PM must govern

In every project where AI enters a regulated environment, there is a triangle that keeps coming back:

**Data governance — Compliance — Automation.**

You can have the most efficient automation in the world, but if it violates data governance policies, it is a risk. You can have impeccable governance, but if it blocks every form of automation, the project stalls. You can be perfectly compliant, but if you do not know which data you are using to train or query the model, compliance is only on paper.

The AI Manager must keep these three vertices in balance. Continuously. Not once at the start of the project — every week.

I have seen projects where AI was integrated without anyone verifying the provenance of the training data. In banking. With data subject to GDPR. The DPO found out three months later.

That is not incompetence. It is absence of governance. It is the absence of someone asking the right question at the right time.

------------------------------------------------------------------------

## 🔬 Integrate, do not replace

Something I repeat in every kickoff meeting: AI integrates into existing architectures. It does not replace them.

It sounds obvious, yet the temptation is always the same: the vendor proposing to "rethink the infrastructure through an AI lens", the consultant who wants a greenfield architecture, the manager who saw a demo and now wants everything new.

No.

Mission-critical architectures are not thrown away because a new technology has arrived. They evolve. They extend. They are protected.

The AI Manager is the person who says "this model plugs in here, with these precautions, with this fallback plan". Not the person who says "let us throw everything away and rebuild with AI".

In thirty years of systems, I have seen at least five "revolutionary" technologies that were supposed to change everything. Client-server. Internet. Cloud. Big Data. Now AI. None of them changed everything. Each of them changed something. And those who governed the change well were those who integrated it with intelligence — not with enthusiasm.

------------------------------------------------------------------------

## 🎯 Why AI is not magic

There is a phrase I use often, and I never tire of repeating it.

AI is not magic. It is architecture applied to intelligence.

A model is a component. Like a database, like a message broker, like a load balancer. It needs clean inputs, monitoring, maintenance, governance. It needs someone who understands what it does, what it can do, and above all what it cannot do.

The Project Manager who ignores these aspects and delegates everything to the technical team is making the same mistake as the one who delegated security to the sysadmin and then was surprised by the data breach.

AI is an architectural responsibility. And like all architectural responsibilities, it must be governed from above. Not from below.

------------------------------------------------------------------------

## 💬 To those deciding whether AI "belongs" in their project

If you are evaluating whether to introduce AI in a project — not an experiment, a real project, with deadlines, budget and stakeholders — here is one piece of advice worth more than any tool.

Do not start from the technology. Start from the problem.

What is the bottleneck? Where does the team lose the most time? Where are decisions slower than they need to be? Where is knowledge being lost?

If the answer to any of these questions involves analyzing large volumes of data, classifying information, or accelerating repetitive processes — then AI can help. But only if someone governs it.

And governing it does not mean controlling it. It means understanding it well enough to know when to trust it and when not to.

That is the job of the AI Manager. And whether you call it that or not, it is a role every serious project will need.
