---
name: review-pr
description: >
  Use when reviewing a pull request as the reviewer half of a paired, multi-round
  review loop — either a PR is freshly ready for review, or the PR author has
  posted a rebuttal that needs re-reviewing. Triggers: "review this PR", "re-review
  the changes", running a converging review cycle. Pairs with the request-pr-review
  skill (the author half). Requires an authenticated `gh` CLI.
---

# review-pr — reviewer side of the PR review loop

## Overview

This is one half of a paired, multi-round PR review workflow. **This skill is the
reviewer.** The author runs `request-pr-review` in a separate session. The two
sessions converge through the PR and never communicate directly.

**Core principle:** review early, re-review *narrowly*. Round 1 is a full review;
later rounds verify prior concerns and suppress new nitpicks — so a one-line fix
never reaches round 7 on style.

## How this pairs

| | |
|---|---|
| You run this | in a session you trigger when a PR is ready for review |
| Author runs | `request-pr-review` in the branch-owning session |
| Channel | the PR only — see Wire protocol |
| Counterpart skill | `request-pr-review` |

## Wire protocol

- You post via `gh pr review <PR> --comment --body "…"` — a `COMMENT` review. A
  single GitHub account **cannot** `--approve` its own PR, so approval is carried
  in a marker, not a native review state.
- Every review you post ends with two markers (the **turn marker** + the **verdict
  trailer**):

  ```
  <!-- review-loop:reviewer round=N -->
  <!-- review-verdict: {"verdict":"changes-requested","round":N,"blocking":2,"nits":3} -->
  ```

  `verdict` is `approved`, `changes-requested`, or `escalated` (you stopped
  without convergence — see Exit); `blocking`/`nits` are counts.
- The author posts issue comments tagged `<!-- review-loop:implementer round=N -->`.
- **Convergence** = you post a review with `"verdict":"approved"` (`blocking` is 0).

### Round numbering

Each side numbers its own posts from 1. **Your round number = 1 + the count of
existing `review-loop:reviewer` turn markers on the PR.** Reviewer review 1 is the
initial review. Thereafter reviewer review *N+1* answers implementer rebuttal *N*.

### Blocking vs. nit

- **Blocking** — correctness, logic error, security, missing tests for new
  behavior, broken build, violated project convention. Must have a `file:line`
  citation. Any blocking finding → `changes-requested`.
- **Nit** — style, naming, optional polish. Never blocks convergence.

## Setup

The reviewer session needs the PR's code locally (re-reviews read files in
context, not just diff hunks):

```bash
gh pr checkout <PR>          # creates/updates the local PR branch
git rev-parse HEAD           # record this SHA as the round-1 baseline
```

## The loop

### On entry (every invocation)

1. Resolve the PR number (skill arg, else `gh pr list` and ask).
2. **Idempotency guard:** count existing turn markers — reviewer `R`,
   implementer `I`. If the latest reviewer review is `approved` or `escalated`,
   stop. If `R > I`, you have already reviewed the latest changes — stop. (This
   guards a re-invocation in a fresh session; a running loop re-enters via the
   Monitor, not here.)

### Round 1 — full review

3. `gh pr checkout <PR>`; record the baseline SHA.
4. Dispatch a **fresh `general-purpose` reviewer subagent** over the diff
   (`gh pr diff <PR>`). Instruct it to:
   - read every `AGENTS.md`/`CLAUDE.md` from the repo root down to each changed
     file's directory, and follow the project's conventions;
   - if `.claude/agents/` contains project reviewer agents (name/description
     mentions code review), dispatch those **as well** and fold in their findings;
   - cover correctness/logic, security, tests, performance, docs, convention
     adherence;
   - cite `file:line` for **every** blocking finding (no citation → it is a nit,
     not blocking);
   - if the diff is very large, prioritize core logic over generated/vendored files.
5. Post a `COMMENT` review using the body template below. `changes-requested` if
   `blocking > 0`, else `approved`. Re-record `git rev-parse HEAD` as the new
   baseline.
6. If converged or `--once`: stop. Otherwise arm the Monitor — **once**.

### Rounds 2+ — scoped re-review

The Monitor runs for the whole loop and emits one line per new author rebuttal —
do not re-arm it. On each notification, finish any in-progress round first, then:

1. `gh pr checkout <PR>` again to pull the author's new commits.
2. Dispatch a fresh `general-purpose` subagent given {your prior reviews, the
   author's rebuttal, the incremental diff `git diff <baseline>..HEAD`, and the
   files it touches}. Instruct it to:
   - review in **full file context** — read surrounding code, not just hunks;
   - **verify each prior blocking concern** is actually resolved;
   - **re-scan the new commits for regressions** — a fix can introduce a new bug;
   - report **blocking findings only** — new nitpicks are suppressed after round 1;
   - **not re-litigate** points the author rebutted soundly that you accepted.
3. Post the new review; re-record the baseline SHA.

### Exit

Stop when **any** of:
- verdict is `approved`;
- you have posted **5** reviewer reviews without convergence;
- a blocking finding stays contested for 2 rounds (author rebuts, you re-assert) —
  a stalemate, not progress.

On a non-approved exit (cap or stalemate), post a final review whose verdict
trailer is `"verdict":"escalated"`, listing what remains — this signals the
author's loop to stop too, so it does not wait forever on a reviewer that has
already left. Then `TaskStop` the Monitor and print a round-by-round summary.

## Review body template

```
## Review — round N

<one-paragraph summary>

### Blocking
1. `path/file.ext:LINE` — <issue and why it blocks>

### Nits
- `path/file.ext:LINE` — <optional improvement>

### Resolved since last round
- <prior concern> — confirmed fixed

<!-- review-loop:reviewer round=N -->
<!-- review-verdict: {"verdict":"changes-requested","round":N,"blocking":1,"nits":1} -->
```

Omit empty sections. Round 1 has no "Resolved" section.

## Monitor

Arm this **once** as a persistent Monitor (`persistent: true`). It tracks state
in its own loop and survives every round — do not re-arm it. Capture the returned
task id for `TaskStop` at exit.

```bash
PR=<number>; seen=0
while true; do
  r=$(gh pr view "$PR" --json comments --jq '.comments[].body' 2>/dev/null \
      | grep -oP 'review-loop:implementer round=\K[0-9]+' | sort -n | tail -1 || true)
  if [ -n "$r" ] && [ "$r" -gt "$seen" ]; then
    echo "implementer rebuttal posted: round $r"; seen="$r"
  fi
  sleep 30
done
```

Each emitted line is your cue to run the next re-review round.

## --once mode

`review-pr <PR> --once` — do Round 1 only, post the review, do not arm the Monitor.

## Anti-patterns

- **Approving to end the loop.** Both sessions are the same model; the reviewer is
  tempted to converge prematurely. `approved` requires `blocking: 0` from an actual
  count, not a vibe.
- **New nitpicks in round ≥2.** Severity-gate: a nit must never block convergence.
- **Reopening settled points** the author already rebutted and you accepted.
- **Reviewing only the diff hunks** on re-review — read the surrounding code.
