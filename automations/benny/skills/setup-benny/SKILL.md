---
name: setup-benny
description: Configure Benny and prepare its triage and repro Orca automations. Use when installing Benny or changing its Slack, tracker, repository, routing, control, schedule, model, or budget settings.
disable-model-invocation: true
---

# Set up Benny

Benny ships as a dormant automation pack inside pstack. This file and the two operational files are not slash skills.

The human enters setup by pointing an agent at the pack's `FOR_AGENTS.md`. The bootstrap flow copies the whole pack into the target repository, then reads this file directly at `.claude/automations/benny/skills/setup-benny/SKILL.md`.

Benny needs external configuration and two live Orca automations. Orca automations run on a schedule, so each run polls the source Slack channel instead of waking on a Slack event.

Do not create or update an automation until the user explicitly asks. Never put a secret value in pack files, prompts, or committed configuration.

## 1. Copy the pack and check the shared pstack skills

Do this before asking for Benny configuration and before creating any automation.

Ask which repository will run the automations. The source pack is the directory containing `FOR_AGENTS.md`. The destination is `<target-repository>/.claude/automations/benny/`.

Merge the entire source pack into the destination:

1. Create the destination when it is absent.
2. Copy every source file to the same relative path.
3. Preserve destination-only files. Never delete unrelated files during install or refresh.
4. Keep user-owned configuration, feature maps, and routing maps outside the destination. Never overwrite them.
5. When an existing source-managed file differs, inspect the diff and merge without discarding local edits. If ownership is ambiguous, stop and ask before replacing it.
6. Verify that the destination contains `FOR_AGENTS.md`, this setup file, both operational files, their references, and the templates.

If this file is already being read from the target destination, treat the copy as complete and run the same verification before continuing.

An Orca automation runs as the local user, so Benny reads pstack's shared skills from that user's install. On the machine that will run the automations, verify that `~/.claude/skills/<name>/SKILL.md` exists for each of these:

- `how`
- `why`
- `tdd`
- `unslop`
- `principle-separate-before-serializing-shared-state`
- `principle-minimize-reader-load`
- `principle-guard-the-context-window`
- `principle-sequence-verifiable-units`
- `principle-fix-root-causes`
- `principle-prove-it-works`

pstack skills set `disable-model-invocation`, so a run loads one by reading that file, not through the `Skill` tool.

If any shared dependency is missing, stop and explain the failure.

The Benny files are read directly from `.claude/automations/benny/`. Do not link that directory into a skills directory or expect its `SKILL.md` files to appear in the slash-skill list.

Tell the user that `.claude/automations/benny/` and any referenced secret-free configuration must be committed before either automation is enabled. Do not commit them unless the user asks.

Once this check passes, live automation prompts may read the committed operational files by their stable repository-relative paths. They must not embed a pstack install path or copy the file contents.

## 2. Adapt the configuration

Open these copied examples:

- `../../templates/configuration.example.yaml`
- `../reproduce-and-fix-issues/references/feature-map.example.md`

Create user-owned copies outside `.claude/automations/benny/`. These are configuration files, not pack files. Example locations:

- Project config, such as `.claude/benny/configuration.yaml`
- Project feature map, such as `.claude/benny/feature-map.md`
- Project routing map, such as `.claude/benny/routing.md`
- User config, such as `~/.config/benny/configuration.yaml`
- User feature map, such as `~/.config/benny/feature-map.md`

Fill one feature-map section for every user-facing feature the automation may reproduce. Keep it at the user point of view. Do not freeze implementation details or current code paths in the map.

Do not edit the copied examples. Pack refreshes may update source-managed files after conflict review, but they must never touch the user-owned copies.

Prefer committed, secret-free files in the target repository, because the repro automation runs in a fresh Orca worktree. Otherwise paraphrase the required values into the live prompt. Reference a repository file only after you confirm with `git ls-files` that it is committed on the automation's base branch.

Use stable repository-relative paths for committed pack and configuration files. Never reference the pack's source directory or a pstack install path from a live automation.

## 3. Fill the required choices

Ask for or confirm:

- Source Slack channel ID
- Optional operations or status channel ID
- Repository URL and default branch
- Triage identity or Slack user ID
- Issue tracker type, team, project, labels, and intake status
- Tracker adapter skill or MCP actions
- Optional routing map path
- Required control skill name
- Required user-facing feature-map path
- Status emoji strings
- Pull request URL format
- Poll lookback and effort budgets
- Provider agent (`claude` or `codex`) and the schedule of each automation
- Subagent model for triage, repro, code work, and media review

A schedule is `hourly`, `daily`, `weekdays`, `weekly`, a 5-field cron expression, or an RRULE. Keep `budgets.poll_lookback_minutes` at several schedule intervals, so a failed run does not drop a report.

The automation itself runs on the provider's configured model. Use only subagent model aliases the `Agent` tool accepts, or `inherit-parent`. Do not guess an alias and do not carry over a private default.

The source channel, triage identity, repository, tracker adapter, control skill, and feature map must be explicit. Fail setup if any required value stays ambiguous.

Use pstack's `unslop` skill on the final automation names and prompts before saving them.

## 4. Check integration capabilities

The triage automation needs:

- Read access to the configured source Slack channel and its threads
- Thread-reply access in that channel
- Attachment metadata and file download access when reports include media
- Search, read, create, and update access through the configured issue-tracker adapter

The repro automation needs:

- Read access to the source thread
- Thread-reply access in the source channel
- Reaction or status access to claim a report
- Optional post and edit access in the configured operations channel
- Repository read and history access
- `gh` signed in, so `gh pr create --draft` can open a draft pull request
- The configured control-adapter skill

Slack access comes from a Slack MCP server that the provider agent has connected. Confirm that a fresh provider session lists its tools before you create an automation. The optional `BENNY_SLACK_BOT_TOKEN` may fill a narrow gap such as editing one operations status message or downloading an attachment. Store the value in a secret manager or environment, not in YAML.

Do not use undocumented integration endpoints.

## 5. Prepare the routing map

If the user wants reroutes or owner pings:

1. Copy `../triage-issue-reports/references/routing.example.md` outside `.claude/automations/benny/`.
2. Replace every placeholder with public or organization-local values.
3. Keep owner pings off by default.
4. Allow a ping only for a configured feature owner or a confirmed likely regression author.

If no routing map is configured, triage may classify a report but must not guess a destination or owner.

## 6. Verify the control adapter

Read `../reproduce-and-fix-issues/references/control-adapter.md` and the user's completed feature map.

Confirm that the named skill can:

- Bring up the target app
- Navigate every mapped feature through the real UI
- Exercise mapped states through declared adapter actions
- Inspect state without forcing the result
- Capture screenshots
- Start and stop a recording
- Clean up its processes and temporary data

A project `verify-*` skill from pstack's `create-verification-skill` fits this role. It drives the app through Orca's embedded browser, Orca terminals, or the `computer-use` skill.

If any capability is missing, leave the repro automation disabled. It must fail closed rather than claim a reproduction it did not perform.

## 7. Prepare the live automations

Load the `orca-cli` skill and its `references/automations.md` before running any automation command. `ORCA` is the executable that skill resolves.

Run `ORCA automations list --json` and check whether `benny-triage` or `benny-reproduce` already exists. That decides between first-time creation and an update.

Read `../../FOR_AGENTS.md` from the copied pack as the primary user-intent source for either path. Use it to understand the two polls, tools, instructions, outcomes, and shared rules.

### First-time creation

Create one automation at a time, always disabled.

For each automation:

1. Read the matching copied prompt template as secondary internal source material.
2. Turn `FOR_AGENTS.md`, the finished Benny configuration, and the template intent into one complete prompt.
3. Tell the prompt to read and follow its exact committed operational file under `.claude/automations/benny/`.
4. Use the stable repository-relative path, not a pack source or install path. Do not copy the operational file contents into the prompt.
5. Confirm that the copied pack and every referenced configuration file are committed on the default branch of the repository the automation uses.
6. Show the user the name, schedule, provider, placement, and full prompt. Create the automation only after the user approves them.
7. Run `ORCA automations show <id> --json` and show the user the saved result before starting the next automation.

Create triage in an existing workspace, so the runs share one session:

```text
ORCA automations create --name benny-triage --trigger "<triage_schedule>" --prompt "<triage prompt>" --provider <provider> --workspace <selector> --reuse-session --disabled --json
```

The triage prompt carries this complete intent, filled from configuration:

- Read and follow `.claude/automations/benny/skills/triage-issue-reports/SKILL.md` for every run.
- Poll the configured source Slack channel for top-level reports inside the lookback window that hold no Benny marker.
- Read each selected thread and reply only inside it.
- Use the configured issue-tracker integration.
- Classify, inspect evidence, trace cause, dedupe, and create only clear new bugs.
- End one thread-only verdict with the configured `[benny:bug]`, `[benny:performance]`, or `[benny:other]` marker and optional tracker URL.
- Never post a source-channel root message.

After the user approves the saved triage automation, create repro on the repository, so every run gets a fresh worktree:

```text
ORCA automations create --name benny-reproduce --trigger "<reproduce_schedule>" --prompt "<repro prompt>" --provider <provider> --repo <selector> --base-branch <default-branch> --disabled --json
```

The repro prompt carries this complete intent:

- Read and follow `.claude/automations/benny/skills/reproduce-and-fix-issues/SKILL.md` for every run.
- Poll the same source Slack channel for the oldest report whose thread holds a trusted bug or performance marker and that no earlier run has claimed. Claim it before any other work.
- Read the source thread and reply only inside it.
- Include `gh pr create --draft` and the configured tracker, control-adapter, and feature-map requirements. Paraphrase mapped user paths and states unless the map is a committed file in the same repository.
- Act only on a trusted triage marker.
- Reproduce the exact symptom twice through the mapped real UI and capture evidence.
- Verify an existing fix without authoring over it.
- Attempt an optional bounded fix only after confirmed repro, then open a draft pull request when proof and checks pass.
- Never post a source-channel root message.

### Existing automations

Finish configuration, routing, control-adapter, and feature-map validation. Then show the user what will change and update each automation in place with `ORCA automations edit <id>`.

For the existing triage automation, check:

- Name
- Direct instruction to read `.claude/automations/benny/skills/triage-issue-reports/SKILL.md`
- Schedule, provider, and workspace
- Poll rule, lookback window, and source channel
- Issue-tracker integration
- Paraphrased triage instructions, thread-only rule, and Benny verdict markers

For the existing repro automation, check:

- Name
- Direct instruction to read `.claude/automations/benny/skills/reproduce-and-fix-issues/SKILL.md`
- Schedule, provider, repository, and base branch
- Poll rule, claim rule, and source channel
- Draft pull request command
- Tracker, control-adapter, and feature-map requirements
- Paraphrased marker gate, evidence, verification, and bounded-fix instructions

Do not create replacements or duplicates.

### Creation boundary

Create and edit automations only through `ORCA automations` commands. Never write Orca's automation storage by hand. Every new automation starts with `--disabled`.

Do not enable either automation until the thread-safety test passes.

## 8. Test thread safety

Use a test channel or a harmless test report. Run each disabled automation by hand with `ORCA automations run <id> --json`, and read the result with `ORCA automations runs --id <id> --json`.

Before testing, confirm that `.claude/automations/benny/` and every referenced secret-free configuration file are committed on the branch the automation checks out. Confirm that both live prompts point at their exact committed operational files. If any check fails, stop. Tell the user that the automation cannot be enabled yet.

Verify:

1. Triage stores the root `thread_ts` and posts exactly one verdict as a reply.
2. The verdict contains one configured marker.
3. A second triage run over the same window posts nothing.
4. Repro accepts the marker only from the configured triage identity.
5. Repro keeps the same immutable source coordinates.
6. A second repro run skips the report the first run claimed.
7. No source-channel root message appears.
8. A delegated worker cannot use any Slack write action.
9. Missing coordinates, a deleted parent, or a failed preflight produces no post and no tracker issue.

Enable normal traffic with `ORCA automations edit <id> --enabled --json` only after all nine checks pass.
