---
description: Implement one story from a PRD in an isolated subagent
argument-hint: "[<prd-slug>] [<story-id>]"
---

# PRD Worker

You are the **dispatcher** for one cycle of PRD work. Your job is to pick the right PRD and story, then hand the actual implementation off to a fresh subagent (so each story gets a clean context window). You do **not** implement the story yourself.

Args passed to this command: $ARGUMENTS

## Step 1 — Resolve the PRD and story

Parse `$ARGUMENTS`. The accepted forms are:

- *(empty)* — use the most recently modified `docs/prds/*/prd.md` in this repo.
- `<slug>` — use `docs/prds/<slug>/prd.md`.
- `<slug> <story-id>` — use that PRD and that specific story (e.g. `user-auth US-003`).

Resolution:

1. If a slug is given, set `PRD_PATH = docs/prds/<slug>/prd.md`. Error and stop if it doesn't exist.
2. If no slug, list `docs/prds/*/prd.md` ordered by mtime and pick the newest. If none exist, tell the user there's no PRD yet — they should run `/prd:create <description>` first — and stop.
3. Read the PRD. Verify it has frontmatter and at least one user story matching `### US-NNN:` headings.
4. Determine the story to work:
   - If a story ID was given (e.g. `US-003`), use that. Error if it isn't in the PRD or its `Status` is already `done`.
   - Otherwise, scan stories and pick the one with the **lowest `Priority`** whose `Status: todo`. Ties broken by ID order.
5. If **no** stories have `Status: todo`, report 🎉 all stories complete, list the PRD path, and stop. Do not spawn a subagent.

Set:

- `PRD_PATH` — absolute path to the PRD
- `PROGRESS_PATH` — sibling `progress.md` next to the PRD (may not exist yet; the subagent creates it on first run)
- `STORY_ID` — the resolved `US-NNN`

Briefly tell the user which PRD and story you're dispatching, e.g. `Dispatching subagent for US-003 (Add priority filter dropdown) from docs/prds/user-auth/prd.md`.

## Step 2 — Ensure a working branch

The subagent commits during the run, so it must not be on the repo's main branch.

1. Get the current branch: `git rev-parse --abbrev-ref HEAD`.
2. If it is **not** `main` or `master`, continue to Step 3 — the user already has a working branch.
3. If it **is** `main` or `master`, do **not** dispatch yet. Propose a feature branch name derived from the PRD slug and story, e.g. `prd/<slug>-<story-id-lowercased>` (like `prd/user-auth-us-003`). Ask the user to confirm the name (they may supply their own).
4. On confirmation, create and switch to it with `git switch -c <branch>`. If the branch already exists, switch to it with `git switch <branch>` instead. Then continue to Step 3.
5. If the user declines to create a branch, stop without dispatching — don't commit work onto `main`.

## Step 3 — Spawn a subagent with the work prompt

Use the **Agent** tool with `subagent_type: "general-purpose"` and pass the block below as the `prompt`. Substitute `{{PRD_PATH}}`, `{{PROGRESS_PATH}}`, and `{{STORY_ID}}` with the values from Step 1 — leave everything else exactly as written.

The subagent has its own context window and full tool access. It will read the PRD, implement the story, run tests, commit, update the PRD status, and append to `progress.md`. When it returns, you'll get a summary.

```
You are implementing a single story from a Product Requirements Document. The parent session has dispatched you with isolated context so you can focus exclusively on this one story.

## Inputs

- PRD: {{PRD_PATH}}
- Progress log: {{PROGRESS_PATH}} (may not exist yet — create on first append)
- Story to implement: {{STORY_ID}}

## Your task

1. Read the PRD. Read the progress log if it exists — pay particular attention to any `## Codebase Patterns` section at the top, which captures conventions learned in earlier iterations.
2. Locate the {{STORY_ID}} story in the PRD. Read its Description, Acceptance Criteria, and the surrounding Functional Requirements / Technical Considerations sections for context.
3. Implement the story. Read whatever existing files you need to understand the codebase first — follow the conventions that are already there rather than inventing new ones.
4. Run the project's **full quality gate** before committing — the whole suite, not just the tests near your change. Every other story runs in its own isolated session and can't see your work; running the complete suite on every story is what stops one story silently breaking another and only finding out in CI.
   - Read the PRD's `## Quality Gate` section and run **every** command listed there, verbatim, against the whole project. You may run targeted tests while iterating, but the gate immediately before committing is the *full* suite passing.
   - If the PRD has **no** `## Quality Gate` section (an older PRD), detect the full-suite commands yourself — `package.json` scripts, `composer.json`, `Makefile`, `pyproject.toml`, `go test ./...`, etc., and mirror the repo's CI config if there is one — run them, and **write the section back into the PRD** so later stories and the review reuse the exact same commands.
   - If a gate command genuinely can't be determined, document that in your final report rather than silently skipping it.
5. Only proceed to commit if the **full** gate is green. If it fails, fix the underlying issue — don't bypass the check, don't `--no-verify`, don't commit broken code, and don't narrow the run to dodge a failure. If the suite is *already* red before you start (a previous story left a regression), fix it if it's small and in scope; otherwise stop and report it as a blocker so it's dealt with deliberately. If you can't get the gate green, stop and explain why in your final report (leave the story as `todo`, don't update progress as if it succeeded).
6. Commit the implementation, matching this repo's existing commit style:
   - First run `git log --oneline -30` to learn the repo's scope vocabulary and casing. Subjects follow a `<Scope>: <imperative description>` convention where the scope is a domain/module/tooling area (e.g. `API`, `Frontend`, `Models`, `claude`), **not** a conventional-commits type. Match the exact casing already used for the area you're touching. If no prior commit covers it, fall back to: acronyms ALLCAPS (`API`, `CI`, `DNS`), product/module names CamelCase (`Frontend`, `Models`, `Billing`), tooling/meta scopes lowercase (`claude`, `tests`, `docs`), sub-areas joined with `/` (`API/Webhooks`).
   - Write the subject in imperative mood (`add`, `fix`, `wire up`), ≤72 chars, no trailing period. **Do not** use `feat:`/`fix:`/`chore:` or any conventional-commits prefix, **do not** put the story ID ({{STORY_ID}}) in the message, and **never** add `Co-Authored-By: Claude`, "Generated with Claude Code", or any emoji/AI footer.
   - Add a body only when the *why* isn't already visible in the diff — plain prose wrapped at ~72 chars, no "Summary:"/"Changes:"/"Testing:" headers, no bullet list of files touched. Most commits need no body.
   - Use a heredoc (`git commit -m "$(cat <<'EOF' … EOF\n)"`) so any body newlines are preserved. Never pass `--no-verify` and never `--amend`; if a commit hook fails, fix the underlying issue and make a **new** commit.
   - Stage **only** the implementation files for this story, by explicit path (`git add path/to/file ...`). Never use `git add -A`, `git add .`, or `git commit -a`. **Do not stage or commit any file under `docs/prds/`** — the PRD and progress log are local tracking artifacts. If the user wants them in version control, that's a manual decision they make themselves.
7. Update the PRD: change this story's `**Status:** todo` line to `**Status:** done`. Leave all other content untouched.
8. Append an entry to the progress log (create the file if it doesn't exist):

   ```
   ## <YYYY-MM-DD HH:MM> — {{STORY_ID}}: <Story Title>

   **What changed**
   - <Concise bullet list of what was implemented>

   **Files touched**
   - path/to/file.ext
   - path/to/other.ext

   **Quality gate** (full suite — every gate command, with its result)
   - <gate command 1>: <pass/fail>
   - <gate command 2>: <pass/fail>

   **Learnings**
   - <Patterns discovered, gotchas, useful context for future stories>

   ---
   ```

9. If a learning is a **general, reusable convention** that future stories will need (not story-specific trivia), also add it as a bullet under a `## Codebase Patterns` section at the very top of the progress file. Create that section if it doesn't exist. Keep patterns terse — one line each, e.g. `- Migrations go in db/migrations/; always run \`npm run db:migrate\` after creating.`

## Quality bar

- One story per run. Don't sneak in unrelated cleanup or other stories.
- Don't bypass failing checks. Fix the root cause or report and stop.
- Follow existing conventions; don't introduce new abstractions speculatively.
- Keep the commit focused. After committing, the only uncommitted changes left in `git status` should be the PRD edits (the status flip, plus the `## Quality Gate` section if you had to back-fill it) and the progress log — leave those uncommitted for the user to handle manually.

## Final report

Return a single concise message to the parent session (no need to dump full diffs — the commit and the progress file carry the detail). Cover:

- Story ID and title
- Commit SHA
- The full quality-gate commands that ran and their pass/fail results
- Anything notable the user should know (a decision you made, an unexpected detail, a follow-up suggestion)

If you had to stop without completing the story (failing tests you couldn't fix, ambiguous requirements, missing dependencies), say so clearly and leave `Status: todo` so the next `/prd:work` can pick it up — but **do** still record an entry in the progress log explaining what blocked you, so the next attempt has that context.
```

## Step 4 — Relay the result

When the subagent finishes, summarize its report for the user in 3–6 lines. Then:

- If the subagent reported success and there are still `Status: todo` stories in the PRD, tell the user they can run `/prd:work` again to continue.
- If all stories are now `done`, congratulate and point at the progress log.
- If the subagent reported it had to stop, surface the blocker clearly and suggest next steps (e.g., "the failing test was X — want to investigate before retrying?").

Do not start a fresh subagent automatically. One story per invocation, by design — the user stays in the loop.
