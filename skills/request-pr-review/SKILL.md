---
name: request-pr-review
description: >
  Use when you are the author of a pull request and want a managed, multi-round
  review loop — you have a branch ready for review and want to open the PR and
  handle reviewer feedback round by round until it converges. Triggers: "open a PR
  and get it reviewed", "respond to the review", running a converging review cycle
  as the author. Pairs with the review-pr skill (the reviewer half). Requires an
  authenticated `gh` CLI.
---

# request-pr-review — author side of the PR review loop

## Overview

This is one half of a paired, multi-round PR review workflow. **This skill is the
author/implementer.** The reviewer runs `review-pr` in a separate session. The two
sessions converge through the PR and never communicate directly.

**Core principle:** respond to review with technical rigor — verify before
implementing, push back when the reviewer is wrong, never perform agreement.

## How this pairs

| | |
|---|---|
| You run this | in the session that owns the branch |
| Reviewer runs | `review-pr` in a separate session you trigger |
| Channel | the PR only — see Wire protocol |
| Counterpart skill | `review-pr` |

## Wire protocol

- You post rebuttals via `gh pr comment <PR> --body "…"` — an issue comment whose
  body ends with a **turn marker**:
  `<!-- review-loop:implementer round=N done=true|false -->`.
  - `done=false` — you pushed changes (or are still answering blocking findings);
    the reviewer should look again. Use it on every `changes-requested` rebuttal
    and on any post-approval rebuttal that carries new commits.
  - `done=true` — you are finished and intend no further changes. Post it **only**
    when the latest reviewer verdict is `approved`. It is your convergence signal:
    it releases the reviewer to disarm.
- The reviewer posts `COMMENT` reviews tagged `<!-- review-loop:reviewer round=N -->`,
  each ending with a verdict trailer:
  `<!-- review-verdict: {"verdict":"approved","round":N,"blocking":0,"nits":0} -->`.
  A single GitHub account cannot `--approve` its own PR, so the verdict is a marker.
- **Verdict values:** `approved` (no blocking findings — provisional, not the end
  of the loop), `escalated` (the reviewer stopped without convergence — see Exit),
  or `changes-requested`. A reviewer review with no parseable verdict trailer
  counts as `changes-requested`.

### Round numbering

Each side numbers its own posts from 1. **Your round number = 1 + the count of
existing `review-loop:implementer` turn markers on the PR.** Your rebuttal *N*
answers reviewer review *N*.

## The loop

### Open the PR

1. Check for an existing PR: `gh pr view --json number,url,state`.
2. If none exists:
   - If the branch has unpushed commits, `git push -u origin HEAD`.
   - `gh pr create --fill` — **always `--fill`**; bare `gh pr create` opens an
     interactive editor, which is unsupported here. (`--fill` derives title/body
     from the commits.) If `gh` reports no commits / no diff vs. the base branch,
     stop and tell the user — there is nothing to review.
3. Tell the user the PR URL and that they can now trigger `review-pr <number>` in
   another session. The user may be away — make the message self-contained.
4. If `--once`: if a reviewer review already exists, respond to it once (steps
   5–8) and stop; if none exists, stop here. Otherwise arm the Monitor — **once**.

### Respond each round

The Monitor runs for the whole loop and emits one line per new reviewer review —
do not re-arm it. On each notification, finish any in-progress round first, then:

5. Parse the verdict trailer of the new review:
   - `"verdict":"escalated"` → the reviewer has stopped without convergence; do
     **not** respond or keep waiting — go to **Exit without convergence**;
   - `"verdict":"approved"` → all blocking findings are clear, but the review may
     carry nits and the loop is **not** closed yet — go to **Respond to approval**;
   - otherwise (`changes-requested`, or no parseable trailer) → triage and respond
     (steps 6–8); your rebuttal's turn marker carries `done=false`.
6. Triage every point. **REQUIRED REFERENCE: read the bundled
   `reference/feedback-triage.md` before triaging — it carries the
   feedback-reception discipline this step depends on.** For each point:
   - bucket it: **bug / doc gap / suggestion / praise**;
   - **verify the claim against the code before acting** — a reviewer's proposed
     fix can itself be wrong;
   - **push back** on incorrect or out-of-scope items with technical reasoning —
     no performative agreement, no thanking.
7. Implement bugs and doc gaps; decide each suggestion on merit. Commit as
   `address review feedback (round N)`; `git push`.
8. Post a rebuttal with `gh pr comment` ending in your turn marker
   `<!-- review-loop:implementer round=N done=false -->`:
   - **Fixed:** what changed (and, if you diverged from the reviewer's suggestion, why);
   - **Skipped:** explicit rationale for each — complexity cost, scope, or a better
     alternative.
   Post the rebuttal **even if every point was skipped** — it shows each point was
   weighed and cues the reviewer's next round. Then keep monitoring.

### Respond to approval

`approved` clears every blocking finding, but its nits are unaddressed and the
loop stays open until you post `done=true`. This is a branch off step 5 — do
**not** jump straight to Converged.

- **Triage the nits** exactly as in step 6 — **read the bundled
  `reference/feedback-triage.md` first** — verifying each against the code and
  pushing back on the unsound ones.
- Then **fork on the outcome**:
  - **You made no code changes** (every nit declined or deferred) → go to
    **Converged**; its comment carries `done=true`.
  - **You made code changes** → commit as `address review nits (round N)`, then
    `git push`, then post a rebuttal whose turn marker is
    `<!-- review-loop:implementer round=N done=false -->` (Fixed/Skipped
    sections as in step 8). Keep monitoring — the reviewer re-reviews the new
    commits, then re-approves (or flags a regression). On the next `approved`,
    return to **Respond to approval**; on `changes-requested`, triage and
    respond via steps 6–8.

### Converged

9. Post the converge comment — it **must** end with a `done=true` turn marker,
   which is the signal that releases the reviewer:

   ```
   Converged at round N — all blocking items resolved; nits triaged.

   <!-- review-loop:implementer round=N done=true -->
   ```

   (`N` = 1 + the count of existing `review-loop:implementer` turn markers.) Then
   `TaskStop` the Monitor (use the task id captured when you armed it).
10. Notify the user with a round-by-round summary, then surface — **do not run** —
    the primed merge command:

    ```bash
    gh pr merge <PR> --squash --delete-branch
    ```

    Run it **only on the user's explicit go-ahead**. Never auto-merge.

### Exit without convergence

Stop, `TaskStop` the Monitor, and escalate to the user — stating the open
disagreement and what each side last said — when **either**:
- a reviewer review carries `"verdict":"escalated"` (the reviewer hit its own cap
  or a stalemate); or
- you have responded to **5** `changes-requested` reviews without reaching an
  `approved` verdict (stop after posting that 5th rebuttal). Post-approval
  `done=false` rounds do not count toward this cap — once the reviewer has
  approved, converge via `done=true` rather than escalating.

## Monitor

Arm this **once** as a persistent Monitor (`persistent: true`). It tracks state in
its own loop and survives every round — do not re-arm it. Capture the returned
task id for `TaskStop` at convergence.

```bash
PR=<number>; seen=0
while true; do
  body=$(gh pr view "$PR" --json reviews --jq '.reviews[].body' 2>/dev/null || true)
  r=$(echo "$body" | grep -oP 'review-loop:reviewer round=\K[0-9]+' | sort -n | tail -1 || true)
  if [ -n "$r" ] && [ "$r" -gt "$seen" ]; then
    v=$(echo "$body" | grep -oP 'review-verdict: \K\{[^}]*\}' | tail -1 || true)
    echo "reviewer round $r verdict: ${v:-MISSING}"; seen="$r"
  fi
  sleep 30
done
```

Each emitted line is your cue to triage and respond — unless the verdict is
`approved`, in which case converge.

## --once mode

`request-pr-review --once` — open the PR (if needed) and respond to the current
review once, without arming the Monitor. If the current review is `approved`,
run **Respond to approval** once (triage nits, then converge with `done=true` or
post a `done=false` rebuttal). If no review exists yet, it just opens the PR.

## Anti-patterns

- **"You're absolutely right!" / thanking the reviewer.** State the fix instead —
  the code shows you heard the feedback.
- **Implementing a suggestion without verifying it.** The reviewer's proposed fix
  can be wrong; check it against the code first.
- **Auto-merging on approval.** The squash-merge is primed but waits for the user.
- **Treating `approved` as the finish line.** `approved` only clears blocking
  findings; the loop closes when you post `done=true`. Triage the nits first.
- **Silent skips.** Every skipped point needs a written rationale in the rebuttal.
