---
name: setup-pstack
description: Configure which model or Orca agent pstack uses per role. Detects the available subagent models and Orca agents and writes an always-applied rule that overrides the skill defaults. Use for /setup-pstack, "configure pstack models", or changing pstack's model choices.
---

# Setup pstack

Write `~/.claude/rules/pstack-models.md`, an always-applied user rule that sets pstack's model per role.

Roles come in two kinds, and the kind fixes what a value may be.

- **Subagent roles** run in-process through the `Agent` tool. The value is a model alias that tool accepts, or `inherit-parent`.
- **Worker roles** run as Orca workers through the `orchestration` skill: `arena runners`, `arena cross-judge pool`, `architect runners`, `interrogate reviewers`, and `swarm workers`. The value is `<agent>`, `<agent>:<model>`, or `<agent>:<model>:<effort>`, passed to `ORCA orchestration worker-start` as `--agent`, `--model`, and `--effort`. A bare `<agent>` runs on that agent's own configured model. `ORCA` is the executable the `orca-cli` skill resolves.

## Steps

### 1. Detect what is available

- Subagent models: the values the `Agent` tool's `model` parameter accepts in this session.
- Orca agents: `ORCA status --json` must report a reachable runtime. An agent counts as available when its CLI is on `PATH` (`command -v claude codex`, plus any other agent id `ORCA orchestration worker-start --help` names).

If you cannot detect either list, ask the user to paste it. Never write a model alias or agent id you have not confirmed. `inherit-parent` and `auto` are always valid for a subagent role.

### 2. Load current state

The default role-to-value mapping is the rule shape shown in step 5 below. If `~/.claude/rules/pstack-models.md` already exists, read it and treat its role values as the current choices. Otherwise start from those defaults. A line whose role is not in step 5 is from a retired role. Drop it.

### 3. Map and confirm

Show every role with its value, marking any value not in the detected set as needing a choice. Also list each line step 2 dropped. Ask whether to accept as-is or change specific roles. Prefer `AskUserQuestion` over free text. Offer the detected models plus `inherit-parent` for a subagent role, and the detected agents for a worker role.

For panel roles (arena runners, architect runners, interrogate reviewers) the value is a list, and one worker runs per entry, so the list length sets the count. A panel earns its signal from different model families, so keep at least two agents in each list when two are available. `arena cross-judge pool` is also a list, but Arena selects one entry from it whose agent differs from the parent's when possible. `swarm workers` is the default for every worker unless a race or comparison assigns another agent per arm.

### 4. Validate

Every model alias and agent id written must be in the detected set. `inherit-parent` and `auto` always pass. If a chosen value is not available, stop and ask again.

### 5. Write the rule

Write `~/.claude/rules/pstack-models.md` with one line per role, using the same labels poteto-mode uses. Overwrite the whole file so re-runs stay idempotent. Shape:

```
# pstack model configuration. One line per role. Delete a line to fall back to the skill default.
# Subagent roles take an `Agent` model alias. `inherit-parent` or `auto` runs the role on the parent chat model (omit Agent `model`).
# Worker roles take Orca launch entries, `<agent>[:<model>[:<effort>]]`, started through the `orchestration` skill.
# pstack skills set `disable-model-invocation`, so load one by reading `~/.claude/skills/<name>/SKILL.md`, not through the `Skill` tool.
feature, refactoring: sonnet
bug-fix: sonnet
perf-issue: sonnet
hillclimb: sonnet
judgment and prose: inherit-parent
hardest tasks: inherit-parent
how explorer: sonnet
how explainer: inherit-parent
why investigators: sonnet
why synthesizer: inherit-parent
reflect tooling: inherit-parent
reflect judgment, divergent, synthesizer: inherit-parent
arena runners: claude, codex
arena cross-judge pool: claude, codex
swarm workers: claude
architect runners: claude, codex
interrogate reviewers: claude, codex
```

### 6. Confirm

Tell the user the rule was written and that it applies to new sessions. Re-running this skill updates it.

### 7. Offer a verification skill (optional)

Check whether the project has a way to drive the real app for proof (a `verify-*` skill, or an existing harness). If not, offer once: "want a project-local verification skill, so agents can drive the app the way a user does and prove changes work? I can generate one with /create-verification-skill." On yes, read `~/.claude/skills/create-verification-skill/SKILL.md` and follow it. On no, move on without pushing.
