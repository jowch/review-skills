# review-skills Package Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Package the existing `review-pr` and `request-pr-review` skills into a standalone public GitHub repo installable via `npx skills add jowch/review-skills` and via Claude Code's plugin marketplace.

**Architecture:** A single repo at `~/projects/review-skills` (already created, git-initialized, with the design spec committed under `docs/`). Two skills live under `skills/<name>/SKILL.md`. `request-pr-review` carries a bundled, verbatim reference file (`reference/feedback-triage.md`) so it has zero dependency on the `superpowers` plugin. The repo doubles as a Claude Code plugin marketplace via `.claude-plugin/marketplace.json` + `.claude-plugin/plugin.json`.

**Tech Stack:** Markdown skill files, JSON plugin manifests, the vercel-labs `skills` CLI (`npx skills`), `git`/`gh`.

---

## Source files (read-only inputs — do not modify the originals)

- `~/.claude/skills/review-pr/SKILL.md` — reviewer skill, copied verbatim.
- `~/.claude/skills/request-pr-review/SKILL.md` — author skill, copied then one edit.
- `~/.claude/plugins/cache/claude-plugins-official/superpowers/<version>/skills/receiving-code-review/SKILL.md` — verbatim source for the bundled reference. As of this writing `<version>` is `5.1.0`; if that path is gone, find the current one with:
  `ls -d ~/.claude/plugins/cache/claude-plugins-official/superpowers/*/skills/receiving-code-review/SKILL.md`

## Target file structure

```
~/projects/review-skills/
├── README.md                              # Task 6
├── LICENSE                                # Task 1
├── .claude-plugin/
│   ├── marketplace.json                   # Task 5
│   └── plugin.json                        # Task 5
├── docs/superpowers/                      # already present (spec + this plan)
└── skills/
    ├── review-pr/
    │   └── SKILL.md                       # Task 2 — verbatim copy
    └── request-pr-review/
        ├── SKILL.md                       # Task 3 — copy + step-6 edit
        └── reference/
            └── feedback-triage.md         # Task 4 — verbatim receiving-code-review + header
```

All shell steps assume the working directory is `~/projects/review-skills` unless stated otherwise.

---

## Task 1: Scaffold directories and LICENSE

**Files:**
- Create: `~/projects/review-skills/skills/review-pr/` (directory)
- Create: `~/projects/review-skills/skills/request-pr-review/reference/` (directory)
- Create: `~/projects/review-skills/.claude-plugin/` (directory)
- Create: `~/projects/review-skills/LICENSE`

- [ ] **Step 1: Create the directory tree**

```bash
cd ~/projects/review-skills
mkdir -p skills/review-pr skills/request-pr-review/reference .claude-plugin
```

- [ ] **Step 2: Create `LICENSE`** with this exact content:

```
MIT License

Copyright (c) 2026 Jonathan Chen

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **Step 3: Verify the structure**

Run: `ls -d skills/review-pr skills/request-pr-review/reference .claude-plugin && test -f LICENSE && echo OK`
Expected: the three directory paths print, then `OK`.

- [ ] **Step 4: Commit**

```bash
git add LICENSE
git commit -m "Add MIT license"
```

---

## Task 2: Add the review-pr skill (verbatim copy)

`review-pr` has no external skill dependencies, so it is copied unchanged.

**Files:**
- Create: `~/projects/review-skills/skills/review-pr/SKILL.md` (copy of `~/.claude/skills/review-pr/SKILL.md`)

- [ ] **Step 1: Copy the skill file verbatim**

```bash
cd ~/projects/review-skills
cp ~/.claude/skills/review-pr/SKILL.md skills/review-pr/SKILL.md
```

- [ ] **Step 2: Verify the copy is byte-identical**

Run: `diff ~/.claude/skills/review-pr/SKILL.md skills/review-pr/SKILL.md && echo IDENTICAL`
Expected: `IDENTICAL` (no diff output).

- [ ] **Step 3: Verify required frontmatter is present**

Run: `head -12 skills/review-pr/SKILL.md`
Expected: a YAML frontmatter block (`---`) containing a `name:` line (`review-pr`) and a `description:` line. Both are required by the `skills` CLI.

- [ ] **Step 4: Commit**

```bash
git add skills/review-pr/SKILL.md
git commit -m "Add review-pr skill"
```

---

## Task 3: Add the request-pr-review skill (copy + re-point step 6)

The only behavior-affecting edit in the whole package: step 6 currently requires the `superpowers:receiving-code-review` sub-skill; re-point it to the bundled reference file created in Task 4.

**Files:**
- Create: `~/projects/review-skills/skills/request-pr-review/SKILL.md` (copy of `~/.claude/skills/request-pr-review/SKILL.md`, with one edit)

- [ ] **Step 1: Copy the skill file**

```bash
cd ~/projects/review-skills
cp ~/.claude/skills/request-pr-review/SKILL.md skills/request-pr-review/SKILL.md
```

- [ ] **Step 2: Re-point step 6 to the bundled reference**

In `skills/request-pr-review/SKILL.md`, find this exact text (it is the start of step 6):

```
6. Triage every point. **REQUIRED SUB-SKILL: use
   `superpowers:receiving-code-review`.** For each point:
```

Replace it with:

```
6. Triage every point. **REQUIRED REFERENCE: read the bundled
   `reference/feedback-triage.md` before triaging — it carries the
   feedback-reception discipline this step depends on.** For each point:
```

- [ ] **Step 3: Verify the superpowers reference is gone and the new one is present**

Run: `grep -n 'superpowers:receiving-code-review' skills/request-pr-review/SKILL.md; echo "exit=$?"`
Expected: no matching lines, `exit=1` (grep found nothing).

Run: `grep -n 'reference/feedback-triage.md' skills/request-pr-review/SKILL.md`
Expected: exactly one matching line, inside step 6.

- [ ] **Step 4: Verify nothing else changed**

Run: `diff <(sed 's#superpowers:receiving-code-review#X#' ~/.claude/skills/request-pr-review/SKILL.md) skills/request-pr-review/SKILL.md`
Expected: the diff shows ONLY the step-6 block changing (the two-to-three lines of the `REQUIRED SUB-SKILL` / `REQUIRED REFERENCE` text). No other hunks. If any other hunk appears, the copy was altered unintentionally — revert and redo Steps 1–2.

- [ ] **Step 5: Commit**

```bash
git add skills/request-pr-review/SKILL.md
git commit -m "Add request-pr-review skill, re-pointed to bundled reference"
```

---

## Task 4: Add the bundled feedback-triage.md reference

A **verbatim** copy of the superpowers `receiving-code-review` skill, with an attribution header prepended above the copied content. It is named `feedback-triage.md` (not `SKILL.md`), so it is a plain reference document and never registers as a skill.

**Files:**
- Create: `~/projects/review-skills/skills/request-pr-review/reference/feedback-triage.md`

- [ ] **Step 1: Copy the receiving-code-review skill verbatim**

```bash
cd ~/projects/review-skills
SRC=$(ls ~/.claude/plugins/cache/claude-plugins-official/superpowers/*/skills/receiving-code-review/SKILL.md | sort -V | tail -1)
echo "copying from: $SRC"
cp "$SRC" skills/request-pr-review/reference/feedback-triage.md
```

- [ ] **Step 2: Prepend the attribution header**

Insert the following block at the very TOP of `skills/request-pr-review/reference/feedback-triage.md`, above the existing leading `---` line (use the Edit tool: match the current first line of the file, which is `---`, and replace it with the header followed by `---`):

```
<!--
=============================================================================
feedback-triage.md

VERBATIM COPY of the `receiving-code-review` skill from the superpowers
project, bundled with the request-pr-review skill as a reference document so
that request-pr-review is self-contained (no dependency on the superpowers
plugin being installed).

  Source:    https://github.com/obra/superpowers
  License:   MIT
  Copyright: (c) 2025 Jesse Vincent

This is a REFERENCE DOCUMENT, not a registered skill. Skill discovery keys on
the filename `SKILL.md`; this file is `feedback-triage.md`, so the `name:` /
`description:` frontmatter below is inert and cannot collide with the
upstream `receiving-code-review` skill.

The MIT license requires this copyright and permission notice to accompany
copies. See the superpowers repository for the full MIT license text.
=============================================================================
-->

---
```

- [ ] **Step 3: Verify the body below the header is byte-identical to the source**

The attribution header in Step 2 is 23 lines (22 HTML-comment lines + 1 blank line); the verbatim content therefore begins at line 24.

```bash
SRC=$(ls ~/.claude/plugins/cache/claude-plugins-official/superpowers/*/skills/receiving-code-review/SKILL.md | sort -V | tail -1)
# Drop the 23-line header, compare the rest to the source.
diff <(tail -n +24 skills/request-pr-review/reference/feedback-triage.md) "$SRC" && echo "BODY IDENTICAL"
```

Expected: `BODY IDENTICAL`. If the diff is non-empty, count the actual header lines you inserted (`grep -n -m1 '^---$' skills/request-pr-review/reference/feedback-triage.md` gives the line number of the verbatim content's first line) and set the `tail -n +N` offset to that line number, then re-run. The check passes when the only thing added to the source file is the header.

- [ ] **Step 4: Verify the file is NOT discoverable as a skill**

Run: `basename skills/request-pr-review/reference/feedback-triage.md`
Expected: `feedback-triage.md` — confirming it is not named `SKILL.md` and therefore not a skill.

- [ ] **Step 5: Commit**

```bash
git add skills/request-pr-review/reference/feedback-triage.md
git commit -m "Bundle receiving-code-review verbatim as feedback-triage reference"
```

---

## Task 5: Add the plugin manifests

Two JSON files in `.claude-plugin/`: `marketplace.json` (the marketplace catalog) and `plugin.json` (the single plugin's manifest). The repo is simultaneously a marketplace and one plugin rooted at `./`. Skills are auto-discovered from `skills/<name>/SKILL.md` — no skill list is needed in either file.

**Files:**
- Create: `~/projects/review-skills/.claude-plugin/marketplace.json`
- Create: `~/projects/review-skills/.claude-plugin/plugin.json`

- [ ] **Step 1: Create `.claude-plugin/marketplace.json`** with this exact content:

```json
{
  "name": "review-skills",
  "owner": {
    "name": "Jonathan Chen"
  },
  "description": "A paired, multi-round pull-request review loop for Claude Code.",
  "plugins": [
    {
      "name": "review-skills",
      "source": "./",
      "description": "Paired PR-review loop: review-pr (reviewer) and request-pr-review (author).",
      "version": "1.0.0",
      "author": {
        "name": "Jonathan Chen"
      },
      "license": "MIT"
    }
  ]
}
```

- [ ] **Step 2: Create `.claude-plugin/plugin.json`** with this exact content:

```json
{
  "name": "review-skills",
  "description": "Paired PR-review loop: review-pr (reviewer) and request-pr-review (author). The two skills run in separate sessions and converge through the PR.",
  "version": "1.0.0",
  "author": {
    "name": "Jonathan Chen"
  },
  "license": "MIT",
  "keywords": ["code-review", "pull-request", "github", "workflow"]
}
```

- [ ] **Step 3: Verify both files are valid JSON**

Run: `node -e "JSON.parse(require('fs').readFileSync('.claude-plugin/marketplace.json','utf8')); JSON.parse(require('fs').readFileSync('.claude-plugin/plugin.json','utf8')); console.log('VALID JSON')"`
Expected: `VALID JSON`. (If `node` is unavailable, use `python3 -c "import json; json.load(open('.claude-plugin/marketplace.json')); json.load(open('.claude-plugin/plugin.json')); print('VALID JSON')"`.)

- [ ] **Step 4: Commit**

```bash
git add .claude-plugin/marketplace.json .claude-plugin/plugin.json
git commit -m "Add Claude Code plugin marketplace manifests"
```

---

## Task 6: Write README.md

**Files:**
- Create: `~/projects/review-skills/README.md`

- [ ] **Step 1: Create `README.md`** with this exact content:

````markdown
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
monitoring loop. See each skill's `SKILL.md` for the full protocol.

## License

MIT — see [`LICENSE`](./LICENSE).

### Third-party content

`skills/request-pr-review/reference/feedback-triage.md` is a verbatim copy of
the `receiving-code-review` skill from the
[superpowers](https://github.com/obra/superpowers) project, used under the MIT
License, Copyright (c) 2025 Jesse Vincent. It is bundled so that
`request-pr-review` is self-contained.
````

- [ ] **Step 2: Verify the README references resolve**

Run: `grep -c 'jowch/review-skills' README.md`
Expected: a count of `3` or more (install commands reference the repo).

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "Add README"
```

---

## Task 7: Verify skill discovery with the `skills` CLI

This is the package's acceptance test: the `skills` CLI must discover exactly the two skills from the local repo.

**Files:** none (verification only).

- [ ] **Step 1: List skills from the local path**

```bash
cd ~/projects/review-skills
npx --yes skills add . --list
```

Expected: the command lists exactly two skills — `review-pr` and `request-pr-review`. It must NOT list a third skill, and must NOT list anything named `feedback-triage` or `receiving-code-review` (that file is a reference doc, not a skill).

- [ ] **Step 2: If discovery fails or the count is wrong, diagnose**

- "No skills found" → check each `skills/<name>/SKILL.md` has valid YAML frontmatter with both `name` and `description` (re-run Task 2 Step 3 / Task 3 Step 3).
- A third skill listed → check that `reference/feedback-triage.md` was not accidentally created as `SKILL.md` (Task 4 Step 4).
- Fix the cause, then re-run Step 1 until it lists exactly the two skills.

- [ ] **Step 3: No commit** — this task only verifies. Proceed once Step 1 passes.

---

## Task 8: Publish to GitHub

Creating a public repo is outward-facing. **Confirm with the user before running Step 2.**

**Files:** none (publishing only).

- [ ] **Step 1: Confirm the working tree is fully committed**

Run: `cd ~/projects/review-skills && git status --short`
Expected: no output (clean tree). If anything is uncommitted, commit it before proceeding.

- [ ] **Step 2: Create the GitHub repo and push** (only after explicit user go-ahead)

```bash
cd ~/projects/review-skills
git branch -M main
gh repo create jowch/review-skills --public --source=. --remote=origin --push
```

Expected: the repo is created at `https://github.com/jowch/review-skills` and `main` is pushed.

- [ ] **Step 3: Verify the published install works**

```bash
npx --yes skills add jowch/review-skills --list
```

Expected: lists `review-pr` and `request-pr-review` from the GitHub shorthand — same result as Task 7 Step 1, now proving the remote install path.

- [ ] **Step 4: Report the install commands to the user**

Tell the user the repo URL and the two install commands:
- `npx skills add jowch/review-skills`
- `/plugin marketplace add jowch/review-skills`

---

## Done criteria

- `npx skills add jowch/review-skills --list` lists exactly `review-pr` and `request-pr-review`.
- `request-pr-review/SKILL.md` contains no reference to `superpowers:receiving-code-review`.
- `feedback-triage.md` body is byte-identical to the upstream skill and carries the MIT attribution header.
- The repo is public on GitHub with a README, LICENSE, and `.claude-plugin/` manifests.
