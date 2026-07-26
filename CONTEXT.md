# Billiard Training App — Domain Glossary

Shared vocabulary across `billiard-training-app-be` and `billiard-training-app-fe`. See
`CLAUDE.md` for the workspace map. This file is a glossary only — no implementation details,
no roadmap, no scratch notes. Scope/sequencing decisions live in `docs/ADR/`.

## Language

**V1**:
The Knowledge Base Explorer + Admin scope defined in `PRD.md`: browse/filter/view shots,
resources, comments, and admin CRUD for shots/categories/books/resources. Excludes
Authentication, Training Sessions, and Statistics — see ADR-003.
_Avoid_: "MVP" as a synonym — it has been used loosely for both V1 and V2-scoped features
(e.g. Practice Sessions) in some docs, which is exactly the ambiguity ADR-003 resolves.

**V2**:
The future scope in `MICROSERVICE_ARCHITECTURE.md`: real Authentication, Training Session
Service, and Statistics & Analytics Service. Not started; see ADR-003 for sequencing relative
to V1.

**Soft-deleted** (Shot):
Hidden from all normal queries after deletion, with no restore path in the current milestone.
A distinct, narrower mechanism from **Archived** — deliberately not the same field, so the
future active/archived design isn't constrained by this decision. See ADR-003.

**Archived** (Shot):
Not yet defined. Deferred to the milestone after this one — see ADR-003. Do not conflate with
Soft-deleted above.
_Avoid_: using "archived" or "soft-delete" interchangeably — they will very likely end up as
two different fields/behaviors.
