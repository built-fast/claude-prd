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

## Step 2 — Spawn a subagent with the work prompt

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
4. Run this project's quality checks before committing. Detect what the project uses (e.g. `package.json` scripts, `Makefile`, `composer.json`, `pyproject.toml`, `go test`, etc.) and run the relevant subset: tests, typecheck, lint. If you genuinely can't determine the right command, document that in your final report rather than skipping the check.
5. Only proceed to commit if quality checks pass. If they fail, fix the underlying issue — don't bypass the check, don't `--no-verify`, and don't commit broken code. If you can't make them pass, stop and explain why in your final report (leave the story as `todo`, don't update progress as if it succeeded).
6. Commit with message: `feat: {{STORY_ID}} - <Story Title from PRD>`. Stage only the files relevant to this story.
7. Update the PRD: change this story's `**Status:** todo` line to `**Status:** done`. Leave all other content untouched.
8. Append an entry to the progress log (create the file if it doesn't exist):

   ```
   ## <YYYY-MM-DD HH:MM> — {{STORY_ID}}: <Story Title>

   **What changed**
   - <Concise bullet list of what was implemented>

   **Files touched**
   - path/to/file.ext
   - path/to/other.ext

   **Quality checks**
   - <Command run>: <result>

   **Learnings**
   - <Patterns discovered, gotchas, useful context for future stories>

   ---
   ```

9. If a learning is a **general, reusable convention** that future stories will need (not story-specific trivia), also add it as a bullet under a `## Codebase Patterns` section at the very top of the progress file. Create that section if it doesn't exist. Keep patterns terse — one line each, e.g. `- Migrations go in db/migrations/; always run \`npm run db:migrate\` after creating.`

## Quality bar

- One story per run. Don't sneak in unrelated cleanup or other stories.
- Don't bypass failing checks. Fix the root cause or report and stop.
- Follow existing conventions; don't introduce new abstractions speculatively.
- Keep the commit focused — `git status` should be clean apart from your story's changes.

## Final report

Return a single concise message to the parent session (no need to dump full diffs — the commit and the progress file carry the detail). Cover:

- Story ID and title
- Commit SHA
- Quality checks that ran and their results
- Anything notable the user should know (a decision you made, an unexpected detail, a follow-up suggestion)

If you had to stop without completing the story (failing tests you couldn't fix, ambiguous requirements, missing dependencies), say so clearly and leave `Status: todo` so the next `/prd:work` can pick it up — but **do** still record an entry in the progress log explaining what blocked you, so the next attempt has that context.
```

## Step 3 — Relay the result

When the subagent finishes, summarize its report for the user in 3–6 lines. Then:

- If the subagent reported success and there are still `Status: todo` stories in the PRD, tell the user they can run `/prd:work` again to continue.
- If all stories are now `done`, congratulate and point at the progress log.
- If the subagent reported it had to stop, surface the blocker clearly and suggest next steps (e.g., "the failing test was X — want to investigate before retrying?").

Do not start a fresh subagent automatically. One story per invocation, by design — the user stays in the loop.
