# Review loop: post-approval round via an implementer done-signal

**Date:** 2026-05-18
**Skills affected:** `review-pr` (reviewer), `request-pr-review` (author/implementer)

## Problem

In the paired PR review loop, both sides today treat a reviewer verdict of
`approved` (blocking=0) as terminal. The reviewer disarms its Monitor the moment
it posts `approved`, even when that review carries `nits > 0`. If the implementer
then acts on those nits and pushes commits, no one reviews the change — the
reviewer has already left.

## Goal

Add one more reviewable round after approval. `approved` becomes *provisional*;
true convergence is an explicit signal from the implementer that it is finished
and intends no further changes. The reviewer stays armed across the gap and
re-reviews any post-approval commits before disarming.

## Approach

Considered two detection mechanisms:

1. **SHA check** — reviewer records HEAD at approval, re-reviews if it moved.
   No new marker, but races an in-flight push and cannot distinguish "declined
   all nits" from "still working."
2. **Explicit done-signal** — the implementer posts a parseable flag.

Chosen: the explicit done-signal. It is unambiguous and race-free.

## Design

### 1. Wire protocol change

The implementer's turn marker gains a `done` attribute:

```
<!-- review-loop:implementer round=N done=true -->
```

- `done=false` — "I pushed changes / I am still responding; look again." Set on
  every `changes-requested` rebuttal and on any post-approval rebuttal that
  carries new commits.
- `done=true` — "I am finished, no more changes coming." Posted only when the
  latest reviewer verdict is `approved` and the implementer is making no further
  changes. Its presence *is* the convergence event.
- Absent / unparseable — treated as `done=true` (backward-compat and
  malformed-post fallback; the loop always terminates rather than hanging).

The reviewer consults `done` **only after it has posted `approved`**. During the
`changes-requested` phase the field is ignored — the reviewer always re-reviews.

### 2. Convergence point shifts

`approved` (blocking=0) is now provisional, not terminal. True convergence = the
implementer posts `done=true`. Both Monitors stay armed across the gap between
`approved` and `done=true`.

### 3. Implementer loop change

On a reviewer verdict of `approved`, the implementer no longer jumps straight to
"Converged." It triages the nits using the same feedback-triage discipline, then
forks:

- **Made no changes** (declined or deferred all nits) → post the converge
  comment carrying `done=true` → disarm, notify the user, surface the primed
  merge command. (This is today's "Converged" step; the converge comment now
  also carries the done-signal.)
- **Made nit changes** → commit, push, post a rebuttal with `done=false` → keep
  monitoring; the reviewer will re-review.

After a `done=false` round the reviewer re-approves (or finds a regression). The
implementer forks again, converging to `done=true` once it stops touching code.

### 4. Reviewer loop change

After posting `approved`, the reviewer does **not** exit. It keeps the Monitor
armed — now with a **bounded post-approval wait** — and waits for the next
implementer round:

- `done=true` (or absent) → converged: `TaskStop` the Monitor, print the
  round-by-round summary, stop silently — no closing PR post. The implementer's
  `done=true` comment is the closing artifact on the PR.
- `done=false` → run one re-review round over the new commits — regression-
  focused, blocking-only (post-round-1 nit suppression already applies, so this
  phase cannot spawn fresh nits to chase). Post `approved` again, or
  `changes-requested` if the nit fix introduced a bug (which drops back into the
  normal loop).
- **post-approval wait elapsed** — if no implementer turn arrives within the
  wait window (the implementer session is slow or dead), the reviewer disarms
  and treats the PR as converged. Safe: the code was already approved.

The reviewer's Monitor emits the `done` flag alongside the round number,
mirroring how the implementer's Monitor already emits the verdict; it also
tracks the post-approval wait and emits a timeout line when the window elapses.

### 5. Caps, bounds, and entry guards

- **Reviewer escalation cap** — counts only `changes-requested` reviews, so
  repeated `approved`s never false-trigger an escalation. Five `changes-requested`
  reviews without resolution → post `escalated`. The stalemate rule (a blocking
  finding contested for 2 rounds) is unchanged.
- **Implementer escalation cap** — likewise counts only `changes-requested`
  reviewer reviews; post-approval `done=false` rounds do not push toward
  escalation.
- **Post-approval wait** — after posting `approved`, the reviewer's Monitor
  waits a bounded window (≈20 minutes of 30-second polls) for the implementer's
  turn. A `done=false` turn resets the window for the next approval; if the
  window elapses with no turn, the reviewer disarms and treats the PR as
  converged. This closes the deadlock where a dead implementer session would
  otherwise leave the reviewer waiting forever. The pre-approval
  (`changes-requested`) wait stays unbounded — only the post-approval phase is
  bounded.
- **Post-approval round cap** — the reviewer runs at most **3** post-approval
  re-review rounds (`done=false` turns answered after an `approved`). On a 4th,
  it disarms and treats the PR as converged rather than chasing further
  voluntary polish. A post-approval re-review that surfaces a regression exits
  this phase into the normal `changes-requested` loop (bounded by the escalation
  cap); the post-approval counter no longer applies until the next `approved`.
- **Reviewer entry idempotency guard** — when the latest reviewer review is
  `approved`, check for the most recent implementer turn (by position in the
  comment list) posted after it: `done=true` (or absent) → converged, stop;
  `done=false` → re-review pending, proceed; none → arm the Monitor (with the
  bounded post-approval wait) and wait.

### 6. `--once` mode

Unchanged in spirit on both sides — each invocation does one step. The
implementer emits the `done` field so a fresh counterpart `--once` invocation can
interpret it.

## Files touched

- `skills/review-pr/SKILL.md` — wire protocol, loop rounds 2+, Exit, entry
  guard, Monitor, anti-patterns.
- `skills/request-pr-review/SKILL.md` — wire protocol, respond-each-round fork,
  Converged, Exit, anti-patterns.
- `README.md` — loop description, if it states `approved` = converged.
- `skills/request-pr-review/reference/feedback-triage.md` — no change; feedback-
  reception discipline is unaffected.

## Out of scope

- A bounded wait on the reviewer's **pre-approval** (`changes-requested`)
  Monitor. That wait stays unbounded, consistent with the existing loop; only
  the post-approval phase is bounded by this design (see §5).
- A bounded wait on the **implementer**'s Monitor. If the implementer posts
  `done=false` and the reviewer has already disarmed, the implementer waits as
  it does today — the existing "both sessions stay alive" assumption still
  holds on that side.
- Changes to the `escalated` path or the stalemate detection logic.
