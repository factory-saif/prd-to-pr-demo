# Orchestrated mode (PDLC)

Read this file only when the caller says it is the PDLC orchestrator. It
overrides `SKILL.md` and the stage files where they differ. Everything not
named here applies unchanged, including every idempotency and trust check.

In this mode a workstream orchestrator starts every agent, and every agent
reports back to it. Agents never hand work to each other.

## Fixed context overrides

| Item | Value |
| --- | --- |
| Linear project | `CBRE Demo`, ID `275f8294-af82-438e-91f3-787caa3efd90`, team `SED`. Use this ID in every project filter and every project check, wherever a stage file says `Demo` |
| Caller | The PDLC orchestrator session |
| Target | One stage and one exact target, named by the caller |

## Target, not selection

The caller names one stage and one target:

| Stage | Target |
| --- | --- |
| `prd` then `plan` | One PRD issue key |
| `dev` | One story key |
| `fix` | One story key and the PR head SHA to fix |
| `review`, `qa` | One PR number and its head SHA |

Skip the stage's "Select work" listing and act only on that target. Still run
every check the listing would have run on it: project, trusted signals,
`route:human`, existing PR, already-reviewed SHA. If a check says skip, do not
act; report `blocked` with the reason. If the PR head SHA is not the one
named, stop and report it.

## Agents

| Agent | What it does |
| --- | --- |
| Planner | Runs `stages/prd.md` then `stages/plan.md` back to back on the named issue, in one session. No Linear delegation is needed. PRD intake is a top-level `CBRE Demo` issue in `Todo`. In `plan`, run only part A for that issue; skip part B (the orchestrator decides releases) |
| Coder | Runs `stages/dev.md` "Build one story" for the named story and stays alive for the story's life. Takes no other story |
| Reviewer | Runs `stages/review.md` "Review one pull request" on the named PR and SHA. Sets `bug` and removes `Release` as today |
| QA | Runs `stages/qa.md` "Test one pull request" and "Decide the story's next step" on the named PR and SHA. Sets `bug`, `Release`, or `route:human` as today |

## Fix rounds

A fix round arrives at the same Coder session as a message from the
orchestrator naming the story and the head SHA. The Coder:

1. Follows `stages/dev.md` "Fix returned stories" steps 2 to 8 for that story
   only. The "at most two per run" limit does not apply.
2. May reuse its own private clone. Run `git fetch origin` first and confirm
   `origin/story/<KEY>` is at the named SHA before changing anything.
3. Still posts and updates `## Fix Loop #<n>`, and still stops at 3 loops with
   `route:human`.
4. Reports with `stage: fix` and the new head SHA.

Review and QA do not rerun "later in this chain run". The orchestrator starts
them on the new SHA.

## Report

End every run with this block as the last thing in the turn, and send the same
block to the orchestrator with `SendMessage`:

```
PDLC-REPORT
stage: prd|plan|dev|fix|review|qa
target: <SED key or PR number>
head_sha: <sha or ->
result: done|clean|changes_requested|pass|fail|blocked|failed
links: <Linear/PR/comment URLs>
notes: <one line>
```

The Planner sends one report for `plan` (or for `prd` when it stops there,
for example after `## PRD Questions`).

## Rules

- Never merge. Never touch a pull request other than the target.
- Never mention `@droid` in any GitHub text.
- Work in a fresh clone under a new `mktemp -d` directory. Never use a
  checkout another session might use.
