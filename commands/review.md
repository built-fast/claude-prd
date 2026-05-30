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
- `REVIEW_PATH` — sibling `review.md` (you will write this in Step 7)
- `SLUG`, `PROJECT_NAME` — from the PRD frontmatter

## Step 2 — Resolve the branch diff

The work was implemented as commits on a feature branch. The review target is everything this branch adds on top of the main branch.

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

## Step 3 — Check the branch is current with the default branch

A stale branch is the quiet failure mode of a worktree workflow: another feature lands on the default branch while this one waits in review, and the conflict only surfaces when the PR won't merge. Catch it here — before the quality gate and the reviewers, so everything downstream runs against the state the branch will actually merge as.

1. **Fetch so "landed" means landed.** Find the remote (`git remote` — usually `origin`); if one exists, `git fetch <remote> MAIN` and set `REMOTE_MAIN = <remote>/MAIN`. With no remote, fall back to the local `MAIN`. Always compare against — and later rebase onto — the remote-tracking ref: that never touches the local `MAIN` branch, which may be checked out in another worktree.
2. **Measure staleness:** `BEHIND = git rev-list --count HEAD..REMOTE_MAIN`. If `0`, the branch already contains the whole default branch — note "current with `REMOTE_MAIN`" and skip to Step 4.
3. **Test for conflicts without touching the working tree:** `git merge-tree --write-tree REMOTE_MAIN HEAD`. Exit `0` = merges clean; non-zero = conflicts, and the output names the conflicting paths. This mirrors how the forge decides PR mergeability. (If the local Git predates `--write-tree` — older than 2.38 — treat "behind" as "may conflict" and let the rebase in the next step reveal it.)
4. **Report where the branch stands and offer to rebase** — e.g. `Branch is 4 commits behind origin/main — merges clean.` or `Branch is 4 commits behind origin/main and conflicts in src/auth.ts, src/router.ts.`
   - **Behind, clean** → offer `git rebase REMOTE_MAIN`. On accept, run it (it should apply without intervention), then **recompute the Step 2 diff range** (`BASE_REF`, `HEAD_REF=HEAD`, `COMMITS`) against the rewritten history before continuing — the SHAs changed.
   - **Behind, conflicts** → do not rebase into a conflict blind. Show the conflicting paths; offer to *start* the rebase so the user can resolve them now, or to proceed with the review as-is. If the rebase stops on a conflict, pause and let the user resolve it — never guess resolutions.
   - **Declined** → proceed, but carry it forward (it feeds Step 6).
   - If the branch was already pushed, say so when you offer: the rebase rewrites history, so they'll need `git push --force-with-lease` afterward.
5. **Capture `BRANCH_CURRENCY`** for the report and aggregation — one of: `current`, `behind N (clean)`, `behind N (conflicts: <paths>)`, or `rebased onto REMOTE_MAIN`.

If you rebased, the gate and the reviewers all see the integrated code — which is the whole point of doing this before Step 4.

## Step 4 — Run the quality gate (the execution backstop)

The parallel reviewers in the next step **read** code; they do not run it. This step is the one place the review actually executes the project — so a regression that static reading misses (a story that broke another story's tests, a failure that only appears when the whole suite runs together) is caught here instead of in CI. Do this yourself in the dispatcher, once, before fanning out.

1. **Find the gate commands.** Read the PRD's `## Quality Gate` section. If it's absent (older PRD), detect the full-suite commands from the repo — mirror the CI config (`.github/workflows/*`, etc.) if there is one — and note in the report that the PRD had no gate recorded.
2. **Test the branch as it stands.** You're on the feature branch at HEAD; the uncommitted PRD/progress edits don't affect the suite. Run from the repo root.
3. **Run every gate command, full-suite.** Capture each result (pass/fail) and, for any failure, the failing tests / key error output.
4. **Record the outcome — it headlines the report:**
   - **All green** → note it and proceed to fan-out.
   - **Any command red** → this is an automatic `blocking` finding and forces a **⚠️ Needs work** verdict, regardless of what the static reviewers conclude. Capture which command failed and the failing tests/errors. Still run the fan-out (its findings feed the same remediation pass), but the suite failure is the headline.

Do not skip this step or assume CI covers it — executing the gate here is the entire point.

## Step 5 — Fan out parallel review subagents

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
- Don't flag things the quality gate would catch — the dispatcher runs the full suite (tests, typecheck, lint) separately, so don't predict pass/fail or duplicate it. Focus on what execution can't reveal: logic errors, design fit, coverage gaps, security.
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
Check test *coverage* relative to what this project already tests — the suite is being run separately by the dispatcher, so don't predict whether it passes; focus on whether the right tests exist. First learn the project's testing style — find the test directory/framework and look at how comparable existing features are tested (unit vs feature/integration). Then check whether the branch adds tests of that same kind for the behaviour it introduces, and whether they actually exercise the acceptance criteria. Flag untested new behaviour that the project's own conventions would normally cover, and tests that assert nothing meaningful. Do NOT demand a level of testing the project doesn't otherwise practise.
```

**Agent 4 — Security & unknowns:**
```
Look for security issues introduced by this branch: injection (SQL/command/template), missing authn/authz checks, unvalidated input, secrets or credentials committed to the repo, unsafe deserialization, SSRF, path traversal, missing output encoding, weak crypto, leaked internal details in errors, and dependency risks (new packages — do they exist and are they reputable?). Separately, surface UNKNOWNS: list any `Open Questions` from the PRD that remain unresolved by the code, plus any new ambiguities or risky assumptions the implementation baked in. Report unknowns as `recommended` findings titled "Open question: ...".
```

## Step 6 — Aggregate and filter

When all four agents return:

1. Collect every finding.
2. **Fold in the quality-gate result from Step 4**: if the suite was red, add it as the **top** `blocking` finding (which command failed, which tests/errors) before anything the static agents surfaced.
3. **Fold in the branch-currency result from Step 3** (skip if you rebased, or if the branch was already current): a merge conflict the user declined to rebase away is a `blocking` finding (`branch N behind <default> — conflicts in <paths>`, ranking just under a red gate); a clean-but-behind branch the user declined to rebase is a `recommended` "rebase onto `<default>` before merge".
4. **Drop false positives**: discard anything with `Confidence < 50`. For `security` findings, keep `Confidence ≥ 40` but tag them `(needs verification)` so the user knows to confirm.
5. **De-duplicate**: if two agents flag the same thing, keep one and note both lenses.
6. **Group by severity**: `blocking` → `recommended` → `nitpick`.
7. **Build the story-completion table**: for each `done` story, mark ✅ (criteria met) or ⚠️ (something unmet — link the relevant finding). Note any stories still `todo` as ⏳ (not reviewed for completion).

Decide an overall **verdict**:
- ✅ **Ready to merge** — the quality gate was green, the branch merges cleanly into the default branch, **and** there are no `blocking` findings.
- ⚠️ **Needs work** — the quality gate was red, the branch has an unresolved merge conflict with the default branch, or there are one or more `blocking` findings. State the count (and lead with the gate failure if that's the cause).

## Step 7 — Write the review report

Write `REVIEW_PATH` (`docs/prds/<slug>/review.md`). If it already exists, prepend a new dated section above the old one (most recent first) rather than overwriting — reviews accumulate like the progress log. Use this structure:

```markdown
# PRD Review — <Project Name> (<slug>)

## <YYYY-MM-DD HH:MM> — <verdict line>

**Branch:** <branch> · **Range:** <base-short-sha>..HEAD (<n> commits)
**Stories reviewed:** <e.g. US-001…US-006 done; US-007 todo>
**Quality gate:** <✅ all green | 🔴 `<command>` failed — <n> tests / key error>
**Branch currency:** <✅ current with origin/main | ♻️ rebased onto origin/main | ⚠️ 4 behind origin/main, merges clean | 🔴 4 behind origin/main — conflicts in src/auth.ts>

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
- [ ] <derived from blocking/recommended findings. The quality gate was already executed in this review — mark it ✅ if green, or list "get the gate green (`<command>`)" as the first item if it was red. If the branch wasn't rebased and still trails the default branch, list "rebase onto `<default>` (resolve conflicts in `<paths>`)" — first if it conflicts.>

### 🚀 Post-merge / deploy
- [ ] <migrations to run, env vars/secrets to set, feature flags, deploy ordering, doc/changelog updates — only the ones that actually apply>
```

Generate the pre-merge and post-merge sections yourself from the findings plus what you can see of the project (migrations, env files, deploy config, changelog). Only list steps that genuinely apply — don't pad with generic advice.

## Step 8 — Propose PRD changes and confirm

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

## Step 9 — Relay the result

Summarize for the user in a few lines:

- The verdict and the count of blocking / recommended findings — lead with the quality-gate result (green, or which command failed).
- Where the full report lives (`docs/prds/<slug>/review.md`).
- What changed in the PRD, if anything.
- Clear next step: if stories were reopened or added, tell them to run `/prd:work` to address them, then `/prd:review` again to re-verify. If the verdict was clean, say it's ready to merge and point at the pre-merge checklist.

Don't start `/prd:work` yourself — the user stays in the loop between review and remediation.
