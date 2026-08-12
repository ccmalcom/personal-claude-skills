---
name: controller-budget
description: "Keep a long-running controller session from burning its budget on context re-reads. Use WHENEVER acting as the driving/controller session for multi-task plan execution — superpowers subagent-driven-development, executing-plans, or any loop that dispatches subagents task after task — and when writing a plan that such a session will execute. Also use when the user asks why a session is expensive, why token/usage burn is high, whether to restart or /compact a session, or how to make agentic execution cheaper. Reach for it the moment work is shaped as 'controller dispatches implementers across many tasks', even if cost was never mentioned."
---

# Controller Budget

A controller session's cost is **context floor × turn count**. Almost nothing else matters. This
skill exists because the intuitive optimization targets — verbose narration, subagent overhead,
the cost of a delegation — were each measured and each turned out to be noise.

## The cost model

Weighting everything in input-equivalent units:

```
cost = cache_read × 0.1  +  cache_write × 1.25  +  input × 1  +  output × 5
```

A controller turn therefore costs roughly `context_size × 0.1`, every turn, forever. Measured
across three controllers executing one plan (11 tasks, 273 turns, 6.23M units):

| | share of spend |
|---|---|
| re-reading context (`cache_read`) | **63–72%** |
| writing new context (`cache_write`) | 11–24% |
| generating output | 13–16% |

The controllers were 89% of total spend; every subagent dispatch combined was 11%.

**The consequence that surprises people:** a turn that thinks hard and writes a long answer costs
about the same as a turn that says "ok". You are paying for the context, not the work. So the only
lever with real leverage is *how big the context is when the turn happens*, multiplied by *how many
turns happen*.

## Rules

### 1. Hand off every 2–3 tasks

Not "between task groups" — that phrasing was tried, measured, and does not fire. A controller
given eight tasks reads them as one group, runs 159 turns, and climbs from a 44k to a 263k floor,
ending at the same per-turn cost as having no rule at all. The one session in that run that did
restart cost **13.5k units/turn against 22.6k** for the two that didn't.

Concretely, at roughly 20 turns per task:

- **After 2–3 completed tasks, stop and hand off.** Do not start task N+1.
- You cannot read your own token count, so **count tasks, not tokens**. The ledger already tracks
  them. (The human can check the real number with `/context`; if they report a floor above ~120k,
  hand off now regardless of task count.)

A restart costs one re-establishment turn — about 40k units — and buys back far more. It is only
free if the state of record is on disk, which is the next rule.

### 2. Write the ledger as you go, not at the end

The handoff is only cheap because the next session can reconstruct everything from files. After
each task, the ledger must carry: what was done, what was verified and how, what is deliberately
deferred and why, and what the next task is. If that is in the session transcript instead of on
disk, a restart loses it and the rule stops being safe to apply.

Superpowers' `subagent-driven-development` already maintains this under `.superpowers/sdd/`. When
using it, the ledger is authoritative — keep it current after *every* task, not batched at the end.

### 3. Never park a loaded controller

Idle time is the single largest one-shot waste. When the prompt cache expires, the whole context
re-enters at `1.25×` instead of `0.1×` — **12.5× the read price**.

Do not reason about how long you can safely idle. The cache TTL is not a constant: it degrades
from 1 hour to 5 minutes as session usage accumulates, and it was measured degrading *within a
single plan execution* (first controller 100% 1-hour buckets, third controller 66% 5-minute). A
33-minute gap that would have been safe under the 1-hour TTL cost **288,771 units**.

So: between tasks, either continue or end the session. If the user needs to step away, hand off
and let them resume fresh — a cold start is cheaper than a warm park.

### 4. Read one task's text per dispatch

Never hold the whole plan document resident. A 20k-token plan sitting in context costs ~2k units
*every turn*; re-reading the one task you need costs ~2.5k *once*.

### 5. Don't optimize the dispatch

Delegation is not where the money goes — measured at 11% of a full plan execution, against 89% for
the controllers. Time spent shaving subagent prompts is time not spent on the floor. If a dispatch
feels expensive, check it against a controller turn before acting: they are usually within a small
multiple of each other.

## When writing a plan for a controller to execute

Two things make a plan cheap to execute, and both are decided at authoring time:

- **Group tasks into handoff-sized batches of 2–3**, and say so in the plan. A plan that reads as
  one 11-task run invites one 159-turn session.
- **Make each task's text self-contained**, so executing it never requires reading its neighbours.
  Shared context belongs in a global-constraints section read once, not re-derived per task.

## What was measured and found irrelevant

Recorded so they don't get re-litigated. Each of these was a confident hypothesis that the data
killed:

- **Verbose narration / thinking.** Turns with no tool call were 14 of 164, ~336k of 4.24M units.
  Output is 13–16% of spend in total. Terseness is not a cost strategy.
- **Delegation round trips.** Polling and orchestration plumbing measured at ~3% of session cost.
- **Tool-call batching to protect the cache.** A tool appeared before 10 cache-boundary moves,
  which looked causal until it was checked: 120 calls, only 12 preceded a rewrite, and those
  clustered at a median context of 264k. Correlation with big-context moments, not causation.
- **Large tool outputs.** Real in principle, absent in practice on a disciplined session — 152
  tool results averaged 287 tokens each. Check before compressing.

The pattern: **cost intuitions are wrong more often than they are right, and they are cheap to
check.** Before optimizing anything here, measure it against the floor.
