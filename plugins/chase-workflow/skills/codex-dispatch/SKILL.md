---
name: codex-dispatch
description: "Delegate implementation work to Codex correctly — how to write the brief, which gates to name, when to re-dispatch fresh instead of resuming, and how to verify what comes back. Use WHENEVER handing a task to Codex (the codex:codex-rescue subagent, /codex commands, or any Codex delegation), when a Codex run reports blocked/incomplete, when a fix round is needed after a Codex implementation, or when writing a plan whose tasks will be executed by Codex. Also use when deciding whether a task is a good fit for Codex at all. Reach for it before writing the dispatch prompt, not after the result comes back disappointing."
---

# Codex Dispatch

Codex is additive capacity, not a cheaper Claude. It runs on a separate quota, so work sent there
does not consume Claude usage — but it is only worth dispatching if the result comes back correct,
and most of what makes that happen is decided in the brief.

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
  the single focused test file command instead.
- **Verify the test command actually matches tests before you send it.** In a repo with two
  runners and complementary scopes, a plausible command can match *zero* tests, exit 0, and read as
  a pass. Check which runner claims that path first. The dangerous gate failures are the ones that
  produce no output rather than a red X.
- **Diff the brief against the current state of the fixtures/tests it references.** When the
  driving session changes something after the plan was written, state the delta in the dispatch as
  an explicit, justified deviation from the otherwise-verbatim brief.
- **Say when a divergence should be documented rather than fixed.** "Keep the code, write the
  comment" is a legitimate outcome. If the brief does not say so, Codex will dutifully "fix" better
  code into worse code to match the reference implementation.
- **Never name a `.env` file or ask Codex to inspect secrets.** Codex runs outside the Claude Code
  hook system, so a `PreToolUse` guard that blocks `cat .env` locally does not protect anything on
  that side. This is the one hard rule in this skill.

## Prefer a fresh dispatch over a resume when the round requires an edit

Write access does not reliably survive a resume. A thread that wrote to a file minutes earlier can
come back from a resumed round with "blocked by the workspace's read-only sandbox: the edit was
rejected, and approval escalation is disabled."

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

Codex reports honestly — including reporting INCOMPLETE rather than claiming green, which is the
right behavior. But honest reporting is not verification.

- **A claim about the code is a factual claim; check it.** This applies to Codex's attributions
  *and* to prior tasks' stated blockers, which arrive with more authority and get less scrutiny.
  One deferral in a measured run was justified by "seeding this would perturb the recorded
  fixtures" — two greps showed it could not, and the gap had never actually been blocked. Cost of
  checking: two greps. Cost of not checking: a route ships untested behind a reason that was never
  true.
- **Mutation-test anything the plan calls load-bearing.** Break the invariant deliberately and
  confirm something goes red. In one run this revealed that all four fixture replays still passed
  under the injected bug — the entire parity apparatus was blind to it, and a single hand-written
  failure-injection test was the only thing catching it. If nothing goes red, the constraint is
  documentation, not engineering. Cost: about three minutes.

## Cost note

Do not optimize the dispatch itself. Measured across a full plan execution, every Codex dispatch
combined was 11% of spend; the controller sessions were 89%. If a dispatch feels expensive, check
it against a single controller turn before acting — see `controller-budget`.
