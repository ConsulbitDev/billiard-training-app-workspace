# ADR-003: Close out V1 before starting V2 (Training Sessions, Statistics, real Auth)

## 📌 Context

`PRD.md` scopes V1 as the Knowledge Base Explorer + Admin only, explicitly excluding Training
Sessions, Statistics, and Authentication. `MICROSERVICE_ARCHITECTURE.md` frames Training Session
Service and Statistics & Analytics Service as V2+ work, and shows V1 as having only
"Authentication — stubbed." Despite this, `billiard-training-be`#12 ("MVP Auth") is an open Epic
for *full* signup/login/JWT auth, and `billiard-training-app-fe/CLAUDE.md`'s "Project Focus (MVP)"
section lists Practice Sessions and Basic Stats as currently targeted — contradicting both other
documents. V1's core CRUD (Shots, Categories, Books, Resources, Comments) is now functionally
complete.

## 💡 Decision

The next milestone closes out three of the remaining V1 scope gaps — add-shot mode, delete-shot,
and YouTube resource-URL normalization — before any V2 work (real Authentication, Training
Sessions, Statistics) begins. Epic #12 (MVP Auth) and the "Project Focus (MVP)" list in
`billiard-training-app-fe/CLAUDE.md` are stale relative to the PRD and should not be treated as
current scope until V2 is deliberately kicked off.

Two items are explicitly deferred out of this milestone:
- **Archive/unarchive a shot** (the `status: active/archived` concept from the PRD's original
  data model) — a real V1 scope item, just not part of this close-out pass.
- **Postgres migration** (#58) — deferred alongside archive/unarchive, not part of this pass.

A related Flyway-introduction task is deliberately excluded from ticketing/automation entirely:
the user is doing that one personally in Mentor/Learner mode (see the Mentor/Learner
Collaboration Model in `billiard-training-app-be/CLAUDE.md`), not as an agent-driven ticket.

**Delete-shot is a soft-delete**, using a new, narrow field on `Shot` dedicated to this purpose
(hidden from all queries, no restore path this milestone) — deliberately *not* the same field as
the deferred archive/unarchive concept, so that future design isn't constrained now. See
`CONTEXT.md`'s Soft-deleted/Archived glossary entries.

## 🔄 Consequences

**Positive**:
- Removes the three-way scope contradiction between PRD, architecture doc, and open issues.
- Ships a fully complete, coherent V1 before taking on auth/sessions/stats, which all depend on
  having a real user concept that doesn't exist yet.

**Follow-up needed**:
- `billiard-training-app-fe/CLAUDE.md`'s "Project Focus (MVP)" section should be corrected to
  drop Practice Sessions/Basic Stats once V2 scope is actually decided, so it stops contradicting
  the PRD.
- Epic #12 should be re-scoped or re-labeled when V2 auth work actually starts, since "MVP Auth"
  implies V1 scope it doesn't have.
