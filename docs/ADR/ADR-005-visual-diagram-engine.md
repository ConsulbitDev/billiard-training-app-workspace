# ADR-005: Visual Diagram engine replaces scanned images for new shots

## 📌 Context

Shots and systems are currently represented by scanned images (books, web searches) stored as
Google Drive `Resource`s (see `billiard-training-app-fe/docs/adr/0001-google-drive-resource-viewer.md`).
This has real, accumulated problems: inconsistent scan quality, layout, and ball colors; margin
instructions on all four sides of the page that are often illegible; some scanned pages actually
depict two unrelated shots that should be two separate `Shot` records; no way to duplicate a shot
and make a small adjustment; and copyright exposure from scanning book pages. It also just feels
like a photo album bolted onto a training app, not a purpose-built billiard tool.

A rough POC already exists outside this workspace at `C:\dev\learning\html\canvas-test`: vanilla
JS + Canvas 2D, with real-world-cm-to-pixel table/ball scaling and basic ball drag-and-drop. No
persistence, no paths, no editor beyond dragging three hardcoded balls.

`ADR-003` scoped "the next milestone" as three specific V1 close-out items (add-shot mode,
delete-shot, YouTube URL normalization) before any V2 work (real Authentication, Training
Sessions, Statistics) begins. The Diagram engine is neither a V1 close-out item nor V2 as
`ADR-003`/`MICROSERVICE_ARCHITECTURE.md` define it — it doesn't fit the existing V1/V2 vocabulary
at all.

## 💡 Decision

Build a visual, coordinate-based **Diagram** engine as a new, first-class part of the `Shot`
domain — see `CONTEXT.md` for full terminology (Diagram, Path, Ball, Cue Ball, Object Ball,
Second Object Ball, Annotation).

**Sequencing:** the three `ADR-003` V1 close-out items ship first. After that, this Diagram
engine work may take priority over starting V2 (Auth/Training Sessions/Statistics) — a deliberate
deviation from `ADR-003`'s "V1 fully closed before V2" rule, made because the current
image-based representation is considered a bigger problem than the absence of V2 features.

**Data model:**
- `Shot.diagram` is a first-class, 1:1 field on `Shot` — not a `Resource`. A `Resource` is a
  pointer to an external Drive-hosted file; a Diagram is native structured data the backend owns.
- A Diagram holds an open list of Balls (not a fixed 3-tuple), each with a role (Cue Ball /
  Object Ball / Second Object Ball today, extendable later), a color, a start position, and an
  optional Path. Italiana and Goriziana are physically 3-ball games, so 3 balls (white cue,
  yellow cue, red object) is the practical default, not a hard constraint.
- A Path is a polyline (start, zero or more bend points for banks/kicks, end). Paths are core,
  not deferred — a Diagram without paths does not meaningfully replace what a scanned
  instructional diagram teaches, since the trajectory is the content.
- Positioned Annotations (text notes anchored to a point/ball/bend, replacing scanned-page margin
  instructions) are named and reserved in the schema as an additive field, but the authoring UI
  for them is explicitly deferred past this first slice.

**Scope boundary:** Diagrams coexist with existing image/video/PDF Resources; they do not replace
them. Existing scanned-image shots (~1,000) are not migrated as part of this work — that is a
separate, incremental effort for later.

**Architecture:** no new microservice. Diagram persistence is a new entity in the existing
`billiard-training-app-be`, stored as a single JSON column plus a `schemaVersion` field (not
normalized relational tables) — the backend stores/returns the blob; interpretation stays in the
frontend engine. The engine (renderer + editor) is a new feature module inside
`billiard-training-app-fe`, rendered with **SVG** (DOM-bound Angular elements), not Canvas 2D —
despite the POC being Canvas-based — because paths with draggable/addable/removable bend points
need per-element interactivity that Canvas's manual hit-testing-and-redraw model handles poorly
as the number of interactive elements grows. The cm→px scaling math and drag concept carry over
from the POC; the literal Canvas drawing code does not. Extracting the engine into its own
service later remains an open option, not a current plan.

**First vertical slice:** a Diagram editor added to the existing Shot admin create/edit form
(place balls, draw/edit paths with bend points, save), plus read-only Diagram rendering on the
Shot Detail page alongside the existing resource sections. Explicitly out of this slice:
Annotation authoring, path animation/playback, and any bulk migration tooling.

## 🔄 Consequences

**Positive:**
- Shots become structured, duplicable, consistent, and free of copyright exposure going forward.
- The "one scanned page, two unrelated shots" problem is resolved by construction (1:1
  Diagram-to-Shot).
- No backend migration cost from schema evolution as Annotations and other fields are added
  later, since the payload is a versioned JSON blob.

**Accepted gaps:**
- `ADR-003`'s "V1 before V2" sequencing no longer holds unconditionally — this ADR is the record
  of why.
- ~1,000 existing image-based shots keep their current representation indefinitely, until a
  separate migration effort is deliberately started.
- No Annotation authoring UI despite known cases needing it.

**Follow-up needed:**
- A migration plan/ADR for existing scanned-image shots, when that work is prioritized.
- Revisit whether Diagram persistence should move to a dedicated service if/when independent
  scaling or data-isolation needs actually appear.
