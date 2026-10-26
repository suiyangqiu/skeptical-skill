# skeptical

A Claude Code skill that makes the agent distrust what it reads and verify it before relying on it.

Agents tend to treat documentation, code comments, PR descriptions and tickets as the truth. Those claims are often stale (the code changed, the comment did not) or overstated in scope (the PR says "fixed everywhere", it fixed one call site). In skeptical mode every written claim is a lead, not a fact, until the agent has checked it against the code that actually runs.

## Usage

`/skeptical` is a modifier. It changes how a task is done, not what the task is.

| Command | What it does |
|---|---|
| `/skeptical <task>` | Do the task under skeptical rules. |
| `/skeptical /other-skill <args>` | Run another skill as normal, under skeptical rules, with the claims ledger appended to its output. |
| `/skeptical` | Apply the rules to the current task and the rest of the conversation. |
| "...and be skeptical" | Saying it in a request works the same as the slash command. |

## What counts as untrusted

Docs, READMEs, wikis, code comments, docstrings, function names, PR titles and descriptions, commit messages, changelogs, tickets, other reviewers' comments, CI status, log and error messages, subagent reports, agent memory, and the agent's own earlier conclusions in the conversation.

## How claims get verified

- Against the source of truth: the code that runs (every caller, every branch the claim covers), git history, a local build, test or script run, and read-only queries against live systems.
- At the scope the claim asserts. "All", "never", "fixed", "unused" mean finding every call site, not the first one.
- With safe checks only. No external writes, deploys or messages just to verify something; if that is what it would take, the claim stays Unverified.

## Claims ledger

Every response in skeptical mode ends with a ledger:

1. **❓ Unverified** - first, because these need the user. Each has where it was claimed, why it could not be checked, what would settle it, and whether the work relied on it.
2. **❌ Refuted** - what is actually true, with evidence, and whether the source was corrected.
3. **✅ Confirmed** - the verified scope and the evidence (`path:line`, command output, commit hash).

## Install

```bash
git clone https://github.com/suiyangqiu/skeptical-skill.git ~/.claude/skills/skeptical
```

## License

MIT
