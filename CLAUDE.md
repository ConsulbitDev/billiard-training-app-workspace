# CLAUDE.md — Workspace Root

This is the unified multi-repo workspace for the Billiard Training App. Each subfolder
below is an **independent git repository** with its own remote and, where applicable,
its own agent doc — this file is only a map to orient a session opened at the workspace
root.

## Cross-cutting context (read these first for anything spanning multiple projects)
- `PRD.md` — product requirements
- `MICROSERVICE_ARCHITECTURE.md` — system architecture across services
- `API_CONTRACT.md` — FE/BE API contract (enum mappings, known issues, implementation status)
- `PROJECT_WORKFLOW.md` — canonical requirements → delivery workflow (Epics/Stories/Tasks, ADRs, GitHub Project board mechanics, cadence)
- `README.md` — short workspace overview and pointer to the docs above
- `docs/ADR/` — architecture decision records
- `docs/templates/` — issue/ADR templates

## Sibling projects

| Folder | Purpose | Own agent doc | Remote |
|---|---|---|---|
| `billiard-training-app-be/` | Spring Boot 3 backend (Java 17, JPA, Flyway) | `billiard-training-app-be/CLAUDE.md` | billiard-training-be |
| `billiard-training-app-fe/` | Angular 19 frontend | `billiard-training-app-fe/CLAUDE.md` | billiard-training-fe |
| `billiard-analyzer/` | Java/Maven JavaCV video-processing engine | `billiard-analyzer/CLAUDE.md` | billiard-analyzer-engine |
| `billiard-analyzer-python/` | Python video extractor (table-shape recognition) | none yet | billiard-analyzer |
| `.github/` | Org-wide community health files (issue/PR templates, workflows) | none (not code) | .github |

## Working across repos

- All sibling folders are flat, directly under this parent — relative paths like
  `../billiard-training-app-fe/` from inside `billiard-training-app-be/` resolve correctly.
- Each sibling has its own git history, remote, and CI — there is no monorepo squash and
  no submodules. Commits/pushes in one sibling are independent of the others.
- If you add a 6th sibling project, update **both** `.idea/modules.xml` and
  `.idea/jb-workspace.xml` in this folder — they are not auto-synced with each other.
- Exception: `billiard-analyzer-python` is intentionally listed only in `modules.xml`
  (as a classic module), not in `jb-workspace.xml`. It has its own self-contained `.idea/`
  project config, and adding it to `jb-workspace.xml` as well triggers an IntelliJ error
  ("Project 'billiard-analyzer-python' cannot be added to workspace") — don't re-add it there.

## Spawning agents that work in a sibling repo

Claude Code's `isolation: "worktree"` only isolates within the **current** repo (this workspace
root). It cannot create a worktree inside a sibling repo like `billiard-training-app-be` or
`billiard-training-app-fe` — an agent pinned to a workspace-root worktree gets its `Edit`/`Write`/
`git` tools hard-blocked the moment it targets a sibling repo's real path, with no way to opt out.

If you need real isolation between two agents both working in the *same* sibling repo, you must
create the worktree manually inside that sibling repo's own `.git` (plain `git worktree add`),
not via this workspace's isolation mechanism. Otherwise, run agents targeting the same sibling
repo **sequentially**, not in parallel — two non-isolated agents editing/committing in the same
real checkout at once risk clobbering each other if their file sets ever overlap. Agents targeting
*different* sibling repos (e.g. one in `-be`, one in `-fe`) are naturally isolated already, since
they're different git repos on disk — no special handling needed there.
