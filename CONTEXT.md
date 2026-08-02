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

**Diagram**:
The structured, coordinate-based representation of a Shot's balls and their paths, as JSON,
rendered by the billiard engine instead of a scanned image. Ball start/end positions and the
trajectory between them (paths) are both core — a Diagram without paths is not a meaningful
replacement for a scanned instructional diagram, since the path is what actually teaches the
shot. Exactly one Diagram per Shot (1:1) — a scanned page showing what looks like two unrelated
shots becomes two Shots, each with its own Diagram, rather than one Diagram holding both. A
`NUMBERING`-topology Shot (a system) still has exactly one Diagram, since a system's multiple
numbered positions are one coherent concept. Diagrams coexist with the existing image/video/PDF
Resource types but are **not** a Resource type themselves — a Resource is a pointer to an
external Drive-hosted file, while a Diagram is native structured data your own backend owns and
serves. It is a separate, first-class 1:1 relationship on Shot (`Shot.diagram`), not a member of
`Shot.resources[]`. Diagrams do not replace scanned-image Resources on existing Shots, and old
shots are not force-migrated.
_Avoid_: "layout" (UI-layout ambiguity), "scene" (unwanted graphics-engine connotation),
"system" (already means something narrower — see `topology: NUMBERING`), treating it as a
Resource subtype.

**Path** (Diagram):
A polyline (start point, zero or more bend points, end point) describing one ball's trajectory
within a Diagram. Bend points represent bank/kick rail contacts. A ball with no Path is static
(sits in place, e.g. as an obstacle) — Paths are optional per ball, not mandatory.

**Cue Ball**:
The ball the current player is shooting/striking with, in a given Diagram.

**Object Ball**:
The ball the cue ball is initially directed at.

**Second Object Ball**:
The other ball on the table — not the cue ball's initial target — that may become involved
indirectly, via carom or combination, during the shot. Term borrowed from English-language
carom/three-cushion billiards terminology, chosen specifically to replace "pallino."
_Avoid_: "pallino" — colloquially just the Italian word for "ball" (as in "pallino bianco"),
not specific to this role, and ambiguous as a result.

**Annotation** (Diagram):
A short text note anchored to a specific point/ball/bend in a Diagram, meant to replace the
margin instructions found on scanned pages. Not yet structurally defined — deferred past the
initial engine milestone. The schema reserves room for it (an additive field), but the editor
does not support authoring it yet.
_Avoid_: assuming `Shot.description` alone covers this — some Diagrams are known to need
positioned, per-point notes that a single free-text field can't represent.

**Ball** (Diagram):
One entry in a Diagram's open list of balls. Has a role (Cue Ball / Object Ball / Second Object
Ball today, extendable later), an associated color, a start position, and an optional Path.
Not a fixed 3-tuple — the list can hold more or fewer than 3 if a shot needs it — but Italiana
and Goriziana are physically 3-ball games, so 3 (white cue, yellow cue, red object) is the
practical default.
