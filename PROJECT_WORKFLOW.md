# Project Workflow

This is the single source of truth for how requirements become work items, how work items are
tracked, and how they get delivered for the Billiard Training App.

**Guiding principle:** ship small slices of value fast (sign up → create plan → log shot → see one stat).

For how to fill out each issue template and the full PR checklist, see [`.github/WORKFLOW.md`](./.github/WORKFLOW.md)
(org-wide reference, shared across all ConsulbitDev repos) and [`.github/pull_request_template.md`](./.github/pull_request_template.md).

---

## 1. Where Requirements Come From

- **Entry point**: all ideas/questions start in HQ chat.
- **Format**: capture as **Job Stories** — *"When … I want to … so I can …"*
- **Owner**: Product Owner + team discussion.

---

## 2. Work Item Types

Use these issue types:
- `Epic`: large outcome spanning multiple stories
- `Story`: user-facing slice of value
- `Task`: implementation task
- `Bug`: broken behavior
- `Spike`: time-boxed research
- `Chore`: maintenance or admin work

Rule of thumb:
- Plan mostly with `Epic` and `Story`
- Use `Task` only when a story needs explicit technical breakdown
- Use `Spike` when the goal is learning or deciding, not shipping

Each type (except `Chore`) has a dedicated issue form template under `.github/ISSUE_TEMPLATE/` —
see `.github/WORKFLOW.md` for the exact fields each template captures.

## Recommended Hierarchy

```
Epic (large user outcome)
├── Story 1 (user-facing value)
│   └── Task A (implementation detail)
├── Story 2
└── Spike (research, if uncertain)
```

Example:
- Epic: `Shots & Systems CRUD`
- Story: `List shots/systems with basic filters`
- Task: `Add repository query for category and priority filters`

---

## 3. Architecture Decision Records (ADR)

- **Purpose**: log key technical/structural choices.
- **Format**: Context → Decision → Consequences → Alternatives.
- **Location**: markdown file under `/docs/adr/`, referenced from the relevant issue.
- Templates live in `docs/templates/ADR.md`; a running log is kept in `docs/github/DECISIONS.md`.

---

## 4. The GitHub Project Board

Project:
- `ConsulbitDev / Billiard Training App`
- URL: `https://github.com/orgs/ConsulbitDev/projects/1`

Repos currently connected to the workflow:
- `ConsulbitDev/billiard-training-fe`
- `ConsulbitDev/billiard-training-be`

Use the GitHub Project as the single planning and tracking board across frontend and backend work.
Create issues in the repo where the work belongs. Track prioritization and delivery in the project board.

### Basic Workflow

1. Create an issue in the relevant repo using the issue form.
2. The issue is automatically added to the GitHub Project.
3. Automation sets `Work Type` from the issue label.
4. In the project, fill or update the planning fields.
5. Move the item through the board until done.
6. Link pull requests back to the issue.

### Project Fields

**Work Type** — set automatically from labels:
`type:epic` → Epic · `type:story` → Story · `type:task` → Task · `type:bug` → Bug · `type:spike` → Spike · `type:chore` → Chore

**Area** — where the work belongs: `Frontend` · `Backend` · `Infra` · `Docs` · `Analytics` · `Security`

**Priority** — urgency: `P0` critical · `P1` important MVP/near-term · `P2` useful, not urgent · `P3` nice to have

**Size** — rough effort: `XS` · `S` · `M` · `L` · `XL` (likely too big for one story)

**Status** — main board flow: `Backlog` → `Ready` → `In progress` → `In review` → `Blocked` → `Done`

**Iteration** — only if planning work by week; leave empty otherwise.

### Existing Automation

Frontend and backend repos already contain GitHub Actions that:
- add newly opened issues to project `ConsulbitDev / Project 1`
- set the `Work Type` project field from the issue label

This means the main manual work is keeping `Area`, `Priority`, `Size`, `Status`, and optional `Iteration` up to date.

---

## 5. Cadence

- **Daily**: 5-min async standup in HQ → *Shipped / Next / Blockers*.
- **Weekly**: review metrics, pick 3 outcomes, log decisions.

---

## 6. Principles

- Deliver value early, not perfection.
- Keep scope small (one Epic = one clear goal).
- Start simple (monolith first, evolve later).
- Privacy & security by default.
- Everything must be GitHub-ready (copy-pasteable issue/PR text).

---

## 7. Practical Advice

- Keep stories small enough to finish in a few days.
- Avoid creating too many `Task` issues unless they help execution.
- Use `Area` to split frontend and backend views.
- Prefer one shared project board for the whole app.
- Split oversized stories instead of carrying `XL` work for too long.

---

With this workflow, any contributor (or agent) can jump in and know: where requirements come
from, how work is shaped, and how progress is tracked.
