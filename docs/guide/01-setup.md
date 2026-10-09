# Set up pstack

In this page you install the skills, pick which models pstack uses, and run your first task. Setup is a clone, three link commands, and a short conversation.

## Install the skills

pstack runs in Claude Code inside an Orca terminal. It needs the Orca `orchestration`, `orca-cli`, and `computer-use` skills, which `orca skills install` provides. Install `codex` as well if you want the review panels to span two model families.

Clone the repository and link its skills and subagents:

```bash
git clone https://github.com/micaelsgarcez/pstack-orca.git ~/pstack-src
mkdir -p ~/.claude/skills ~/.claude/agents ~/.agents/skills
ln -s ~/pstack-src/skills/* ~/.claude/skills/
ln -s ~/pstack-src/agents/*.md ~/.claude/agents/
ln -s ~/pstack-src/skills/* ~/.agents/skills/
```

The last line lets `codex` workers read the same skills. Start a new Claude Code session, then type `/poteto-mode` to confirm the skills are there.

## Pick your models

Run:

```text
/setup-pstack
```

[`/setup-pstack`](../../skills/setup-pstack/SKILL.md) detects the subagent models and Orca agents you have, shows you each role (code delegates, judgment, the review panels), and asks what you want. Answer the questions. It writes `~/.claude/rules/pstack-models.md`, a small rule every pstack skill reads. This step is optional, because every role has a default.

You only override what you care about. A role with no line in the rule keeps the skill's default. To restore a default, delete that role's line. A rerun of `/setup-pstack` keeps the choices already in the file. When a default changes, a rule written before the change still pins the old default, so delete those role lines, or delete the file, then run `/setup-pstack` again.

Roles come in two kinds. A subagent role (code delegates, `how`, `why`, `reflect`) takes a Claude Code model alias such as `sonnet`. Set it to `inherit-parent` or `auto` and pstack omits the subagent `model` field, so the subagent inherits your session's model. A worker role (`arena`, `architect`, `interrogate`, `swarm`) takes Orca agents, written as `<agent>`, `<agent>:<model>`, or `<agent>:<model>:<effort>`. For a panel role the value is a list, and one Orca worker runs per entry, so the list length sets the panel size. The default panel is `claude, codex`. Setup also configures `swarm workers`, the default agent for every `/swarm` worker unless a race names one for each arm.

## Accept the verification offer, or don't

At the end of setup, `/setup-pstack` looks for a way to prove app behavior in your project, either a `verify-*` skill or an existing harness. If it finds neither, it offers once to generate one with [`/create-verification-skill`](../../skills/create-verification-skill/SKILL.md).

Say yes and it writes `.claude/skills/verify-<app>/`, a project-local skill that teaches agents to drive your app the way a user does. It proves the skill works once before handing it over. Say no and setup moves on. You can run `/create-verification-skill` yourself any time. [Verify and ship](./06-verify-and-ship.md#create-a-project-verification-skill) covers it in depth.

If you're new to pstack, say yes. An agent that can check its own work keeps going until the check passes. An agent that can't hands every result back to you to check by hand. Of everything in this guide, the verification skill pays off the most.

After setup, start a new chat. The model rule applies to new sessions.

## Keep the cost in check

pstack spends extra tokens on subagents and review panels. That's the price of the rigor. To spend fewer:

- Rerun `/setup-pstack` and pick a smaller reasoning budget or cheaper models. A strong model in the main chat with cheaper, faster models in the code roles is a good split.
- Set a role to `auto` or `inherit-parent` so it runs on the chat's own model.
- Shorten a panel list. Each entry runs one subagent.
- Save `/poteto-mode` for work that needs rigor. A small, obvious edit doesn't.

## Run your first task

Pick something real but small, and describe it the way you'd describe it to a colleague:

```text
/poteto-mode add a --json flag to this command. text output stays byte-identical. verify both.
```

Watch the todo list. Its first items are the matched playbook's steps copied in, the Feature playbook for this prompt. If `/poteto-mode` skips a step, the step stays in the list with `skip: <reason>`, so you can see what it chose not to do.

From here you can type normal follow-ups. The skill stays in the conversation once loaded, but it fades as the chat moves on, so start each new task with `/poteto-mode` again.

Next: [Route work through `/poteto-mode`](./02-poteto-mode.md).
