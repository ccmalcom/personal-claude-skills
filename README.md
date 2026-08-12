# personal-claude-skills

A Claude Code plugin marketplace for skills that are general engineering technique rather than
personal setup.

It's one of three repos:

- [claude-config](https://github.com/ccmalcom/claude-config) — `settings.json`, global `CLAUDE.md`,
  hooks, auto-memory, and personal-workflow skills that reference my own vault and repos.
- [claude-knowledge-vault](https://github.com/ccmalcom/claude-knowledge-vault) — the notes those
  skills read and write.
- **this repo** — skills that are general technique, packaged as a marketplace.

The split is **personal identity and machine config there, knowledge in the vault, shareable
technique here**, because a plugin can carry skills, agents, commands, and hooks, but cannot carry
your settings or your `CLAUDE.md`. Nothing here references my vault or my repos, which is what
makes it installable by anyone.

## Install

```
/plugin marketplace add ccmalcom/personal-claude-skills
/plugin install chase-workflow@chase-skills
```

## What's in it

### `chase-workflow`

Two skills for plan-driven agentic work, both derived from measuring a real multi-task execution
(11 tasks, 273 controller turns, 6.23M input-equivalent units) rather than from intuition.

**`controller-budget`** — a controller session's cost is context floor × turn count, and nothing
else moves the needle. Covers the cost model, the hand-off-every-2–3-tasks rule, why idle time is
the worst single waste, and a list of optimizations that were measured and found irrelevant so
they don't get re-litigated.

**`codex-dispatch`** — how to write a Codex brief that survives being read literally, which gates
to name (fast and scoped, never a slow whole-repo suite), why a fresh dispatch beats a resume when
the round requires an edit, and how to verify what comes back.

## These layer on superpowers; they don't replace it

Plugin skills are namespaced, so shipping a skill named `subagent-driven-development` here would
give you *two* of them rather than an override. Forking the upstream skill would also mean
maintaining a 500-line copy that silently drifts on every superpowers release.

So these are additive instead. Keep using superpowers' `writing-plans`, `executing-plans`, and
`subagent-driven-development`; these two skills add the cost discipline and delegation patterns on
top and stay small enough to survive upstream changes.

## Provenance

Every threshold here has a measurement behind it. Where a plausible-sounding rule was tested and
turned out to be wrong, that is recorded too — `controller-budget` ends with four such cases,
including two I was confident about. Treat the numbers as evidence from one codebase's workflow,
not universal constants; re-measure before porting a threshold somewhere very different.
