# PRD Plugin for Claude Code

Create and work through Product Requirements Documents story-by-story. The plugin provides two slash commands that pair together: one to author a PRD via guided Q&A, and one to implement it incrementally with a fresh subagent per story.

## Install

```
/plugin marketplace add built-fast/claude-prd
/plugin install prd@claude-prd
```

After installing, `/prd:create` and `/prd:work` are available in any project.

## Commands

### `/prd:create <brief feature description>`

Guides you through writing a PRD via lettered clarifying questions, then writes a structured markdown file to `docs/prds/<slug>/prd.md`. Produces an ordered list of user stories with stable IDs (`US-001`, `US-002`, ...), verifiable acceptance criteria, and a `Status: todo` flag on each — designed to be consumed by `/prd:work`.

Does not write any implementation code.

### `/prd:work [<prd-slug>] [<story-id>]`

Dispatches a fresh subagent to implement **one** story from a PRD. The subagent:

1. Reads the PRD and any existing `progress.md`.
2. Implements the lowest-priority `todo` story (or the one you specified).
3. Runs the project's quality checks (tests, typecheck, lint).
4. Commits with `feat: US-NNN - <Story Title>`.
5. Flips the story's status to `done` in the PRD.
6. Appends an entry to `docs/prds/<slug>/progress.md`, including any reusable codebase patterns it discovered.

Argument forms:

- `/prd:work` — picks the most recently modified PRD and its next `todo` story.
- `/prd:work <slug>` — uses that PRD, picks the next `todo` story.
- `/prd:work <slug> <story-id>` — uses that PRD and that specific story.

One story per invocation, by design — you stay in the loop between stories.

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
```

## Why two commands

Story-by-story execution keeps each implementation session focused with a clean context window, and gives you a natural review checkpoint between stories. The PRD acts as the durable contract; `progress.md` captures cross-story learnings so later stories inherit the conventions discovered in earlier ones.

## Requirements

- Claude Code with plugin support
- A git repo (the worker commits each story)

## License

MIT - see [LICENSE](LICENSE) for details.
