---
name: interrogate
description: "Use for \"interrogate\", \"adversarial review\", \"multi-model review\", \"challenge this\", \"stress test this code\", \"find blind spots\", or \"tear this apart\". Multiple LLM reviewers challenge changes from independent angles."
disable-model-invocation: true
---

# Interrogate

Spawn one reviewer per configured model to adversarially review code changes. Each model gets the same prompt and rubric. The adversarial signal comes from model diversity, not assigned personas.

The deliverable is a synthesized verdict. Do NOT auto-apply changes.

## Step 1, Determine Scope

Identify what to review from context:

- If the user points at specific files or a diff, use that
- If on a feature branch, run `git diff main...HEAD` (or the appropriate base branch) for the full changeset
- If the user's message references recent work, gather the relevant files

Package the diff (or file contents) plus any surrounding context files the reviewers need to understand the code.

## Step 2, State the Intent

Before spawning reviewers, state the intent explicitly. Derive this from:

- The user's message
- Commit messages
- PR description if one exists
- The code itself

Write one clear paragraph. If you're unsure about the intent, ask the user before proceeding.

## Step 3, Spawn Reviewers

Run the reviewers as Orca workers, so each one is a different agent CLI on a different model family. Use the `interrogate reviewers` line in `~/.claude/rules/pstack-models.md`, one reviewer per entry, extending or shrinking the Reviewer A/B labels below to the configured entry count. Each entry is an Orca launch entry of the form `<agent>[:<model>[:<effort>]]`. If the rule or that line is missing, use the table defaults.

| Worker | Default agent |
|----------|---------------|
| Reviewer A | `claude` |
| Reviewer B | `codex` |

Read `references/reviewer-prompt.md` and fill in the template with:
1. The stated intent
2. The diff or file contents
3. The review rubric from `references/rubric.md`
4. The code-quality lens from `references/code-quality-review.md`

The same filled template goes to all reviewers, so every model applies the code-quality lens. Create a fresh `<run-dir>` with `mktemp -d`, under the session scratchpad when the harness names one and under `/tmp` otherwise, and write the filled template to `<run-dir>/prompt.md`.

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

As results come back, build a unified picture:

1. **Parse all findings** from the reviewers
2. **Identify consensus**. Findings raised by 2+ models independently are highest signal.
3. **Identify lone-model findings**. Still worth reading, but weight accordingly.
4. **Deduplicate**. Different models may describe the same issue differently. Merge these and note which models raised it.
5. **Note disagreements**. If one model flags something and another explicitly says the opposite, that's useful context for the verdict.

## Step 5, Lead Judgment

You are the lead reviewer, a pragmatic senior engineer, not a neutral aggregator.

Read `references/lead-judgment.md` for the full framework.

Categorize every finding using these buckets:

- **Act on**. Real issues affecting correctness, security, or maintainability given the actual goals. These would block a real PR.
- **Consider**. Legitimate points, but you're not sure they outweigh the cost of addressing them right now. Worth the user's attention.
- **Noted**. Technically valid but not actionable. Context-dependent, premature optimization, or low-impact given the current stage.
- **Dismissed**. Wrong, nitpicky, or missing context. Brief explanation why.

For each finding, include:
- Which model(s) raised it
- The category (act on / consider / noted / dismissed)
- A one-line rationale for the categorization

## Output Format

Present the verdict in this structure:

### Intent
> [The stated intent paragraph from Step 2]

### Reviewers
- Reviewer [label]: [agent and model], [N findings] (one bullet per reviewer; a dropped one reads `dropped, [reason]`)

### Act On
[Findings that should be addressed. For each: description, which models raised it, why it matters.]

### Consider
[Findings worth thinking about. For each: description, which models raised it, tradeoff involved.]

### Noted
[Valid but low-priority. Brief list.]

### Dismissed
[Rejected findings with brief rationale.]

### Agreement Map
[Where did models agree, where did they diverge, and what does the pattern of agreement/disagreement tell us?]
