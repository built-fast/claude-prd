---
description: Create a Product Requirements Document via guided Q&A
argument-hint: brief feature description
---

# PRD Author

You are helping the user write a Product Requirements Document (PRD) for a new feature. The PRD will later be implemented one story at a time by `/prd:work`, which spawns a fresh subagent per story — so the stories you produce here become the contract for that downstream work.

Initial feature description: $ARGUMENTS

## Your job

Produce a single markdown file at `docs/prds/<slug>/prd.md` (relative to the current working directory). **Do not write any implementation code.** You are authoring requirements, not building the feature.

## Step 1 — Ask clarifying questions

Before drafting anything, ask 3–5 short clarifying questions to pin down the parts of the request that are ambiguous. Format them as lettered options so the user can answer with `1A, 2C, 3B` instead of writing paragraphs. Indent the options under each question.

```
1. What is the primary goal of this feature?
   A. Improve user onboarding
   B. Increase retention
   C. Reduce support burden
   D. Other: [please specify]

2. Who is the target user?
   A. New users only
   B. Existing users only
   C. All users
   D. Admin users only
```

Focus your questions on whichever of these are unclear from the initial description:

- **Problem / goal** — what problem does this solve, why does it matter?
- **Core behavior** — the key actions or flows
- **Scope boundaries** — what is explicitly out of scope?
- **Success criteria** — how do we know it works?
- **Constraints** — known technical or design constraints
- **Existing patterns** — anything in the codebase to mirror or reuse?

If the user's initial description is already very specific (e.g., they pasted a detailed spec), skip questions and confirm your understanding in 2–3 sentences instead.

After the user answers, briefly restate your understanding and then proceed to write the PRD.

## Step 2 — Choose the slug and create the directory

Derive a short kebab-case `<slug>` from the feature name (e.g., "user authentication" → `user-auth`, "Stripe webhook handler" → `stripe-webhooks`). Keep it under ~30 characters and avoid generic names like `feature` or `update`.

Create the directory `docs/prds/<slug>/` and write the PRD to `docs/prds/<slug>/prd.md`. If a PRD already exists at that path, ask the user whether to overwrite, pick a different slug, or stop.

## Step 3 — Write the PRD

Use this exact structure. The frontmatter is consumed by `/prd:work`, so keep those fields stable.

```markdown
---
project: <Human-readable project name>
slug: <kebab-case-slug>
created: <YYYY-MM-DD>
---

# PRD: <Project Name>

## Introduction

One short paragraph: what this feature is and what problem it solves.

## Goals

- Specific, measurable objective
- Another objective

## User Stories

### US-001: <Short descriptive title>
**Priority:** 1
**Status:** todo

**Description:** As a <user type>, I want <capability> so that <benefit>.

**Acceptance Criteria:**
- [ ] Verifiable criterion (not "works correctly")
- [ ] Another verifiable criterion
- [ ] Tests pass / typecheck passes (use whatever this project actually requires)

### US-002: <Title>
**Priority:** 2
**Status:** todo

...

## Functional Requirements

- FR-1: The system must <do X>.
- FR-2: When the user does Y, the system must <do Z>.

## Non-Goals (Out of Scope)

- Things this feature deliberately will not do. This section is critical for scope control.

## Design Considerations

UI/UX notes, mockup links, existing components to reuse. Omit the section if not relevant.

## Technical Considerations

Known constraints, dependencies, integration points, performance requirements. Omit the section if not relevant.

## Success Metrics

How we'll know it worked. Be concrete:
- "Users can do X in under 3 clicks"
- "Reduces Y time by 50%"

## Open Questions

Anything still unresolved that doesn't block starting work.
```

### Rules for the stories — these matter

The `/prd:work` subagent will pick the lowest-numbered `Priority` with `Status: todo` and implement it in one focused session. So:

- **Order by build sequence.** Foundations (data model, types, migrations) come before features that depend on them. Priority 1 should be implementable without any of the others existing.
- **One story = one coherent commit.** If a story can't realistically be done in a single focused session ending in a green test run, split it.
- **Acceptance criteria must be verifiable.** "Button shows confirmation dialog before deleting" is good. "Works correctly" is not. The subagent uses these as its done-check.
- **Bake quality into each story.** Include the project's actual quality bar (tests pass, typecheck passes, lint clean) as the last acceptance criterion. Don't invent a separate "add tests" story at the end — tests are part of each story.
- **Preserve `Status: todo` on every story.** `/prd:work` flips it to `done` as stories are completed.
- **Use stable IDs.** `US-001`, `US-002`, ... — these are referenced in commit messages and progress logs.

### Writing for the implementer

The reader is a fresh Claude subagent with no prior context on this feature beyond what's in the PRD and `progress.md`. So:

- Be explicit and unambiguous. Avoid jargon, or define it when you use it.
- Reference concrete files, components, or endpoints when you know them.
- Numbered functional requirements (FR-N) give the subagent precise hooks to verify against.

## Step 4 — Confirm and hand off

After writing the file, tell the user:

1. The path you wrote to (`docs/prds/<slug>/prd.md`)
2. How many stories you produced and the rough sequence
3. That they can review/edit the file directly, then run `/prd:work <slug>` to start implementation (or just `/prd:work` to pick up the most recent PRD)

Do not start implementation yourself. End your turn after handing off.
