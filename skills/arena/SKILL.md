---
name: arena
description: "Spawn N parallel candidates at the same task, pick a base, graft the strongest parts of the losers into it. Use for /arena, 'arena this', 'throw it in the arena', or when one attempt at a non-trivial artifact would lock in the wrong shape."
disable-model-invocation: true
---

# Arena

Open a todolist with one entry per phase (Frame, Fan out, Cross-judge, Pick, Graft, Verify) before launching anything.

## Phase A: Frame

Every candidate gets the same prompt, so the prompt is the contract.

1. State the artifact each candidate is producing.
2. Derive the rubric, 3-6 concrete gradeable criteria for what success looks like on *this* task. It is the picker's tool in Phase D. Candidates only see the task.
3. Pick the runners. Use the `arena runners` line in `~/.claude/rules/pstack-models.md`. Each entry is an Orca launch entry of the form `<agent>[:<model>[:<effort>]]`. If the rule or that line is missing, default to one each on `claude` and `codex`. If `worker-start` rejects an entry, run that seat on `claude` and say so. Spawn more when the arena covers multiple design directions. Same agent N times when the work is generation-bound rather than judgment-sensitive.
4. Assign output paths. Each candidate writes to its own location (its own Orca worktree when the artifact is code in the repo, otherwise `/tmp/arena-<slug>/candidate-<n>/`), per the **separate-before-serializing-shared-state** principle skill.

## Phase B: Fan out

Load the `orchestration` skill and run its supervised loop as coordinator. Create one Run, then start all N candidates before the first wait, one call each. `ORCA` is the executable that skill resolves.

```text
ORCA orchestration worker-start --spec "<prompt>" --agent <agent> --worktree <placement> --json
```

Add `--model` and `--effort` when the entry names them. A candidate that writes code in the repo gets `--worktree new-child --name arena-<slug>-<n>`. A candidate that writes under `/tmp` uses `--worktree current`. Each prompt carries the task, the path to the shared grounding, its own output path, and instructions to produce the artifact and a short rationale naming the alternatives it considered and what it rejected.

If Orca's runtime is not reachable, run the candidates as in-process `Agent` subagents on the parent model and record in the synthesis note that the arena lost its model diversity.

If a candidate produces no output, proceed with N-1 and note the dropout.

## Phase C: Cross-judge

After all Phase B candidates settle, release their workers and choose one entry from the `arena cross-judge pool` line in `~/.claude/rules/pstack-models.md`. If the rule or that line is missing, choose from `claude` and `codex`. Prefer a different agent from the one you run as. Start one judge worker on that entry in the same Run with `--worktree current`, and tell it to edit nothing. It sees the rubric and the candidates by path label, scores each criterion, and recommends a base with rationale. It runs in parallel with the parent's reading in Phase D, not with the candidates themselves. Don't spawn the judge while candidates are still writing.

## Phase D: Pick a base

Read every candidate end to end before picking. Score each against the rubric criterion by criterion, not on holistic feel, and compare with the cross-judge. Agreement on the base confirms the pick. Disagreement means one of you is biased or the rubric was ambiguous. Read both rationales before deciding.

Pick the base a future maintainer can extend most easily without breaking invariants. When two feel tied, prefer the cleaner boundary or smaller API, per the Laziness Protocol.

## Phase E: Graft

Walk each losing candidate once more for what is worth porting into the base, usually one or two things per candidate, not most of it. Fold each graft in by hand, per the **redesign-from-first-principles** principle skill, so the result stays coherent under one mental model. Don't paste mechanically.

If the candidates converge on one shape, that is strong agreement. Note it and ship the consensus shape. No graft is needed. If they wildly diverge, Phase A was under-specified. Reframe and re-run rather than averaging the divergence.

## Phase F: Verify

Verify the synthesized artifact like any other output, per the **prove-it-works** principle skill. If verification finds a problem the arena missed, either Phase A was wrong (re-frame and re-run) or a candidate caught it and you missed the graft (go back to Phase E). Don't paper over it.

After verification, remove the candidate worktrees the arena created with `ORCA worktree rm`.

## Outputs

One synthesized artifact, with a short synthesis note alongside naming the base and why, the cross-judge's verdict, each graft with its source candidate, what was rejected and why, any convergence or dropouts, and the verification result.
