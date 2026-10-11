---
name: interrogate
description: "Use for \"interrogate\", \"adversarial review\", \"multi-model review\", \"challenge this\", \"stress test this code\", \"find blind spots\", or \"tear this apart\". Multiple LLM reviewers challenge changes from independent angles."
disable-model-invocation: true
---

# Interrogate

Spawn one reviewer per configured model to adversarially review code changes. The adversarial signal comes from model diversity, not assigned personas. The deliverable is a synthesized verdict. Do NOT auto-apply changes.

## Step 1, Scope

Identify what to review from context:

- If the user points at specific files or a diff, use that.
- If on a feature branch, run `git diff main...HEAD` (or the appropriate base branch) for the full changeset.
- If the user's message references recent work, gather the relevant files.

Package the diff or file contents with any surrounding context files the reviewers need to understand the code.

## Step 2, Intent

Before spawning reviewers, state the intent in one clear paragraph from the user's message, commit messages, the PR description if one exists, and the code. If you're unsure about the intent, ask the user before proceeding.

## Step 3, Spawn Reviewers

Run the reviewers as Orca workers, so each one is a different agent CLI on a different model family. Use the `interrogate reviewers` line in `~/.claude/rules/pstack-models.md`, one reviewer per entry, extending or shrinking the Reviewer A/B labels below to the configured entry count. Each entry is an Orca launch entry of the form `<agent>[:<model>[:<effort>]]`. If the rule or that line is missing, use the table defaults.

| Worker | Default agent |
|----------|---------------|
| Reviewer A | `claude` |
| Reviewer B | `codex` |

Read `references/reviewer-prompt.md` and fill in the same template for every reviewer with the stated intent, the diff or file contents, the rubric from `references/rubric.md`, and the code-quality lens from `references/code-quality-review.md`. Create a fresh `<run-dir>` with `mktemp -d`, under the session scratchpad when the harness names one and under `/tmp` otherwise, and write the filled template to `<run-dir>/prompt.md`.

Load the `orchestration` skill and run its supervised loop as coordinator. Create one Run, then start every reviewer before the first wait, one call each. `<label>` is the reviewer's letter from the table (`A`, `B`, ...), `<target>` is the absolute path of the repo or folder under review, and `ORCA` is the executable that skill resolves.

```text
ORCA orchestration worker-start --spec "Target: <target>, read-only. Read <run-dir>/prompt.md and follow it. Review only, edit nothing. Write your findings to <run-dir>/reviewer-<label>.md and pass that path as --report-path on worker_done. Done when that file holds a '## Findings' section or the words 'no findings'." --agent <agent> --worktree current --json
```

Add `--model` and `--effort` when the entry names them.

### Count each reviewer

A reviewer counts when its `worker_done` carries `outcome: succeeded` and the `reportPath` in its payload names a file holding a `## Findings` section or `no findings`. Check this as each Dispatch settles, then release the worker. A reviewer that misses is dropped: a missing report is never an empty review. Continue once every Dispatch has settled, with the reviewers that count.

### When a reviewer cannot start

The two outputs of a non-zero `worker-start` take different paths:

- **Refused**: the output is an `error.code` with no `failedStage` or `residualResources`, so no worker exists. Run that reviewer on bare `claude` when no other reviewer is on `claude`, and drop it otherwise.
- **Failed or unknown**: the output names `failedStage` or `residualResources`, so a worker may still be running. Follow the orchestration skill's `references/recovery-and-cleanup.md`. The reviewer settles there or is dropped, and its seat stays empty.

When no Orca worker can start at all, because `ORCA status --json` fails before the first launch or every entry is refused, run one in-process `Agent` subagent per entry on the parent model with the same prompt and file contract.

Name every dropped or substituted reviewer in the verdict's Reviewers list. When two reviewers share a model, say that the review lost model diversity and count their agreement as one voice in Step 4.

## Step 4, Synthesize

As results come back, build a unified picture. Parse every reviewer's findings. Merge findings that describe the same issue differently, and note which models raised each one. Findings two or more models raised independently are the highest signal. Read a lone model's finding, but weight it accordingly. Note disagreements. If one model flags something and another explicitly says the opposite, that is context for the verdict.

## Step 5, Lead Judgment

You are the lead reviewer, a pragmatic senior engineer, not a neutral aggregator. Read `references/lead-judgment.md` for the full framework. Put every finding in one of four buckets, with the model(s) that raised it and a one-line rationale.

- **Act on**. Real issues affecting correctness, security, or maintainability given the actual goals. They would block a real PR.
- **Consider**. Legitimate, but you're not sure they outweigh the cost of addressing them now. Worth the user's attention.
- **Noted**. Technically valid but not actionable. Context-dependent, premature optimization, or low-impact at this stage.
- **Dismissed**. Wrong, nitpicky, or missing context, with a brief reason.

## Output Format

### Intent
> [The stated intent paragraph from Step 2]

### Reviewers
- Reviewer [label]: [agent and model], [N findings] (one bullet per reviewer; a dropped one reads `dropped, [reason]`)

### Act On
[Each: description, which models raised it, why it matters.]

### Consider
[Each: description, which models raised it, the tradeoff.]

### Noted
[Brief list.]

### Dismissed
[Each with a brief rationale.]

### Agreement Map
[Where models agreed and diverged, and what that pattern tells us.]
