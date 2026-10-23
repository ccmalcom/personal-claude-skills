---
name: codex-dispatch
description: "Delegate work to Codex correctly — how to write the brief, which gates to name, when to re-dispatch fresh instead of resuming, and how to verify what comes back. Use WHENEVER handing a task to Codex (the codex:codex-rescue subagent, /codex commands, or any Codex delegation), when a Codex run reports blocked/incomplete, when a fix round is needed after a Codex implementation, when writing a plan whose tasks will be executed by Codex, and before executing a plan that Codex could review first. Also use when deciding whether a task is a good fit for Codex at all, or when an inventory of the repo is needed before scoping work. Reach for it before writing the dispatch prompt, not after the result comes back disappointing."
---

# Codex Dispatch

Codex is additive capacity, not a cheaper Claude. It runs on a separate quota, so work sent there
does not consume Claude usage — but it is only worth dispatching if the result comes back correct,
and most of what makes that happen is decided in the brief.

## Start from the templates, always

`prompt-templates.md` (this directory) holds the tested prompt for each role plus a standing-rules
block, a pre-send gate check, Claude's review checklist, and verified helper commands. **Paste the
relevant block, then edit down. Never compose one from memory.**

The failure mode of a hand-rolled prompt is *silent subtraction*. One dispatch named three sensible
gates, Codex ran exactly those three, all green, work correct — and the stop hook then failed the
turn on `prettier`, which had been line 158 of the template the whole time. Nothing about that
dispatch looked incomplete. What was missing was only visible against the template.

## Where Codex fits in the pipeline

The skills run in order — `superpowers:brainstorming` → `superpowers:writing-plans` →
`superpowers:subagent-driven-development` — and Codex has a role at three of those seams, not just
the last one:

| Stage | Codex role | Template |
|---|---|---|
| Before brainstorming/scoping | Read-only repo inventory | §1 |
| Plan drafting | Draft the plan doc from the inventory | §2 |
| **After writing-plans, before SDD** | **Read-only plan review** | **§3** |
| Inside the SDD loop | Task execution, one task per dispatch | §4 |

**The plan review is the highest value-per-minute use found so far.** A plan asserts dozens of
checkable facts in one place and checking them is pure lookup — exactly what a budget-boxed run is
good at. One pass returned four findings, all four real, including two symbols that did not exist
and would otherwise have failed at the first `tsc` mid-wave, after a dispatch had already spent its
budget writing against a fictional API. Run it between `writing-plans` and the first SDD dispatch.

Claude keeps scope, design, and adjudication. The loop that works:
`Codex explores → Claude decides scope → Codex drafts → Claude verifies against the real repo →
Codex reviews the plan → SDD executes`.

## Keep the forwarder file-blind

The dispatch subagent is a thin forwarder that reads nothing from the repo. That is a property to
preserve, not an accident: it is why a dispatch costs a tight 18k–56k units instead of scaling with
repo size. If you find yourself having the forwarder explore, read files, or summarize context
before handing off, stop — put what Codex needs *in the brief* instead.

## Write the brief so it survives being read literally

Codex follows a brief faithfully. That is its main virtue and the source of every failure mode
below: **a stale brief propagates cleanly into stale work.**

- **Name only gates that are fast and scoped to the touched files.** A slow whole-repo suite in a
  Codex prompt buys nothing — the controller re-runs it regardless — and can eat the task. One
  measured case: a 163-second server suite (still growing) against a ~10-minute budget that also
  had to cover Codex's own repo exploration. It hung, got killed, and cost a full round trip. Give
  the single focused test file command instead. Excluding a slow gate and running it yourself is
  fine; forgetting a fast one is not.
- **Verify each gate command matches something before you send it.** `npx jest --listTests <path>`
  and `npx vitest list <path>` each answer this in one command. Three separate waves shipped a gate
  that matched *zero* tests, exited 0, and read as a pass — a path only the other runner owned, a
  `.ts` glob against a `.tsx` file, a directory outside the runner's `include`. **The dangerous gate
  failures produce no output rather than a red X.** Check `package.json`'s `scripts` once per wave
  too: `npm test` reads as "the test suite" and here it is jest only, a quarter of one.
- **A `| tail -N` is not a gate.** It silently depends on how chatty the tool's epilogue is, and
  upgrades move that boundary — a `pytest -q | tail -3` stopped showing the pass count entirely when
  the warnings summary moved last. Any gate meant to produce a *number* must extract it (`grep`,
  `--json`), not slice the tail.
- **Check the task's dependencies exist before writing the brief.** Codex's sandbox has no network,
  so a task whose tests need a new devDependency is controller work first, dispatch second — or the
  run spends its budget discovering it cannot proceed.
- **Diff the brief against the current state of the fixtures/tests it references.** When the
  driving session changes something after the plan was written, state the delta in the dispatch as
  an explicit, justified deviation from the otherwise-verbatim brief.
- **Say when a divergence should be documented rather than fixed.** "Keep the code, write the
  comment" is a legitimate outcome. If the brief does not say so, Codex will dutifully "fix" better
  code into worse code to match the reference implementation.
- **Never name a `.env` file or ask Codex to inspect secrets.** Codex runs outside the Claude Code
  hook system, so a `PreToolUse` guard that blocks `cat .env` locally does not protect anything on
  that side. This is the one hard rule in this skill.

### Gates that cannot be delegated at all

Some gates are structurally impossible in the sandbox, not flaky. `next build` dies fetching Google
Fonts; `npm install` cannot run. **Never read the absence of one of these from a Codex report as a
pass** — the controller owns them and runs them every wave. `npm run build` in particular has caught
what no other gate could: eight consecutive red preview deploys across four waves, invisible to
tests, `tsc`, and eslint alike.

## Prefer a fresh dispatch over a resume when the round requires an edit

A thread's sandbox is fixed at creation, and `--write` must be on the **first** call. That is why a
thread that wrote to a file minutes earlier can come back from a resumed round with "blocked by the
workspace's read-only sandbox: the edit was rejected, and approval escalation is disabled" — and why
re-resuming with `--write` cannot rescue it.

Recognize it by shape: the run reports a *permissions* block rather than a code problem, and
`git status` shows nothing changed. Check the tree before doing anything else — the report is
honest about having applied nothing, but a killed run does **not** roll back partial edits, so
confirm independently.

Practical rule: **resume is reliable for question-answering and re-running gates. If the round
requires an edit, dispatch fresh.** A fresh agent knows nothing of the prior thread, so that
dispatch must be self-contained:

1. Paste the current body of the function being changed.
2. State the problem and the intended fix.
3. **List the DO-NOTs explicitly.** A fresh agent does not know the neighbouring function is
   deliberately inconsistent and will happily "harmonize" them.

## Verify what comes back

Codex reports honestly — including reporting INCOMPLETE rather than claiming green, and halting
rather than adapting when a brief disagrees with the repo. That is the right behavior. But honest
reporting is not verification.

- **A claim about the code is a factual claim; check it.** This applies to Codex's attributions
  *and* to prior tasks' stated blockers, which arrive with more authority and get less scrutiny.
  One deferral in a measured run was justified by "seeding this would perturb the recorded
  fixtures" — two greps showed it could not, and the gap had never actually been blocked. Cost of
  checking: two greps. Cost of not checking: a route ships untested behind a reason that was never
  true.
- **Verify a STOP the same way — the finding, not the framing.** Keep the "stop and report"
  instruction even though most stops resolve in one message: across three stops in one wave, all
  three observations were factually accurate and two carried conclusions that did not follow ("this
  function has two callers, the plan said one" — the second was the unit test *of the function being
  deleted*). Cheap stops beat silent adaptation; a silently "fixed" expectation in the third case
  would have destroyed the only end-to-end evidence that the change worked.
- **Measure before theorising about a slow run.** A 22-minute `verifying` phase was confidently
  attributed to a slow suite; measured on the same machine, the whole gate set ran in under four
  minutes. The cause is still unknown, and the invented one would have taught everyone to over-budget
  every dispatch. Budget Codex's *exploration*, not its verification.
- **Mutation-test anything the plan calls load-bearing.** Break the invariant deliberately and
  confirm something goes red. In one run this revealed that all four fixture replays still passed
  under the injected bug — the entire parity apparatus was blind to it, and a single hand-written
  failure-injection test was the only thing catching it. If nothing goes red, the constraint is
  documentation, not engineering. Cost: about three minutes.

## When the plan itself is wrong

Codex executes the plan literally, so a defect in the plan lands in the repo intact. Two shapes seen
repeatedly, both worth checking while writing the plan (`superpowers:writing-plans`) rather than
during execution:

- **A plan's expected-output table is a hypothesis, not an oracle.** One plan told the executor to
  compare production columns against an expected shape and **STOP** on mismatch. Production
  mismatched on two of six — and both times the plan was wrong, not the database. When reality
  disagrees, read the source that produces reality before honouring a stop instruction. Put
  `Record what you actually observe, not what this plan predicts` in the plan itself; it makes
  benign drift cheap instead of tempting the executor to reconcile it silently.
- **Earlier waves leave deliberate tripwires.** An assertion planted to fail loudly when a later
  wave flips a switch is good practice — but a plan that flips it and never mentions the assertion
  produces a red suite with no explanation. Grep for pinning assertions before planning the flip,
  and when triaging that red, **delete** the stale assertion rather than weakening it to accept
  either value. The tripwire did its job.

## Cost note

Do not optimize the dispatch itself. Measured across a full plan execution, every Codex dispatch
combined was 11% of spend; the controller sessions were 89%. If a dispatch feels expensive, check
it against a single controller turn before acting — see `controller-budget`.
