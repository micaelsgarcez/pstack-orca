# benny

benny gives you two orca automations for slack issue reports. one triages each report. the other reproduces confirmed bugs and may prepare a small draft fix.

orca automations run on a schedule, not on a slack event. each run polls the source channel and picks the reports that still need work. the benny markers in each thread keep the runs idempotent.

the files in this directory are dormant setup and automation sources. they do not appear as slash skills.

## what you need

- orca running on the machine that hosts the automations, with `claude` or `codex` as the provider.
- pstack installed for that user (`~/.claude/skills/`), for the shared skills benny reads.
- a slack mcp server the provider agent can use to read threads and reply in them.
- `gh` signed in, for draft pull requests.

## set it up

1. point your agent at [`FOR_AGENTS.md`](./FOR_AGENTS.md) and name the target repository.
2. let setup merge this whole directory into the target at `.claude/automations/benny/`. it must preserve destination-only files and review conflicts instead of overwriting local edits.
3. keep user-owned configuration outside the copied pack, for example in `.claude/benny/`. adapt [`configuration.example.yaml`](./templates/configuration.example.yaml) and [`feature-map.example.md`](./skills/reproduce-and-fix-issues/references/feature-map.example.md).
4. commit `.claude/automations/benny/` and any secret-free configuration before enabling either automation.
5. let setup create both automations disabled with `orca automations create --disabled`. review each one with `orca automations show <id>`, send a harmless test report, run each automation once with `orca automations run <id>`, and verify every source-channel post stays in the original thread. enable them after that.
