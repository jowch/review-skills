# Review Loop Done-Signal Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a post-approval review round to the paired PR review loop, gated by an explicit implementer `done` signal, so nit fixes pushed after `approved` still get reviewed.

**Architecture:** Two coupled Skill markdown files describe the wire protocol of a paired review loop. `approved` becomes provisional; the implementer's turn marker gains a `done=true|false` attribute that becomes the true convergence event. The reviewer stays armed after approving and disarms only on `done=true` (absent/unparseable `done` after approval is treated as `done=true`).

**Tech Stack:** Markdown Skill files (`SKILL.md`), `gh` CLI snippets, bash Monitor loops. No code, no test framework — verification is by inspecting the edited files and grepping for protocol-term consistency.

---

## Notes for the implementer

- This is a documentation change to two `SKILL.md` files. There is no test suite.
  "Verify" steps are `grep`/read checks confirming the edit landed and the
  protocol vocabulary stays consistent across both files.
- The two files must agree exactly on the wire protocol. The marker is
  `<!-- review-loop:implementer round=N done=true|false -->`. Convergence is the
  implementer posting `done=true`. After the reviewer's `approved`, an
  absent/unparseable `done` is treated as `done=true`.
- Source of truth for intent: `docs/superpowers/specs/2026-05-18-review-loop-done-signal-design.md`.
- The `README.md` and `skills/request-pr-review/reference/feedback-triage.md` need
  **no change** — confirmed in Task 3.
- Make each Edit exactly as written; the `old_string` blocks are the current file
  contents.

---

## Task 1: Update `skills/review-pr/SKILL.md` (reviewer side)

**Files:**
- Modify: `skills/review-pr/SKILL.md`

- [ ] **Step 1: Update the Wire protocol — implementer marker and convergence**

Replace this block:

```
- The author posts issue comments tagged `<!-- review-loop:implementer round=N -->`.
- **Convergence** = you post a review with `"verdict":"approved"` (`blocking` is 0).
```

with:

```
- The author posts issue comments tagged
  `<!-- review-loop:implementer round=N done=true|false -->`.
- **`approved` is provisional, not terminal.** When you post
  `"verdict":"approved"` (`blocking` is 0) the author may still act on nits, so
  you stay armed. **Convergence** = the author posts a turn marker with
  `done=true`. After your `approved`, an author turn whose `done` is absent or
  unparseable also counts as `done=true` (the loop terminates rather than
  hanging). The `done` field is consulted **only after you have approved** —
  during the `changes-requested` phase you always re-review and ignore it.
```

- [ ] **Step 2: Update the on-entry idempotency guard**

Replace this block:

```
2. **Idempotency guard:** count existing turn markers — reviewer `R`,
   implementer `I`. If the latest reviewer review is `approved` or `escalated`,
   stop. If `R > I`, you have already reviewed the latest changes — stop. (This
   guards a re-invocation in a fresh session; a running loop re-enters via the
   Monitor, not here.)
```

with:

```
2. **Idempotency guard:** count existing turn markers — reviewer `R`,
   implementer `I`. (This guards a re-invocation in a fresh session; a running
   loop re-enters via the Monitor, not here.)
   - Latest reviewer review is `escalated` → stop.
   - Latest reviewer review is `approved` → find the latest implementer turn
     posted after it: `done=true` (or absent/unparseable) → converged, stop;
     `done=false` → a post-approval re-review is pending, proceed; no such turn
     yet → arm the Monitor and wait.
   - Otherwise, if `R > I` you have already reviewed the latest changes — stop.
```

- [ ] **Step 3: Update Round 1 step 6 (round 1 never converges)**

Replace this line:

```
6. If converged or `--once`: stop. Otherwise arm the Monitor — **once**.
```

with:

```
6. If `--once`: stop. Otherwise arm the Monitor — **once**. Round 1 cannot
   converge: the author has not responded yet, so even an `approved` round 1
   stays armed and waits for the author's `done` signal.
```

- [ ] **Step 4: Rewrite the "Rounds 2+" section**

Replace this block:

```
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
```

with:

```
### Rounds 2+ — scoped re-review

The Monitor runs for the whole loop and emits one line per new author turn,
carrying that turn's `done` flag — do not re-arm it. On each notification, finish
any in-progress round first, then branch.

**If your last verdict was `approved`,** act on the `done` flag:
- `done=true` (or absent/unparseable) → **converged.** Go to Exit.
- `done=false` → the author pushed post-approval changes; run the re-review
  (steps 1–3 below). Nits are suppressed after round 1, so this re-review can
  only re-approve or surface a regression — it never requests new nit work.

**If your last verdict was `changes-requested`,** always run the re-review; the
`done` flag is not consulted in that phase.

1. `gh pr checkout <PR>` again to pull the author's new commits.
2. Dispatch a fresh `general-purpose` subagent given {your prior reviews, the
   author's rebuttal, the incremental diff `git diff <baseline>..HEAD`, and the
   files it touches}. Instruct it to:
   - review in **full file context** — read surrounding code, not just hunks;
   - **verify each prior blocking concern** is actually resolved;
   - **re-scan the new commits for regressions** — a fix can introduce a new bug;
   - report **blocking findings only** — new nitpicks are suppressed after round 1;
   - **not re-litigate** points the author rebutted soundly that you accepted.
3. Post the new review; re-record the baseline SHA. If the new verdict is
   `approved`, stay armed and wait for the author's next turn (their `done`
   signal).
```

- [ ] **Step 5: Rewrite the Exit section**

Replace this block:

```
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
```

with:

```
### Exit

Stop when **any** of:
- the author posts a turn marker with `done=true` (or, after your `approved`, a
  turn marker whose `done` is absent/unparseable) — **convergence**;
- you have posted **5** `changes-requested` reviews without resolution —
  `approved` reviews do not count toward this ceiling;
- a blocking finding stays contested for 2 rounds (author rebuts, you re-assert) —
  a stalemate, not progress.

On **convergence**, `TaskStop` the Monitor and print a round-by-round summary. Do
**not** post a closing review — the author's `done=true` comment is the closing
artifact on the PR.

On a non-converged exit (cap or stalemate), post a final review whose verdict
trailer is `"verdict":"escalated"`, listing what remains — this signals the
author's loop to stop too, so it does not wait forever on a reviewer that has
already left. Then `TaskStop` the Monitor and print a round-by-round summary.
```

- [ ] **Step 6: Update the Monitor to emit the `done` flag**

Replace this block:

````
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
````

with:

````
```bash
PR=<number>; seen=0
while true; do
  body=$(gh pr view "$PR" --json comments --jq '.comments[].body' 2>/dev/null || true)
  m=$(echo "$body" | grep -oP 'review-loop:implementer round=[0-9]+( done=(true|false))?' \
      | sort -t= -k2 -n | tail -1 || true)
  r=$(echo "$m" | grep -oP 'round=\K[0-9]+' || true)
  if [ -n "$r" ] && [ "$r" -gt "$seen" ]; then
    d=$(echo "$m" | grep -oP 'done=\K(true|false)' || true)
    echo "implementer round $r done=${d:-true}"; seen="$r"
  fi
  sleep 30
done
```

Each emitted line is your cue to act: `done=false` → run the next re-review
round; `done=true` → converge (see Exit). An absent `done` defaults to `true`.
````

- [ ] **Step 7: Add a Monitor convergence anti-pattern**

Replace this anti-pattern line:

```
- **Approving to end the loop.** Both sessions are the same model; the reviewer is
  tempted to converge prematurely. `approved` requires `blocking: 0` from an actual
  count, not a vibe.
```

with:

```
- **Approving to end the loop.** Both sessions are the same model; the reviewer is
  tempted to converge prematurely. `approved` requires `blocking: 0` from an actual
  count, not a vibe.
- **Disarming on your own `approved`.** Approval is provisional — the author may
  still push nit fixes. Stay armed until the author posts `done=true`.
```

- [ ] **Step 8: Verify the edits landed and the protocol vocabulary is consistent**

Run: `grep -n 'done=' skills/review-pr/SKILL.md`
Expected: at least 8 matching lines, spanning Wire protocol, idempotency guard,
Rounds 2+, Exit, Monitor, and anti-patterns.

Run: `grep -Fn 'review-loop:implementer round=N done=true|false' skills/review-pr/SKILL.md`
Expected: 1 match (the Wire protocol marker definition).

Run: `grep -Fnc 'verdict is `approved`;' skills/review-pr/SKILL.md`
Expected: 0 — the old "verdict is `approved`;" Exit bullet is gone.

- [ ] **Step 9: Commit**

```bash
git add skills/review-pr/SKILL.md
git commit -m "review-pr: stay armed after approval until implementer done-signal

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 2: Update `skills/request-pr-review/SKILL.md` (implementer side)

**Files:**
- Modify: `skills/request-pr-review/SKILL.md`

- [ ] **Step 1: Update the Wire protocol — turn marker and verdict note**

Replace this block:

```
- You post rebuttals via `gh pr comment <PR> --body "…"` — an issue comment whose
  body ends with a **turn marker**: `<!-- review-loop:implementer round=N -->`.
- The reviewer posts `COMMENT` reviews tagged `<!-- review-loop:reviewer round=N -->`,
  each ending with a verdict trailer:
  `<!-- review-verdict: {"verdict":"approved","round":N,"blocking":0,"nits":0} -->`.
  A single GitHub account cannot `--approve` its own PR, so the verdict is a marker.
- **Verdict values:** `approved` (converged), `escalated` (the reviewer stopped
  without convergence — see Exit), or `changes-requested`. A reviewer review with
  no parseable verdict trailer counts as `changes-requested`.
```

with:

```
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
```

- [ ] **Step 2: Update "Respond each round" step 5 to route `approved` to a new section**

Replace this block:

```
5. Parse the verdict trailer of the new review:
   - `"verdict":"approved"` → go to **Converged**;
   - `"verdict":"escalated"` → the reviewer has stopped without convergence; do
     **not** respond or keep waiting — go to **Exit without convergence**;
   - otherwise (`changes-requested`, or no parseable trailer) → triage and respond.
```

with:

```
5. Parse the verdict trailer of the new review:
   - `"verdict":"escalated"` → the reviewer has stopped without convergence; do
     **not** respond or keep waiting — go to **Exit without convergence**;
   - `"verdict":"approved"` → all blocking findings are clear, but the review may
     carry nits and the loop is **not** closed yet — go to **Respond to approval**;
   - otherwise (`changes-requested`, or no parseable trailer) → triage and respond
     (steps 6–8); your rebuttal's turn marker carries `done=false`.
```

- [ ] **Step 3: Add `done=false` to the round-rebuttal step 8**

Replace this block:

```
8. Post a rebuttal with `gh pr comment` ending in your turn marker:
   - **Fixed:** what changed (and, if you diverged from the reviewer's suggestion, why);
   - **Skipped:** explicit rationale for each — complexity cost, scope, or a better
     alternative.
   Post the rebuttal **even if every point was skipped** — it closes the loop and
   shows each point was weighed. Then keep monitoring.
```

with:

```
8. Post a rebuttal with `gh pr comment` ending in your turn marker
   `<!-- review-loop:implementer round=N done=false -->`:
   - **Fixed:** what changed (and, if you diverged from the reviewer's suggestion, why);
   - **Skipped:** explicit rationale for each — complexity cost, scope, or a better
     alternative.
   Post the rebuttal **even if every point was skipped** — it shows each point was
   weighed and cues the reviewer's next round. Then keep monitoring.
```

- [ ] **Step 4: Insert the "Respond to approval" section before "Converged"**

Insert this new section immediately before the `### Converged` heading:

```
### Respond to approval

`approved` clears every blocking finding, but its nits are unaddressed and the
loop stays open until you post `done=true`. Do **not** jump straight to Converged.

5a. Triage the nits exactly as in step 6 — **read the bundled
    `reference/feedback-triage.md` first** — verifying each against the code and
    pushing back on the unsound ones.
5b. Fork on the outcome:
    - **You made no code changes** (every nit declined or deferred) → go to
      **Converged**; its comment carries `done=true`.
    - **You made code changes** → commit as `address review nits (round N)`, then
      `git push`, then post a rebuttal whose turn marker is
      `<!-- review-loop:implementer round=N done=false -->` (Fixed/Skipped
      sections as in step 8). Keep monitoring — the reviewer re-reviews the new
      commits, then re-approves (or flags a regression). On the next `approved`,
      return to **Respond to approval**; on `changes-requested`, triage and
      respond via steps 6–8.
```

- [ ] **Step 5: Rewrite the "Converged" section so the converge comment carries `done=true`**

Replace this block:

```
### Converged

9. Post a brief plain comment: `Converged at round N — all blocking items resolved.`
   `TaskStop` the Monitor (use the task id captured when you armed it).
10. Notify the user with a round-by-round summary, then surface — **do not run** —
    the primed merge command:
```

with:

```
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
```

- [ ] **Step 6: Rewrite the "Exit without convergence" cap**

Replace this block:

```
Stop, `TaskStop` the Monitor, and escalate to the user — stating the open
disagreement and what each side last said — when **either**:
- a reviewer review carries `"verdict":"escalated"` (the reviewer hit its own cap
  or a stalemate); or
- you have responded to **5** reviewer reviews without an `approved` verdict (stop
  after posting your round-5 rebuttal).
```

with:

```
Stop, `TaskStop` the Monitor, and escalate to the user — stating the open
disagreement and what each side last said — when **either**:
- a reviewer review carries `"verdict":"escalated"` (the reviewer hit its own cap
  or a stalemate); or
- you have responded to **5** `changes-requested` reviews without reaching an
  `approved` verdict (stop after posting that 5th rebuttal). Post-approval
  `done=false` rounds do not count toward this cap — once the reviewer has
  approved, converge via `done=true` rather than escalating.
```

- [ ] **Step 7: Update `--once` mode for the approval path**

Replace this block:

```
`request-pr-review --once` — open the PR (if needed) and respond to the current
review once, without arming the Monitor. If no review exists yet, it just opens
the PR.
```

with:

```
`request-pr-review --once` — open the PR (if needed) and respond to the current
review once, without arming the Monitor. If the current review is `approved`,
run **Respond to approval** once (triage nits, then converge with `done=true` or
post a `done=false` rebuttal). If no review exists yet, it just opens the PR.
```

- [ ] **Step 8: Add an anti-pattern for treating `approved` as the finish line**

Replace this anti-pattern line:

```
- **Auto-merging on approval.** The squash-merge is primed but waits for the user.
```

with:

```
- **Auto-merging on approval.** The squash-merge is primed but waits for the user.
- **Treating `approved` as the finish line.** `approved` only clears blocking
  findings; the loop closes when you post `done=true`. Triage the nits first.
```

- [ ] **Step 9: Verify the edits landed and the protocol vocabulary is consistent**

Run: `grep -n 'done=' skills/request-pr-review/SKILL.md`
Expected: at least 9 matching lines, spanning Wire protocol, step 5, step 8,
Respond to approval, Converged, Exit, `--once`, and anti-patterns.

Run: `grep -Fn 'review-loop:implementer round=N done=true|false' skills/request-pr-review/SKILL.md`
Expected: 1 match (the Wire protocol marker definition).

Run: `grep -Fn 'Respond to approval' skills/request-pr-review/SKILL.md`
Expected: at least 3 matches — the step-5 route, the section heading, and the
loop-back reference in section 5b.

- [ ] **Step 10: Commit**

```bash
git add skills/request-pr-review/SKILL.md
git commit -m "request-pr-review: triage nits after approval, converge via done-signal

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 3: Cross-skill consistency check

**Files:**
- Read-only: `skills/review-pr/SKILL.md`, `skills/request-pr-review/SKILL.md`, `README.md`, `skills/request-pr-review/reference/feedback-triage.md`

- [ ] **Step 1: Confirm both skills define the marker identically**

Run: `grep -hn 'review-loop:implementer round=N done=true|false' skills/review-pr/SKILL.md skills/request-pr-review/SKILL.md`
Expected: the marker string `<!-- review-loop:implementer round=N done=true|false -->` appears once in each file, character-for-character identical.

- [ ] **Step 2: Confirm both skills agree on the convergence definition**

Read the Wire protocol of each file. Confirm both state: convergence = the
implementer posts `done=true`; an absent/unparseable `done` after `approved` is
treated as `done=true`; `done` is consulted only after the reviewer has approved.
If either file disagrees, fix it to match the spec and amend the relevant commit.

- [ ] **Step 3: Confirm both skills agree the caps count `changes-requested` reviews only**

Run: `grep -n 'changes-requested' skills/review-pr/SKILL.md skills/request-pr-review/SKILL.md | grep -i '5'`
Expected: the reviewer Exit cap and the implementer Exit cap both count `5` `changes-requested` reviews, not all reviews.

- [ ] **Step 4: Confirm `README.md` needs no change**

Read `README.md`. Confirm it describes convergence generically ("converging on an
explicit verdict", "until the review converges") and never claims `approved` is
itself terminal. No edit expected. If it does claim that, update the wording to
match the new protocol and commit.

- [ ] **Step 5: Confirm `feedback-triage.md` is untouched**

Run: `git status --short skills/request-pr-review/reference/feedback-triage.md`
Expected: empty output — the bundled reference is unchanged (the design states
feedback-reception discipline is unaffected).

- [ ] **Step 6: Final commit if Task 3 produced fixes**

If steps 2 or 4 required edits, commit them:

```bash
git add -A
git commit -m "review-loop: align skill protocol wording across both sides

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

If no fixes were needed, skip this step.
