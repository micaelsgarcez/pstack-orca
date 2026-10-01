# Reproduce automation prompt

> Source material for the copied setup workflow. Paraphrase this intent into the `--prompt` of the `benny-reproduce` Orca automation after you confirm that the copied pack is committed in the repository where the automation will run.

Read and follow `.claude/automations/benny/skills/reproduce-and-fix-issues/SKILL.md` for this run.

Configuration source. Include this repository-relative path only when it is committed in the same target repository. Otherwise paraphrase the configured values. Never use a pstack install path:

```text
{{BENNY_CONFIG_PATH}}
```

Poll. An Orca automation runs on a schedule, not on a Slack event, so every run selects its own report:

1. Read the top-level messages posted in the configured source Slack channel within `budgets.poll_lookback_minutes`.
2. Keep a report only when its thread holds a `[benny:bug]` or `[benny:performance]` marker from the configured triage identity.
3. Skip a report that already carries a configured Benny status emoji or a Benny repro result.
4. Take the oldest remaining report. Claim it with the configured `seen` status before any other work. With none left, end the run without posting.

Trigger, built for the claimed report:

```json
{
	"source_channel_id": "{{SLACK_CHANNEL_ID}}",
	"ts": "{{SLACK_MESSAGE_TS}}",
	"thread_ts": "{{SLACK_THREAD_TS_OR_EMPTY}}"
}
```

The automation runs in a fresh Orca worktree of the configured repository at its default branch. It uses the configured issue tracker, control adapter, and feature map, and opens pull requests with `gh pr create --draft`.

Treat the source channel and root thread timestamp as immutable. If either is missing or does not match configuration, stop without posting.

Proceed only for `[benny:bug]` or `[benny:performance]` from the configured triage identity in this exact thread.

Require the configured control-adapter skill before attempting a repro. Reproduce the exact discriminating symptom twice through the real UI. Verify existing pull requests or commits without authoring over them. Attempt a bounded fix only after a confirmed repro and the operational file's fix gate.

The coordinator is the only Slack poster. Every child prompt must forbid `SendSlackMessage`, `PostToSlack`, `chat.postMessage`, and all other Slack writes. Children return findings only.

Never post a root message in the source channel.
