# Codex prompt templates

Copy a template, fill the brackets, delete what doesn't apply. **Paste these blocks; do not compose
one from memory.** The failure mode of a hand-rolled prompt is silent subtraction — a dispatch that
names three plausible gates and passes all three looks complete, and what is missing is only visible
against the template.

Every rule here was paid for by a defect. The narrative for each lives in the originating repo's
workflow notes; this file is the operational version.

---

## Standing rules — paste into every prompt

```
Treat every code block and file:line citation you are given as an UNVERIFIED SKETCH. Check every
symbol, type, column, and function name against the real repo before using it, and report any
deviation you find rather than silently adapting. Do not invent `any` to work around a type that
does not exist.

Do not run git commit, cherry-pick, merge, push, or deploy. Chase does all of those by hand.
Run `npx prettier --write` only on files you touched. Never format the repo.

Always use `git --no-pager` or prefix with `GIT_PAGER=cat`. A paged git command hangs your shell.

You cannot run these — do not try, and report the blocker instead of inventing a result:
  - [LIST THEM: see "What Codex cannot run" below]
If a step needs one, say so and stop; never fabricate a value you could not measure.

If the task tells you to edit something you cannot find, do NOT invent the edit and do NOT silently
skip it. Locate it, and if it genuinely is not there, say so plainly and stop.
```

The "unverified sketch" paragraph earns its place — Codex reports unprompted, correct deviations.
**But note the asymmetry it creates:** it primes Codex to expect plan errors, which makes it more
likely to accept a *wrong* correction you hand it. See the review checklist.

### What Codex cannot run

Derive this list per repo instead of copying a stale one. Codex's sandbox **has no network**, which
is the root cause of most of it:

- `npm install` / `pip install` — so **any task whose tests need a new dependency is controller work
  first, dispatch second**, or the run burns its budget discovering it cannot proceed. Check the
  dependency exists before writing the brief.
- `next build` — dies fetching Google Fonts. Structural, not flaky; never read its absence from a
  Codex report as a pass. The controller runs `build` every wave.
- Any fixture recorder or generator that reaches a live service.
- Any suite that needs a service the sandbox has no access to. (Historical example from the
  ShelfSprite Python era: `tests/conftest.py` imported FastAPI's `TestClient`, which put the *entire*
  pytest suite off-limits, not just the API tests. The shape recurs; the specific file will not.)

**Budget the task to ~10 minutes of wall clock, including Codex's own exploration.** `--background`
is a Claude-side flag that never reaches `task`; the underlying job runs foreground with a hard cap.
Prefer several narrow dispatches over one broad one. After any killed run, check `git status`:
**applied file changes are not rolled back.**

Budget Codex's *exploration*, not its verification. When a run is slow, measure the gates yourself
before theorising — a 22-minute `verifying` phase was once attributed to a slow suite that measured
under four minutes, and the real cause is still unknown. An invented cause teaches every future
reader to over-budget every dispatch.

---

## 1. Research / inventory run

Read-only. Use before committing to any scope.

```
Read-only investigation. Do not edit any file.

Context: [2-3 sentences. Where the project is, what already shipped, what this is for.]

Produce a precise inventory I can turn into an implementation plan:

1. [deliverable]
2. [deliverable]
...
N. How an already-working example is structured end to end: pick [KNOWN GOOD EXAMPLE] and list
   every file involved, the layering convention, where the transaction wrapper goes, and how its
   tests are wired.
N+1. Anything in scope with NO equivalent yet that would need new infrastructure.
N+2. Search ALL of [TARGET DIR] and list every [item] in scope that ALREADY exists there, with its
   module path and exported name. Do not restrict the search to the modules I named elsewhere in
   this prompt.

Output as a structured markdown report with file:line citations throughout. Be exhaustive on
specifics, do not summarize away detail. Do not propose a design or write code.
```

Why each line matters:

- **`Do not propose a design or write code`** is load-bearing. Without it you get an opinion where
  you needed facts.
- **Numbered deliverables** produce inventories; vague asks produce summaries.
- **`file:line` citations** make the output usable as plan input instead of something to re-verify.
- **The known-good-precedent item** surfaces layering conventions in reusable form — the single
  highest-value item in the first inventory that used it.
- **The infrastructure-gap item** catches "this isn't a port, it's a design problem" before you plan
  it as a port.
- **The already-exists item (N+2)** catches the inverse. One draft plan told the executor to
  implement `difflib.SequenceMatcher` from scratch — a hard algorithm an earlier wave had already
  shipped, alongside three normalizers. Everything the draft said about the *source* language was
  true; it just did not know what the *target* side already held. **Phrase it as a search across the
  whole target directory, not a question about the modules you named** — naming modules elsewhere in
  the prompt is precisely what confines the search to them.

Persist the result into the repo immediately — `result <job-id> > docs/.../inventory.md`. The
execution session has no memory of the conversation that produced it.

**Let the inventory land before committing to scope.** It has overturned the assumed scope in every
wave that ran one.

---

## 2. Plan-drafting run

```
Write ONE new file: [PATH]
Do not modify any other file. No source changes.

Follow the EXACT structure and level of detail of [EXISTING PLAN PATH] — read it first. Its
section order is: [list sections]. Match its voice and density: real code in the steps, exact
file paths.

Source material already in the repo:
- [inventory path] — [what's in it]
- [source-of-truth files]
- CLAUDE.md — project conventions

SCOPE — this wave is [X] ONLY:
- [what's in]
- [what's out] is wave [Y]. Do NOT plan it. State the boundary in Global Constraints.

Requirements the plan must encode:
1. [requirement] — because [what went wrong last time it wasn't done].
...

WRITE THE TEST BODIES. Every test must be complete runnable code against real schema columns —
not a describe block with comments explaining what to assert. Assertions compare whole objects
so that supersets fail.

PROOF-OF-FIX CHECKS: when a step must prove something landed, cite `grep -n '<exact code>'`, never
`sed -n 'N,Mp'`. Line numbers drift between plan-authoring and plan-execution; symbols do not.

Record what you actually observe, not what this plan predicts.

This doc gets executed by an agent in a fresh session with no memory of this conversation.
```

Rules paid for in defects:

- **`WRITE THE TEST BODIES`** — Codex reliably writes complete implementation code and reduces test
  code to prose-in-comments. Asking explicitly fixes it entirely.
- **`grep -n` not `sed -n`** — one plan's gate cited a `542,551p` window for a guard that had moved
  to line 552. A literal reading would have failed a gate that actually passed.
- **`Record what you actually observe`** — a plan's "Verified Facts" drift benignly between authoring
  and execution (test counts, file counts, commits ahead). Without this line an executor silently
  reconciles the difference; with it, the drift is cheap and visible.

Encode past failures **with the reason attached**. "Do X" gets followed loosely; "do X because last
time not doing it left the repo with a failing suite" gets followed precisely.

---

## 3. Plan-review run (before any execution)

Read-only, and the highest value-per-minute Codex has produced. A plan asserts dozens of checkable
facts in one place, and checking them is pure lookup with no judgement required — exactly what a
budget-boxed run is good at. One pass returned four findings, four real, including two symbols that
did not exist and would have failed at the first `tsc` mid-wave, after a dispatch had already burned
its budget writing against a fictional API.

```
Read-only review. Do not edit any file.

Review [PLAN PATH] against the real repo. It asserts these symbols and signatures:
  - [symbol] in [file]
  - [symbol] in [file]
  ...
Report any that do not exist, have a different signature, or live somewhere else.

Also check behavior, not just symbols: where the plan's code claims to reproduce existing behavior
in [reference file/function], compare them and report any divergence.

Routing checks (when the plan has any `**Routing:**` block or a "Routing snapshot" heading): fail a plan
that has Routing blocks and no snapshot, or a snapshot and no blocks; a task tagged class mechanical or executor tier cheap whose text needs a design choice;
an executor `inline` on a task that creates a file or touches more than two; a reviewer tier below the
executor tier on a task the plan calls load-bearing; a Gemini route without `cap <tokens>`; more than a
third of tasks inline; any route id not in core/routes.json of the registry version the snapshot names.

[REGISTRY: absolute path to agent-config's core/routes.json at the snapshot's registry version, or its route id -> tier list pasted here]

If a claim is correct, do NOT list it. I want an actionable list, not an audit.

[standing rules block]
```

A by-hand plan-seam review writes one `seam-review` record, not `reviewer`, per agent-config's `skills/routing/references/seams.md` ("The record").

- **Name the symbols explicitly.** Not "review this plan" — the findings map one-to-one onto the
  checklist items you provide.
- **`If a claim is correct, do not list it`** turned a forty-item audit into a four-item list.
- **Ask for behavior as well as symbols.** The same pass caught a parity bug no fixture could: the
  reference implementation stripped its inputs before storing; the plan's code trimmed only inside
  the guard and stored the untrimmed value.
- **Flag your own uncertainty in the plan first.** A plan that already says "find X's real signature
  — it may be `(db, userId)` or `(userId)`" gets a better review than one that hides its
  uncertainty. Codex closes exactly the gaps you advertise.

---

## 4. Task-execution run

One task at a time, task text lifted verbatim from the plan.

```
Execute Task [N] of [PLAN PATH] exactly as written. Read the plan's Global Constraints first.

[PASTE THE TASK TEXT VERBATIM]

[standing rules block]

VERIFICATION — run ALL of these, regardless of what the task step lists:
  npx vitest run [the specific test files]     # or the runner that actually owns those paths
  npm run type-check
  npx eslint [files you touched]
  npx prettier --write [files you touched]

Expected to be RED before your fix, by test name: [name them individually].

When done, report: files touched, commands run with their actual output, and any place the plan
disagreed with the real repo.
```

After a by-hand dispatch (outside a routing-skill controller run, which writes its own records), write one `executor` or `reviewer` record: `agent-routing record -` with the fields and roles in agent-config's `skills/routing/references/execute-time.md` ("The ledger"); the writer exits 2 and says why on a malformed record.

**Paste that verification block verbatim, then add task-specific gates to it.** Editing down from
the template is safe; composing up from memory is not. One dispatch that named three sensible gates
dropped `prettier` — all three named gates passed, the work was correct, and the stop-hook failed
the turn on formatting. The gate was in §4's VERIFICATION block the whole time.

Rules in that block, each paid for:

- **Codex runs the commands you list and no others.** An unlisted gate is out of scope, not
  forgotten. `type-check` and `lint` once appeared only in a plan's final task, so Task 1 shipped
  green tests and a broken `tsc` **twice**. Deferring type-check to the last task means errors pile
  up silently and land on the task least able to attribute them.
- **State expected-red by test NAME, never by count.** "Expect exactly 2 failures" was wrong (the
  real number was 4). A count drifts with test topology, and a wrong one teaches the executor to
  ignore the gate.
- **Allow for the thing not being there.** Told to fix wording in a doc where a grep found nothing,
  asking Codex to *locate it and report plainly if absent* surfaced the real wording the grep pattern
  had missed — instead of an invented edit or a silent skip.
- **Only fast, scoped gates belong here.** A slow whole-repo suite buys nothing (the controller
  re-runs it regardless) and can eat the task. Excluding a slow gate and running it yourself is fine;
  forgetting a fast one is not.

### Before you send it: verify every gate command matches something

Three separate waves shipped a gate command that matched **zero tests, exited 0, and read as a
pass** — a plan-named vitest path that only jest owned, a `.ts` glob against a `.tsx` file, a
component path outside vitest's `include`. **The dangerous gate failures produce no output rather
than a red X.**

```bash
npx jest --listTests <path>     # jest: does it claim this file?
npx vitest list <path>          # vitest: does it claim this file?
```

One command each, and check `package.json`'s `scripts` block once at the start of a wave — `npm test`
reads as "the test suite" and in this repo it is jest only, a quarter of one.

**`| tail -N` is not a gate.** It is a display convenience that silently depends on how chatty the
tool's epilogue is, and upgrades move that boundary. A `pytest -q 2>&1 | tail -3` stopped showing
the pass count when a version change emitted the warnings summary last — three lines of warnings,
zero exit, no number, and the executor records a pass having never observed the count. Any gate whose
purpose is to produce a *number* must extract the number (`grep`, `--json`), not slice the tail.

---

## 5. Techniques that actually find defects

One wave turned up **six plan defects and four helper-level divergences. Every one was found by
running something; not one by reading code.** Ranked by yield:

1. **Differential-test the port against the original.** Run the new implementation and its
   counterpart over a shared input matrix and diff the outputs. This found four divergences
   (unpadded dates, mixed date separators, hex/octal/binary literals, underscore separators) that
   **11 passing hand-written tests had missed**, and separately proved a JSON escaper byte-identical
   to `json.dumps` across 22 cases. Mechanical, cheap, and it doesn't depend on guessing which cases
   matter — which is exactly what hand-written tests get wrong.
2. **Probe the live app for every error body the task must reproduce, before dispatching.** This
   caught an invented 422 and an unstated error precedence. A plan's error *strings* are
   transcriptions; its error *ordering* is usually never written down at all.
3. **Run the actual library when the plan asserts its byte-level output.** A CPython check found a
   wrong golden in two places before any code existed.
4. **When a port has a live original, diff both against one shared database.** Two implementations
   run against the same container came out byte-identical in both formats — with real non-ASCII rows
   that ASCII-only fixtures could never have exercised. No recording step, no masking.
5. **Mutation-test whatever the plan calls load-bearing.** Break the invariant deliberately and
   confirm something goes red. One such check revealed that all four fixture replays still passed
   under the injected bug — the whole parity apparatus was blind to it, and a single hand-written
   failure-injection test was the only thing catching it. Cost: three minutes.
6. **Design fixture rows against branches, not as filler.** A row whose foreign key matches nothing
   is the only thing proving the join emits `null` instead of omitting the key; a second-tenant row
   proves the join for more than the default tenant; a 2/1/1 category split makes a filter a real
   filter. A fixture that merely has rows proves the query runs.

**The corollary is the actual rule: a green suite proves the specified cases work, not that the
implementation is faithful.** Every defect in that wave lived in an input shape the test file could
not express — a zero-data-row CSV, an unpadded date, a hex-looking cell, a non-ASCII title. When
reviewing, ask what the tests *cannot* say, then go measure that.

---

## 6. Claude's review checklist for Codex output

Run this on every returned task. This is where the workflow's errors actually get caught.

- [ ] **Verify factual claims, don't spot-check them.** In one wave every npm script name, helper
      path, and schema export checked out — but `DbTx` did not exist, and that was invisible without
      grepping.
- [ ] **Before contradicting a citation, open the cited range.** A single-form grep is not proof of
      absence for any API with more than one spelling. One false "the plan is wrong" finding came
      from `grep -c 'references('` returning 0 while the schema declared its only FK via the
      table-level `foreignKey({...})` form. *"My grep found nothing" and "the fact is not there" are
      different claims.*
- [ ] **Never propagate an unverified review finding into later prompts.** That false FK finding was
      fed into two subsequent task prompts as established fact. Propagation is efficient when right
      and contaminating when wrong.
- [ ] **Codex's observations are reliable; its attributions need checking.** Real observation,
      confident and plausible *wrong* conclusion, three times in one wave. A job reported a generator
      "did not complete after roughly ten minutes"; it runs in **1.8 seconds** — a paged `git diff`
      had hung the shell and eaten the budget. Default expectation: **when a run reports "X is
      broken/too slow", verify X directly before believing it.**
- [ ] **Verify a STOP the same way — the finding, not the framing.** Codex halting rather than
      adapting is correct behavior and cheap (one message each). Across three stops in one wave, all
      three reports were factually accurate and two carried conclusions that did not follow ("this
      function has two callers, the plan said one" — the second caller was the unit test of the
      function being deleted). Opening the file took one command each time.
- [ ] **Re-run verification independently.** Green tests are not proof of a correct primitive: a
      `context.columns` property that does not exist on csv-parse's `InfoRecord` was `undefined` at
      runtime, silently took a fallback path, and shipped a passing suite with a wrong helper.
- [ ] **Re-review anything Codex added beyond what you asked for.** The additions are usually good,
      but they're the parts nobody specified and therefore nobody checked. One unrequested header
      block introduced a naming inconsistency.
- [ ] **Check the diff for formatting churn.** `prettier --write` on a touched file reformats
      pre-existing code; one wave saw 11 purely cosmetic deletions in a file nobody meant to touch.
- [ ] **Confirm scope.** `git status --short --untracked-files=all` — did it touch only the intended
      files?
- [ ] **When a table-driven test fails, check every row before editing one.** Of seven rating rows in
      one case table, exactly one legitimately changed. Checking made the edit obviously surgical
      rather than a guess.
- [ ] **A deliberate behavior change needs an inline comment at the expectation site.** Two import
      changes read cold as regressions (`.5` yielding `0.5` instead of `1`). Without the comment the
      next reader's instinct is to restore the old number.
- [ ] **Ledger record written.** If you have not already, after a by-hand dispatch write one `executor` or `reviewer` record with `agent-routing record -` (fields: §4).

---

## 7. Useful commands

**Canonical invocation — copy this, don't improvise it.** Both gotchas below have been hit again
*by someone who had just read them*, because each required reconstructing a command instead of
pasting one.

```bash
CX='node /home/chase/.claude/plugins/cache/openai-codex/codex/1.0.6/scripts/codex-companion.mjs'
# The version is in the path; check `ls ~/.claude/plugins/cache/openai-codex/codex/` after an update.

# Long prompt: write it to a file first. Never inline a multi-paragraph prompt into the shell.
$CX task "$(cat /path/to/prompt.txt)"            # READ-ONLY — inventory, plan-review, review runs
$CX task --write "$(cat /path/to/prompt.txt)"    # anything that must create or edit a file
$CX task --write --resume-last "<delta only>"    # continue a thread that was ALREADY --write
$CX cancel <job-id>                              # kill a stray job the moment you notice it
```

**`--write` is not a companion default, and it must be on the FIRST call.** The `codex:codex-rescue`
skill adds it; the CLI does not. Calling `task` directly to skip the forwarding layer also skips its
flag defaults — a plan-drafting run without `--write` explores, then fails with "the workspace is
mounted read-only."

**You cannot rescue that thread.** A thread's sandbox is fixed when it is created, so
`--write --resume-last` fails the same way even though the companion genuinely sends
`sandbox: "workspace-write"` on resume. A read-only thread is a dead end for any work that writes:
start fresh with `--write` and re-pay the exploration. Decide read-vs-write at dispatch, because
that is the only moment you can.

- **Never pipe a running job through `tail`/`head`** — the pipe buffers, the output file stays empty,
  and it looks exactly like a hang. Watch with `status` instead; nothing is lost either way because
  the job store has the full output. Corollary: **a timed-out Bash call kills the client, not the
  Codex job.** Before concluding anything died, check `status --json` and
  `git status --short --untracked-files=all` — one such "hang" was a live job at 10m12s with both
  file edits already applied.
- **`status --json` is unusable raw** — it echoes the entire prompt back, so one status check on a
  large task can dominate the context window. Filter it:

```bash
$CX status --json | python3 -c "
import json,sys
d=json.load(sys.stdin)
for label in ('running','recent'):
    for j in d.get(label) or []:
        print(f\"[{label}] {j['id']} {j['status']}/{j.get('phase') or '-'} \"
              f\"{j.get('elapsed') or j.get('duration') or ''} :: {(j.get('summary') or '')[:60]}\")
"
```

Verified 2026-08-10 against companion 1.0.6. Top-level keys are `running`, `recent`,
`latestFinished`, `needsReview`, `config`, `sessionRuntime`, `workspaceRoot` — **there is no
`activeJobs` key**, and `title` is always the literal string `"Codex Task"`, so filter on `summary`.
`recent` is scoped to the current session; add `--all` for earlier sessions in this repo.

- Full result payload: `$CX result <job-id> --json` → `.storedJob.result.rawOutput`
- Persist a result: `$CX result <job-id> > docs/.../<name>.md`
- Resume a thread days later: `codex resume <session-id>` (session IDs are in the run log)
- `task --help` is **not** a help flag — `task` forwards its whole argument string as the prompt.
  Read `~/.claude/plugins/cache/openai-codex/codex/<version>/commands/*.md` for flags.
- **Record job IDs with the artifact.** Each run prints a Codex session ID and a
  `codex resume <session-id>` line — that is how you continue a thread days later instead of
  re-paying for the exploration.

### Cheap read-only checks that retire a dispatch

Before writing a prompt, ask whether one command answers the question:

- `git merge-tree --write-tree HEAD <branch>` settles "will this conflict?" without touching the
  working tree. One wave reserved a dispatch for a possible merge conflict; this retired it before
  the prompt was written.
- `git ls-tree <deploy-branch> <migrations-dir>/` against the deployed revision catches a migration
  applied to production from a feature branch — a landmine with no immediate symptom, since the
  running container booted fine and only a *restart* trips it.
