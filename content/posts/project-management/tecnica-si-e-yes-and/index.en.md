---
title: "The Yes-And technique: how I defused a meeting that was about to blow up"
description: "The Yes-And technique, born in improvisational theatre, applied to conflict management in IT teams. A real case of a meeting that was going south — and how three words changed everything."
date: "2026-01-13T10:00:00+01:00"
draft: false
translationKey: "tecnica_si_e_yes_and"
tags: ["conflict-management", "team-leadership", "communication", "meeting"]
categories: ["Project Management"]
image: "tecnica-si-e-yes-and.cover.jpg"
---

It was a Thursday afternoon, one of those meetings that was supposed to last an hour on paper. Seven of us, connected on a call. The agenda was straightforward: decide the migration strategy for an Oracle database from on-premise to cloud.

Straightforward, sure. On paper.

Twenty minutes in, the meeting had turned into a duel.

------------------------------------------------------------------------

## 🔥 The spark

On one side was the infrastructure manager. Experienced, twenty years of datacenters behind him. His position was rock-solid: **lift-and-shift migration, zero changes to the architecture, we move everything as-is**.

On the other, the lead developer. Young, sharp, with clear ideas. He wanted to rewrite the application layer, adopt cloud-native services, containerize everything. **Let's rebuild from scratch, the code is old anyway.**

Two legitimate positions. Two real perspectives. Two smart people.

But the conversation had taken a familiar — and dangerous — turn.

"No, it makes no sense to move everything to cloud without rethinking the architecture."\
"No, rewriting everything is a huge risk and we don't have the budget."

No. No. No.

Every sentence started with "no". Every answer was a negation of the previous one. Arms crossed, tone rising, sentences getting shorter. I know that pattern. I've seen it hundreds of times. And I know how it ends: it doesn't end. The meeting closes without a decision, gets rescheduled to next week, and in the meantime nobody does anything because "we haven't decided yet".

The project stalls. Not for technical reasons. For pride.

------------------------------------------------------------------------

## 🎭 Three words that change everything

At that point I did something very simple. I waited for a pause — because in heated discussions there's always a moment when everyone catches their breath — and I said:

> "Marco, you're right: moving everything to cloud without changing anything is the fastest way to get to production. **And** we could also identify two or three components that, during the migration, it makes sense to rethink as cloud-native. Luca, which ones would you pick?"

Nobody said "no". Nobody was contradicted.

Marco saw his position validated — his conservative approach was the starting point. Luca was given a concrete role — choosing what to modernize, with a clear mandate.

In thirty seconds, two people who were fighting found themselves collaborating on the same whiteboard.

The meeting ended early. With a decision. A real one.

------------------------------------------------------------------------

## 🧠 What the "Yes-And" technique is

What I did has a name. It's called **"Yes-And"**. It comes from improvisational theatre, where there's a fundamental rule: **never deny your scene partner's proposal**.

If someone says "We're on a boat in the middle of the ocean", you don't respond "No, we're in an office". You respond "Yes, and it looks like a storm is coming". You build. You add. You move forward.

In project management it works the same way.

When someone proposes something and you respond "No, but...", here's what happens psychologically:

- the other person goes on the defensive
- they stop listening to whatever comes after "but"
- they focus on how to counter-argue, not how to solve
- the conversation becomes a ping-pong of negations

When you respond "Yes, and...", the opposite happens:

- the other person feels acknowledged
- defences come down
- they become open to hearing your addition
- the conversation becomes constructive

It's not manipulation. It's not empty diplomacy. It's a precise technique for moving decisions forward without burning relationships.

------------------------------------------------------------------------

## 🛠️ How it works in daily practice

In thirty years of projects, I've applied "Yes-And" in dozens of situations. It works wherever there's a decision to make and multiple people with different opinions.

### In project meetings

**Instead of:** "No, the three-month timeline is unrealistic."\
**Try:** "Yes, three months is the target. And to get there we'd need to cut the first release scope to these three features — the rest goes into phase two."

See the difference? In the first version you have a wall. In the second you have a plan.

### In code reviews

**Instead of:** "No, this approach is wrong, you wrote it in an overly complicated way."\
**Try:** "Yes, it works. And we could simplify it by extracting this logic into a separate method — it becomes more testable."

The developer doesn't feel attacked. They feel helped. And next time they come to you for input *before* writing the code, not after.

### In stakeholder negotiations

**Instead of:** "No, we can't add that feature now, we're already behind schedule."\
**Try:** "Yes, that feature makes sense. And to include it without compromising the release date, we'd need to swap it with this other one that's lower priority. Which of the two do you prefer?"

The stakeholder doesn't hear a "no". They hear a "yes, and now let's decide together how to do it".

------------------------------------------------------------------------

## ⚠️ When "Yes-And" doesn't work

It would be nice to say it always works. It doesn't. There are situations where "Yes-And" is the wrong tool.

**Security issues.** If someone proposes removing authentication from the production database because "it slows down queries", the answer is not "Yes, and...". The answer is "No. Full stop."

**Process violations.** If a developer wants to deploy to production on Friday evening without tests, there's no "Yes-And" for that. There's a process, and it must be followed.

**Non-negotiable deadlines.** When go-live is Monday and it's Thursday, it's not the time to build on everyone's ideas. It's the time to decide, execute and close.

**Toxic behaviour.** "Yes-And" works with people acting in good faith who have different opinions. It doesn't work with people who just want to be right, who sabotage, who refuse to listen on principle. In those cases you need a different kind of conversation — private, direct and very frank.

The technique is not a magic formula. It's a tool. And like all tools, you need to know when to use it and when to put it down.

------------------------------------------------------------------------

## 📊 The hidden cost of "No, but..."

I tried to do a rough calculation on a project I managed two years ago. A team of eight, meetings three times a week.

| Situation | Average meeting length | Decisions made |
|---|---|---|
| Before ("No, but..." culture) | 1h 20min | 0.5 per meeting |
| After ("Yes, and..." culture) | 45min | 1.8 per meeting |

The team was making decisions three times faster and meetings lasted almost half as long.

I don't have scientific data. These are empirical numbers, collected on one specific project. But the pattern is consistent with what I've seen over twenty years: teams that discuss constructively move faster than those that argue. Not because they avoid conflict — because they get through it better.

------------------------------------------------------------------------

## 🎯 What I've learned

"Yes-And" is not diplomacy. It's not avoiding confrontation. It's not saying yes to everything.

It's recognizing that **most discussions in IT projects aren't about who's right**. They're about how to move things forward. And things move forward when people feel heard, not when they're defeated.

I've seen projects stall for weeks because two brilliant people couldn't stop saying "no" to each other. And I've seen those same projects unblock in fifteen minutes when someone had the good sense to say "yes, and...".

You don't need a communication course. You don't need a coach. You need to try, the next time someone says something you disagree with, to respond "Yes, and..." instead of "No, but...".

A simple exercise. One that changes the way decisions are made.\
That changes the way people work together.\
And that, sometimes, saves a meeting that was about to explode.

------------------------------------------------------------------------

## 💬 For anyone who's been there at least once

If you've ever sat in a meeting where two people were talking over each other and nobody was listening to anyone. If you've ever seen a project stall not because of a technical problem, but because of a communication problem. If you've ever thought "why can't we just decide?"

Try "Yes-And". Next meeting. Just once.

It costs nothing. It needs no approval. It doesn't require a budget.\
It just requires the ability to hold back for one second before saying "no" — and replace it with "yes, and...".

The result might surprise you.
