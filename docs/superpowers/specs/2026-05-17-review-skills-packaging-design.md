# review-skills — portable PR-review skill package

**Date:** 2026-05-17
**Status:** Approved design, ready for implementation planning

## Goal

Package the two PR-review skills currently living in `~/.claude/skills/`
(`review-pr` and `request-pr-review`) into a standalone public GitHub repo
that anyone can install with the vercel-labs `skills` CLI
(`npx skills add jowch/review-skills`) and, secondarily, with Claude Code's
native plugin marketplace (`/plugin marketplace add jowch/review-skills`).

The package must be **self-contained** — zero dependency on any other plugin
or skill being installed.

## Background

The two skills implement a paired, multi-round PR-review loop:

- `review-pr` — the reviewer half. Standalone; no external skill dependencies.
- `request-pr-review` — the author half.

**Problem this design removes:** in the *source copy* of `request-pr-review`
(at `~/.claude/skills/`), step 6 today reads *"REQUIRED SUB-SKILL: use
`superpowers:receiving-code-review`."* That sub-skill ships in the
`superpowers` plugin (`github.com/obra/superpowers`, MIT © 2025 Jesse
Vincent), which a `npx skills` user will not have installed. The packaged
copy eliminates this reference (see Self-containment, below), so **the
finished `review-skills` package depends on superpowers neither at install
time nor at runtime.** Superpowers is named again only under Licensing —
purely as the attribution source for derived content, which is a copyright
credit, not a dependency.

The vercel-labs `skills` CLI discovers skills by looking for `SKILL.md` files
(standard locations, `.claude-plugin/marketplace.json`/`plugin.json`
declarations, then recursive search). A skill is any directory with a
`SKILL.md` carrying `name` + `description` frontmatter. Plain `.md` files
without that frontmatter are **not** skills and are never registered.

## Design

### Repository layout

Built locally at `~/projects/review-skills`, pushed to a new public GitHub
repo named `review-skills`.

```
review-skills/
├── README.md
├── LICENSE                       # MIT — covers the two original skills
├── .claude-plugin/
│   └── marketplace.json          # declares both skills for /plugin discovery
├── docs/
│   └── superpowers/specs/        # this design doc
└── skills/
    ├── review-pr/
    │   └── SKILL.md              # copied verbatim from the source skill
    └── request-pr-review/
        ├── SKILL.md              # step 6 re-pointed to the bundled reference
        └── reference/
            └── feedback-triage.md   # condensed triage guidance
```

The `skills/` subdirectory keeps the repo root clean and matches the
`vercel-labs/agent-skills` convention.

### Exactly two skills

The package contains **two and only two** skills: `review-pr` and
`request-pr-review`. No third skill is created. Bundling
`receiving-code-review` as its own `SKILL.md` was explicitly rejected — it
would duplicate the `superpowers:receiving-code-review` skill name for any
user who has both packages installed, confusing skill discovery.

### Self-containment via a reference file

`request-pr-review/SKILL.md` step 6 currently points to
`superpowers:receiving-code-review`. It will instead point, via relative
path, to a bundled reference file:

`skills/request-pr-review/reference/feedback-triage.md`

This file is **not a skill** — it has no `SKILL.md` name and no frontmatter,
so it never registers in the skill registry and cannot collide with the
upstream `superpowers:receiving-code-review` skill. It is reachable only
through the link inside `request-pr-review/SKILL.md`.

The filename is deliberately `feedback-triage.md`, not `receiving-code-review.md`,
so it bears no resemblance — by name or by skill identity — to the upstream
skill.

Its content is a **condensed** version of the essential guidance from
`superpowers:receiving-code-review` (213 lines): the read → understand →
verify response pattern, verify-before-implementing, and the rules for
pushing back on incorrect or out-of-scope feedback. It is scoped to what
`request-pr-review`'s triage step actually needs.

`review-pr/SKILL.md` has no external skill dependencies and is copied verbatim.

### Licensing and attribution

The repo is MIT licensed. The two skills are the user's own work, so the
`LICENSE` file is a plain MIT notice for them.

`feedback-triage.md` is derived from MIT-licensed superpowers content. MIT
requires the upstream copyright notice to travel with substantial derived
portions. Therefore:

- `feedback-triage.md` opens with a one-line attribution header:
  *"Condensed from the `receiving-code-review` skill in obra/superpowers,
  MIT © 2025 Jesse Vincent — https://github.com/obra/superpowers"*
- The `README.md` License section adds a "Third-party content" note linking
  to the superpowers repo.

### Dual installation

- **`npx skills`** — works with no extra metadata; the CLI's recursive search
  finds both `SKILL.md` files under `skills/`.
- **Claude Code plugin marketplace** — `.claude-plugin/marketplace.json`
  declares the two skills so the repo also works via
  `/plugin marketplace add jowch/review-skills`. The exact `marketplace.json`
  schema will be verified against current Claude Code plugin-marketplace docs
  during implementation rather than assumed.

### README contents

- One-line description of the paired review loop.
- Install instructions for both paths (`npx skills add` and `/plugin`).
- Per-skill summary (`review-pr`, `request-pr-review`) and the note that the
  two skills run in **separate sessions**.
- Prerequisite: an authenticated `gh` CLI.
- License section with the third-party-content attribution note.

## Verification

- Before pushing: `npx skills add ~/projects/review-skills --list` lists both
  skills from the local path.
- After pushing: the same command against the `jowch/review-skills`
  shorthand lists both skills.

## Out of scope (YAGNI)

- No CI workflow.
- No skill versioning scheme.
- No submission to the skills.sh registry.
- No changes to skill *behavior* — the only edit to skill content is
  re-pointing `request-pr-review` step 6 from the superpowers sub-skill to the
  bundled reference file.
