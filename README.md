# review-skills

A paired, multi-round **pull-request review loop** for Claude Code, packaged as
two skills:

- **`review-pr`** — the *reviewer*. Reviews a PR in full on round 1, then
  re-reviews narrowly on later rounds, converging on an explicit verdict.
- **`request-pr-review`** — the *author*. Opens the PR, triages reviewer
  feedback with technical rigor, pushes back where the reviewer is wrong, and
  responds round by round until the review converges.

The two skills run in **separate Claude Code sessions** and communicate only
through the pull request itself. Both require an authenticated [`gh`
CLI](https://cli.github.com/).

## Install

**Prerequisite:** both skills drive an authenticated [GitHub CLI](https://cli.github.com/). Install it and run `gh auth login` before using them.

### With `npx skills`

```bash
# Install both skills
npx skills add jowch/review-skills

# Or pick one
npx skills add jowch/review-skills --skill review-pr
```

### As a Claude Code plugin marketplace

```text
/plugin marketplace add jowch/review-skills
/plugin install review-skills@review-skills
```

## Usage

In the session that owns the branch:

> "Open a PR and get it reviewed" — triggers `request-pr-review`.

In a second session you start:

> "Review PR #123" — triggers `review-pr`.

Each skill also supports a `--once` mode for a single round without the
monitoring loop. See [`review-pr`](./skills/review-pr/SKILL.md) and
[`request-pr-review`](./skills/request-pr-review/SKILL.md) for the full protocol.

## License

MIT — see [`LICENSE`](./LICENSE).

### Third-party content

`skills/request-pr-review/reference/feedback-triage.md` is a verbatim copy of
the `receiving-code-review` skill from the
[superpowers](https://github.com/obra/superpowers) project, used under the MIT
License, Copyright (c) 2025 Jesse Vincent. It is bundled so that
`request-pr-review` is self-contained.
