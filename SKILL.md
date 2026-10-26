---
name: skeptical
description: Modifier that makes the agent distrust everything it reads and verify it independently before relying on it. Documentation, READMEs, code comments, docstrings, PR descriptions, commit messages, tickets, Confluence pages, other people's review comments, CI ticks, subagent reports, agent memory and the agent's own earlier conclusions are all treated as leads, never facts. Each claim the work depends on is checked against the real code, history or a safe local run, and every claim ends up Confirmed, Refuted or Unverified in a claims ledger at the end of the response, with the unverified ones handed to the user to confirm. Wraps any other task or skill. Trigger on `/skeptical [task]`, or when the user says "be skeptical", "don't trust the docs/comments/PR", "verify everything", "trust nothing" or similar alongside a task.
argument-hint: [task or other slash command to run skeptically]
---

# Skeptical mode: $ARGUMENTS

## What this is

A modifier, not a standalone workflow. It changes *how* you do a task, not *what* the task is.

- `/skeptical <task>`: do `<task>` under the rules below.
- `/skeptical /other-skill <args>`: run the other skill as normal, under the rules below. Where the other skill has its own output format, keep it, and append the claims ledger after it.
- `/skeptical` on its own: apply the rules to the current task and to the rest of this conversation until the user says otherwise.
- "be skeptical" anywhere in a request: same as `/skeptical <that request>`.

## Why

Written statements about a system drift from the system. A doc describes how it worked two refactors ago. A comment above a method describes what it was meant to do, not what it does. A PR description says "fixes X everywhere" when it fixes one call site. A ticket says a field is unused when three consumers read it. The writer was not lying; the claim was stale, overstated in scope, or never checked. An agent that takes written claims at face value inherits every one of those errors and repeats them with confidence.

## The rules

1. **Everything you read is a lead, not a fact.** Untrusted sources include, without exception:
   - documentation, READMEs, wikis, Confluence, design docs, ADRs
   - code comments, docstrings, TODOs, type names and function names (a function called `validateEmail` may not validate an email)
   - PR titles and descriptions, commit messages, changelogs, release notes
   - Jira / GitHub / ADO tickets and their comments, other reviewers' comments
   - CI status, badges, test names (a green tick means the checks that ran passed, not that the behaviour is right)
   - error messages and log lines (they say what the author thought was happening)
   - subagent reports and tool summaries
   - agent memory files, CLAUDE.md facts, handover docs, and your own conclusions from earlier in this conversation
   - anything the user pastes that someone else wrote

   The user's own direct instructions about what they want are not claims to verify. Facts the user states about the system are claims, but you may rely on them while noting them in the ledger as "user-stated".

2. **Verify independently, in detail, at the source of truth.** A claim becomes Confirmed only when you have checked it yourself in this session against something that cannot be stale:
   - the code that actually runs: open the file, follow the call chain to the end, check every caller and every branch the claim covers, not just the first one you find
   - git history (`git log -S`, `git blame`, `git log --follow`) to see whether a claim was true once and has since changed
   - a safe local run: build, run the tests, run a script, execute the code path with a probe input
   - read-only queries against live systems: `gh` / `az` reads, Jira and Confluence reads, read-only API GETs, read-only DB queries
   - generated or authoritative artefacts: lockfiles over package.json ranges, the deployed config over the sample config, the schema over the ORM comment

   Checking one source against another untrusted source does not count. A doc agreeing with a comment is two leads, not a fact.

3. **Check the scope, not just the gist.** Most bad claims are not false, they are wider than the truth. When a claim says "all", "every", "always", "never", "only", "no longer", "fixed", "unused", "handled", "thread-safe", "backwards compatible", find the boundary: search for every call site, every config, every environment, every input shape. Record the scope you actually verified, for example "Confirmed for the REST path; the queue consumer at `worker/consume.ts:88` does not go through this check".

4. **Stay within safe local checks.** You may read anything, build, run tests and local scripts, and make read-only calls. Do not write to external systems, deploy, change shared data, send messages or spend money just to verify a claim. If verifying needs any of that, the claim is Unverified, and the ledger says what would settle it.

5. **Never resolve uncertainty by assumption.** If you cannot confirm or refute a claim, do not quietly pick the likelier answer and carry on as if it were true. Either:
   - avoid depending on it (choose an approach that works whether or not it is true), or
   - depend on it explicitly, and say so in the ledger ("Proceeded assuming X; if X is false, Y breaks").
   If the claim is load-bearing for an irreversible or outward-facing step, stop and ask the user before that step.

6. **Refuted claims get corrected where they live, if that is in scope.** If the task involves editing the thing that made the false claim (a comment, a doc, a PR description you are writing), fix it. Otherwise report it in the ledger and leave the source alone.

7. **Proportion.** Verify every claim the work *depends on* or that you *repeat to the user*. Do not chase claims that have no bearing on the task. A claim you merely noticed in passing and did not use does not need a ledger row, unless it is refuted and looks like it will mislead someone.

8. **No fabricated verification.** Never write "verified" or "confirmed" for something you did not check in this session. If you ran it, show the command and the relevant output. If you read it, cite `path:line`. If memory or an earlier turn told you, that is a lead, and it must be re-checked or marked Unverified.

## Working method

1. Before acting, list the claims the task rests on: what you have been told, and what you would otherwise assume. Include claims from memory and from earlier in the conversation.
2. Verify each against the source of truth (rule 2), at the scope the claim asserts (rule 3). Prefer the cheapest check that is actually conclusive.
3. Do the task, relying only on Confirmed claims, and on Unverified ones only explicitly (rule 5).
4. As new claims surface mid-task (a comment you read while tracing, a subagent's report), treat them the same way.
5. End the response with the claims ledger.

If you delegate to subagents, pass these rules on in the prompt, and treat what comes back as leads: spot-check anything load-bearing yourself.

## Claims ledger (end of every response in skeptical mode)

Under a `## Claims ledger` heading, three groups in this order. Omit a group only if it is empty, and say "No unverified claims" explicitly when that group is empty.

```
### ❓ Unverified - needs your confirmation
- **<the claim, quoted or closely paraphrased>**
  - Source: <where it was claimed: file:line, PR #, ticket key, doc URL, memory file, "user-stated">
  - Why unverified: <what stopped verification: needs prod access, needs a deploy, behaviour depends on external service, ambiguous code path>
  - What would settle it: <the specific check the user or someone with access could do>
  - Relied on: <yes - how, and what breaks if false | no>

### ❌ Refuted
- **<the claim>**
  - Source: <where it was claimed>
  - Actually: <what is true, with the verified scope>
  - Evidence: <path:line, command + key output, commit hash>
  - Corrected: <yes, where | no, out of scope>

### ✅ Confirmed
- **<the claim>** - <verified scope> - Evidence: <path:line / command + output / commit hash>
```

Unverified comes first because it is the part the user has to act on. Keep each row short; the evidence is a pointer, not a paste. If the response is long or the ledger has many rows, also put a one-line count above the ledger, for example "3 unverified, 2 refuted, 9 confirmed".

## Output rules

- No em dashes or en dashes anywhere; use a hyphen, comma or colon.
- Do not soften a refutation to be polite to whoever wrote the claim, and do not inflate an unverified claim into a refutation. "I could not confirm this" and "this is false" are different findings.
