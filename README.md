# PRD Plugin for Claude Code

Create and work through Product Requirements Documents story-by-story. The plugin provides three slash commands that pair together: one to author a PRD via guided Q&A, one to implement it incrementally with a fresh subagent per story, and one to review the resulting branch against the PRD.

## Install

```
/plugin marketplace add built-fast/claude-prd
/plugin install prd@claude-prd
```

After installing, `/prd:create`, `/prd:work`, and `/prd:review` are available in any project.

## Commands

### `/prd:create <brief feature description>`

Guides you through writing a PRD via lettered clarifying questions, then writes a structured markdown file to `docs/prds/<slug>/prd.md`. Produces an ordered list of user stories with stable IDs (`US-001`, `US-002`, ...), verifiable acceptance criteria, and a `Status: todo` flag on each — designed to be consumed by `/prd:work`. It also detects the project's full-suite commands and records them in a `## Quality Gate` section, which becomes the single source of truth that `/prd:work` and `/prd:review` both run.

Does not write any implementation code.

### `/prd:work [<prd-slug>] [<story-id>]`

Dispatches a fresh subagent to implement **one** story from a PRD. The subagent:

1. Reads the PRD and any existing `progress.md`.
2. Implements the lowest-priority `todo` story (or the one you specified).
3. Runs the **full** quality gate (whole test suite, typecheck, lint) — not just the tests near the change — so a story can't silently break another and only fail in CI. If the PRD predates the gate, it detects the commands and back-fills the section.
4. Commits in the repo's own style — a `<Scope>: <imperative description>` subject learned from `git log`, no conventional-commits prefix and no story code (the story is tracked via the PRD status and progress log instead).
5. Flips the story's status to `done` in the PRD.
6. Appends an entry to `docs/prds/<slug>/progress.md`, including any reusable codebase patterns it discovered.

Argument forms:

- `/prd:work` — picks the most recently modified PRD and its next `todo` story.
- `/prd:work <slug>` — uses that PRD, picks the next `todo` story.
- `/prd:work <slug> <story-id>` — uses that PRD and that specific story.

One story per invocation, by design — you stay in the loop between stories.

### `/prd:review [<prd-slug>]`

Reviews the current feature branch against the PRD. The dispatcher resolves the PRD and the branch diff (everything the branch adds on top of `main`/`master`), then **runs the quality gate on the branch HEAD** — the one point in the workflow where the review actually executes the code. A red suite is an automatic blocking finding that forces a "Needs work" verdict, so a regression is caught here rather than in CI. It then fans out **four parallel subagents** (which read, not run) that each check one concern:

1. **Completion** — are the `done` stories' acceptance criteria actually met by the code?
2. **Conventions** — does the work follow the project's existing structure, naming, and patterns?
3. **Tests** — does it carry the kind of tests this project already writes for comparable behavior?
4. **Security & unknowns** — any vulnerabilities, committed secrets, or unresolved open questions?

Findings are aggregated, false-positive-filtered by confidence, and grouped by severity (blocking / recommended / nitpick). The command writes a dated report to `docs/prds/<slug>/review.md` (with a story-completion table, pre-merge checklist, and post-merge/deploy steps), then proposes concrete PRD changes for your confirmation:

- A `done` story whose criteria aren't met is **reopened** (`done` → `todo`) with the specific gap added as new acceptance criteria.
- Genuinely new scope (security fixes, missing tests, open questions to build) becomes **new `US-NNN` stories**.

Nothing is written to the PRD until you confirm. After applying changes, run `/prd:work` to address them, then `/prd:review` again to re-verify.

Argument forms:

- `/prd:review` — reviews the most recently modified PRD.
- `/prd:review <slug>` — reviews that specific PRD.

## Typical flow

```
/prd:create add a CSV export to the reports page
# answer the clarifying questions
# review docs/prds/csv-export/prd.md, edit if needed

/prd:work
# subagent implements US-001, commits, updates status

/prd:work
# subagent implements US-002, builds on US-001's work
# ... repeat until all stories are done

/prd:review
# runs the full quality gate, then parallel subagents check
# completion, conventions, tests, security
# writes review.md, proposes fixes as reopened/new stories

/prd:work    # address any remediation stories
/prd:review  # re-verify, then merge
```

## Why these commands

Story-by-story execution keeps each implementation session focused with a clean context window, and gives you a natural review checkpoint between stories. The PRD acts as the durable contract; `progress.md` captures cross-story learnings so later stories inherit the conventions discovered in earlier ones. `/prd:review` closes the loop, checking the finished branch back against that contract and feeding any gaps back into the same story-driven workflow.

## Requirements

- Claude Code with plugin support
- A git repo (the worker commits each story)

## License

MIT - see [LICENSE](LICENSE) for details.
