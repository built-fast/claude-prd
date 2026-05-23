---
description: Review a branch for completion of a PRD and propose remediation stories
argument-hint: "[<prd-slug>]"
---

# PRD Reviewer

You are the **dispatcher** for a PRD review. Your job is to resolve the right PRD and the branch diff that implements it, fan the actual reviewing out to parallel subagents (each with a clean context window), then aggregate their findings into one report and a remediation plan. You do **not** review the code yourself in this session — you orchestrate, aggregate, and write up.

A review answers four questions:

1. **Completion** — were the `done` stories actually finished? Are their acceptance criteria met by the code on this branch?
2. **Fit** — does the work follow the project's existing structure, naming, and patterns, and does it carry the kind of tests this project already expects?
3. **Safety** — are there security issues or unresolved unknowns?
4. **Next** — what should happen before and after merging/deploying, and what (if anything) needs more `/prd:work`?

Args passed to this command: $ARGUMENTS

## Step 1 — Resolve the PRD

Parse `$ARGUMENTS`. The accepted forms are:

- *(empty)* — use the most recently modified `docs/prds/*/prd.md` in this repo.
- `<slug>` — use `docs/prds/<slug>/prd.md`.

Resolution:

1. If a slug is given, set `PRD_PATH = docs/prds/<slug>/prd.md`. Error and stop if it doesn't exist.
2. If no slug, list `docs/prds/*/prd.md` ordered by mtime and pick the newest. If none exist, tell the user there's no PRD to review — they should run `/prd:create <description>` first — and stop.
3. Read the PRD. Verify it has frontmatter and at least one user story matching `### US-NNN:` headings.
4. Count stories by `Status`. If **zero** stories are `done`, there is nothing to review yet — tell the user to run `/prd:work` first, and stop.

Set:

- `PRD_PATH` — absolute path to the PRD
- `PROGRESS_PATH` — sibling `progress.md` (may not exist; treat its `## Codebase Patterns` section, if present, as the source of truth for conventions learned during implementation)
- `REVIEW_PATH` — sibling `review.md` (you will write this in Step 5)
- `SLUG`, `PROJECT_NAME` — from the PRD frontmatter

## Step 2 — Resolve the branch diff

The work was implemented as commits on a feature branch (each story is a `feat: US-NNN - ...` commit). The review target is everything this branch adds on top of the main branch.

1. Get the current branch: `git rev-parse --abbrev-ref HEAD`.
2. If it **is** `main` or `master`, stop — there's no feature branch to review. Tell the user to switch to the branch that holds the PRD work (`git switch <branch>`) and rerun.
3. Determine the repo's main branch: use `main` if it exists, else `master` (`git rev-parse --verify <name>`). Call it `MAIN`.
4. Compute the base: `BASE_REF = git merge-base MAIN HEAD`. Set `HEAD_REF = HEAD`.
5. Sanity-check the range has content: `git rev-list --count BASE_REF..HEAD_REF`. If it's `0`, the branch has no commits beyond `MAIN` — tell the user there's nothing to review and stop.
6. Capture, for the report and the agents:
   - `BRANCH` — current branch name
   - `BASE_REF` — the merge-base SHA (short form for display)
   - `COMMITS` — `git log --oneline BASE_REF..HEAD_REF` (so you know which stories landed and in what order)

Note: the PRD and progress files are intentionally **not** committed (`/prd:work` never stages `docs/prds/`), so they won't appear in the diff — the agents read them directly from disk.

Briefly tell the user what you're reviewing, e.g. `Reviewing branch prd/user-auth (6 commits) against main for docs/prds/user-auth/prd.md — US-001…US-006 marked done.`

## Step 3 — Fan out parallel review subagents

Spawn **four** subagents with the **Agent** tool, `subagent_type: "general-purpose"`, **in a single message so they run concurrently**. Each gets the same context block; only the `{{FOCUS}}` differs. Substitute `{{PRD_PATH}}`, `{{PROGRESS_PATH}}`, `{{BASE_REF}}`, `{{HEAD_REF}}`, `{{BRANCH}}`, and `{{FOCUS}}` — leave everything else exactly as written.

Pass the diff by **reference**, not by value: give each agent the refs and let it run its own `git` commands and read full files. That keeps prompts small and lets each agent pull as much surrounding context as it needs.

```
You are one of several parallel reviewers checking a feature branch against the Product Requirements Document it was built from. You have isolated context so you can focus exclusively on your assigned concern.

## Inputs
- PRD: {{PRD_PATH}}
- Progress log: {{PROGRESS_PATH}} (may not exist)
- Branch under review: {{BRANCH}}
- Diff range: {{BASE_REF}}..{{HEAD_REF}}

## Orient yourself first
1. Read the PRD in full — Goals, every `### US-NNN` story (note which are `Status: done`), Functional Requirements, Non-Goals, Technical/Design Considerations, and Open Questions.
2. Read the progress log if it exists. Treat its `## Codebase Patterns` section as the project's established conventions.
3. Inspect the change: `git diff {{BASE_REF}}..{{HEAD_REF}}` for the full diff, `git log --stat {{BASE_REF}}..{{HEAD_REF}}` for the shape, and `git show <sha>` for individual story commits. Read whole files (not just hunks) and neighbouring existing files when you need context to judge a change fairly.

## Your focus
{{FOCUS}}

## Output — return ONLY a findings list in this exact format, nothing else
For each finding:

- **[severity]** <one-line title>
  - Story: <US-NNN it relates to, or "cross-cutting">
  - Evidence: <path:line, or commit sha, or "missing — expected in X">
  - Why: <the concrete problem, in one or two sentences>
  - Fix: <the smallest change that resolves it>
  - Confidence: <0-100, your honest read of whether this is real and not a false positive>

`severity` is one of: `blocking` (must fix before merge — breaks a stated acceptance criterion, a real bug, or a security hole), `recommended` (should fix — convention break, missing test, fragility), `nitpick` (minor/optional).

Rules:
- Only flag things you can point at with evidence. No vague "consider improving X".
- Judge against THIS project's conventions and THIS PRD's criteria — not generic ideals. If the project doesn't test a certain layer, don't demand tests for it.
- Don't flag things a linter/formatter/typechecker would catch — assume CI runs those.
- Don't flag pre-existing issues on lines this branch didn't touch.
- If your focus area is clean, return exactly: `No findings.`
```

The four `{{FOCUS}}` values:

**Agent 1 — Completion (acceptance criteria):**
```
Verify the work is actually done. For every story marked `Status: done` in the PRD, check each acceptance-criteria checkbox against the code on this branch: is it genuinely satisfied? Flag any criterion that is unmet, partially met, or faked. Also confirm the Functional Requirements (FR-N) the done stories cover are honoured, and check the diff didn't violate any Non-Goal. Map every finding to its US-NNN. If a story is marked done but you cannot find the code that implements it, that's a `blocking` finding.
```

**Agent 2 — Conventions & structure:**
```
Check that the new code fits the existing codebase. Compare against neighbouring files and the progress log's `## Codebase Patterns`: do new classes/modules/files follow the same naming and directory structure already in use? Are the same abstractions, error-handling patterns, and idioms reused rather than reinvented? Flag new bespoke patterns where an established one exists, misplaced files, and naming that breaks the local convention.
```

**Agent 3 — Tests:**
```
Check test coverage *relative to what this project already tests*. First learn the project's testing style — find the test directory/framework and look at how comparable existing features are tested (unit vs feature/integration). Then check whether the branch adds tests of that same kind for the behaviour it introduces, and whether they actually exercise the acceptance criteria. Flag untested new behaviour that the project's own conventions would normally cover, and tests that assert nothing meaningful. Do NOT demand a level of testing the project doesn't otherwise practise.
```

**Agent 4 — Security & unknowns:**
```
Look for security issues introduced by this branch: injection (SQL/command/template), missing authn/authz checks, unvalidated input, secrets or credentials committed to the repo, unsafe deserialization, SSRF, path traversal, missing output encoding, weak crypto, leaked internal details in errors, and dependency risks (new packages — do they exist and are they reputable?). Separately, surface UNKNOWNS: list any `Open Questions` from the PRD that remain unresolved by the code, plus any new ambiguities or risky assumptions the implementation baked in. Report unknowns as `recommended` findings titled "Open question: ...".
```

## Step 4 — Aggregate and filter

When all four agents return:

1. Collect every finding.
2. **Drop false positives**: discard anything with `Confidence < 50`. For `security` findings, keep `Confidence ≥ 40` but tag them `(needs verification)` so the user knows to confirm.
3. **De-duplicate**: if two agents flag the same thing, keep one and note both lenses.
4. **Group by severity**: `blocking` → `recommended` → `nitpick`.
5. **Build the story-completion table**: for each `done` story, mark ✅ (criteria met) or ⚠️ (something unmet — link the relevant finding). Note any stories still `todo` as ⏳ (not reviewed for completion).

Decide an overall **verdict**:
- ✅ **Ready to merge** — no `blocking` findings.
- ⚠️ **Needs work** — one or more `blocking` findings. State the count.

## Step 5 — Write the review report

Write `REVIEW_PATH` (`docs/prds/<slug>/review.md`). If it already exists, prepend a new dated section above the old one (most recent first) rather than overwriting — reviews accumulate like the progress log. Use this structure:

```markdown
# PRD Review — <Project Name> (<slug>)

## <YYYY-MM-DD HH:MM> — <verdict line>

**Branch:** <branch> · **Range:** <base-short-sha>..HEAD (<n> commits)
**Stories reviewed:** <e.g. US-001…US-006 done; US-007 todo>

### Story completion
- US-001 <title> — ✅
- US-002 <title> — ⚠️ <which criterion is unmet>
- US-007 <title> — ⏳ not started

### 🔴 Blocking
1. **<title>** (US-NNN) — <why>. Fix: <fix>. _Evidence: path:line_

### 🟡 Recommended
1. ...

### 🔵 Nitpicks
1. ...

### ❓ Open questions
- <unresolved PRD open questions + new ones surfaced>

### ✅ Pre-merge checklist
- [ ] <derived from blocking/recommended findings + project basics: full test suite green, etc.>

### 🚀 Post-merge / deploy
- [ ] <migrations to run, env vars/secrets to set, feature flags, deploy ordering, doc/changelog updates — only the ones that actually apply>
```

Generate the pre-merge and post-merge sections yourself from the findings plus what you can see of the project (migrations, env files, deploy config, changelog). Only list steps that genuinely apply — don't pad with generic advice.

## Step 6 — Propose PRD changes and confirm

Translate the actionable findings into a concrete remediation plan, using **per-issue judgment**:

- **Acceptance criteria unmet on a `done` story** → reopen it: flip `**Status:** done` → `**Status:** todo`, and append the specific gap(s) as new acceptance-criteria checkboxes under that story (e.g. `- [ ] <the unmet behaviour>`) so the next `/prd:work` knows exactly what to finish.
- **Genuinely new scope** (security gap, missing tests, an open question that needs building, anything not in the original criteria) → append a **new** story `US-NNN` (next sequential ID) with `**Status:** todo`, a sensible `**Priority:**` (after the existing ones unless it's urgent), a Description, and verifiable acceptance criteria derived from the finding.
- **Nitpicks / notes** → leave in `review.md` only; don't clutter the PRD.

Present the plan to the user as a short list, e.g.:

```
Proposed PRD changes:
  • Reopen US-002 (done→todo): file-upload size limit from its AC was never enforced
  • Add US-008 (todo): add feature tests for the export endpoint (matches existing tests/Feature style)
  • Add US-009 (todo): sanitize the filename param before path join (security: path traversal)
```

**Do not edit the PRD until the user confirms.** Let them adjust, drop, or reprioritize items. On confirmation, edit `PRD_PATH` to apply exactly the agreed changes — preserve all other content, keep IDs stable, and don't touch stories that passed review. If the user declines, leave the PRD untouched.

## Step 7 — Relay the result

Summarize for the user in a few lines:

- The verdict and the count of blocking / recommended findings.
- Where the full report lives (`docs/prds/<slug>/review.md`).
- What changed in the PRD, if anything.
- Clear next step: if stories were reopened or added, tell them to run `/prd:work` to address them, then `/prd:review` again to re-verify. If the verdict was clean, say it's ready to merge and point at the pre-merge checklist.

Don't start `/prd:work` yourself — the user stays in the loop between review and remediation.
