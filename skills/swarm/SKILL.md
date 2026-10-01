---
name: swarm
description: "Fan out N parallel workers, drain them, and return one report. Use for /swarm, 'swarm this', or parallel coverage, races, gauntlets, and exploration."
disable-model-invocation: true
---

# Swarm

Fan out N parallel Orca workers. They may cover separate slices, race the same brief, or mix both. The parent waits, aggregates, and returns one report.

## Start

Open a todolist with one entry per phase before launching anything.

1. Frame
2. Fan out
3. Aggregate
4. Report

## Phase A: Frame

1. State the done predicate and the artifact or report the swarm must return.
2. Choose the shape. Partition into slices, race N workers on identical briefs, or mix both. For a race or mixed shape, declare `first pass`, `rank all`, or `best-of` before spawning.
3. Set N from the user or derive it from the shape. N is total workers, not a concurrency limit.
4. Pick the worker agent from the `swarm workers` line in `~/.claude/rules/pstack-models.md`. The value is an Orca launch entry of the form `<agent>[:<model>[:<effort>]]`. If the rule or that line is missing, use `claude`. If `worker-start` rejects an entry, use the default and say so. For a model race, name each arm's entry up front.
5. Give each worker its own writable output when it writes. When workers verify or measure commits, each brief names the exact SHAs. A measurement brief also names the method (sample count, what one sample is, order). The worker records both in its result.

## Phase B: Fan out

Load the `orchestration` skill and run its supervised loop as coordinator. Create one Run, then start all N workers before the first wait, one call each. `ORCA` is the executable that skill resolves.

```text
ORCA orchestration worker-start --spec "<brief>" --agent <agent> --worktree <placement> --json
```

Add `--model` and `--effort` when the step 4 entry names them. A worker that writes gets its own worktree with `--worktree new-child --name <slug>-<n>`. A worker that only reads, verifies, or measures uses `--worktree current`. When a worker must start from a non-default pushed branch, pass `--base-branch <ref>`.

If Orca's runtime is not reachable, run the N workers as in-process `Agent` subagents on the parent model and say so in the report.

Every brief stands alone. Include the goal, scope, exact slice or race arm, how to verify, and what to report. Each brief names its own report file, `/tmp/swarm-<slug>/worker-<n>.md`, and the worker passes that path as `--report-path` on `worker_done`. Reports use `PASS`, `ISSUES`, or `BLOCKED` with evidence. A worker that can prove a defect reports `ISSUES` and lists every issue it can prove, not only the first.

If a worker drops out, proceed with N-1 and note it.

## Phase C: Aggregate

Wait until every Dispatch settles, release each settled worker, then read the report files. Drop a result that does not record the SHAs and method its brief names, and rerun that worker once. After a second miss, record a gap. A gap does not count as a pass. For coverage, every required slice needs a result. For a race, apply the selection rule declared up front. Use first pass, rank all, or best-of. Do not paste raw worker dumps.

Keep a compact result table, one-line evidenced issues, and explicit gaps or dropouts.

## Phase D: Report

Return one consolidated in-chat report with the table, issue one-liners, gaps or dropouts, and the race rule when used.
