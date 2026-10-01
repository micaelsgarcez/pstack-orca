# Triage automation prompt

> Source material for the copied setup workflow. Paraphrase this intent into the `--prompt` of the `benny-triage` Orca automation after you confirm that the copied pack is committed in the repository where the automation will run.

Read and follow `.claude/automations/benny/skills/triage-issue-reports/SKILL.md` for this run.

Configuration source. Include this repository-relative path only when it is committed in the same target repository. Otherwise paraphrase the configured values. Never use a pstack install path:

```text
{{BENNY_CONFIG_PATH}}
```

Poll. An Orca automation runs on a schedule, not on a Slack event, so every run selects its own reports:

1. Read the top-level messages posted in the configured source Slack channel within `budgets.poll_lookback_minutes`.
2. Skip a report whose thread already holds a configured Benny marker from the configured triage identity.
3. Take the remaining reports oldest first, one at a time. With none left, end the run without posting.

Trigger, built once for each selected report:

```json
{
	"source_channel_id": "{{SLACK_CHANNEL_ID}}",
	"ts": "{{SLACK_MESSAGE_TS}}",
	"thread_ts": "{{SLACK_THREAD_TS_OR_EMPTY}}"
}
```

Treat the source channel and root thread timestamp as immutable. If either is missing or does not match configuration, stop without posting or writing to the issue tracker.

The committed operational file owns classification, attachment review, cause tracing, routing, dedupe, tracker writes, and the final verdict. Post no progress messages. Never post a root message in the source channel.

Two runs can overlap. Read the thread again immediately before the verdict post, and post nothing if a Benny marker has appeared.

The coordinator is the only Slack poster. Any delegated worker must be read-only, return findings only, and receive an explicit ban on every Slack write action.

End the single verdict with exactly one configured marker:

```text
[benny:bug]
[benny:performance]
[benny:other]
```

A bug or performance marker may add `tracker=<URL>`.
