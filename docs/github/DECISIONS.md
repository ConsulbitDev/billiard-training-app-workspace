# 📘 Decisions Log

This file tracks key architecture and product decisions for the **Billiard Training App**.  
Each entry points to a full ADR file stored in [`docs/ADR/`](../ADR/).

---

## Format
`YYYY-MM-DD – Decision – Reason – Impact – Link`

---

## Decisions

- 2026-03-09 – Start with a **monolith architecture** – Avoid premature complexity, focus on speed – Simplifies first release – [ADR-001](../ADR/ADR-001-monolith.md)
- 2026-03-09 – Adopt **GitHub Project + Issue Templates** – Ensure consistent workflow – Keeps backlog structured – [ADR-002](../ADR/ADR-002-workflow.md)
- 2026-07-26 – **Close out V1 before starting V2** – Finish the Knowledge Base scope before Training Sessions, Statistics and real Auth – Fixes what "V1" and "V2" mean, and defers Archived – [ADR-003](../ADR/ADR-003-close-v1-before-v2.md)
- 2026-07-26 – **Remove the delete-shot UI pending authorization** – Destructive action with no accounts behind it – Soft-delete stays in the backend; the UI returns with Auth – [ADR-004](../ADR/ADR-004-remove-delete-ui-pending-auth.md)
- 2026-08-02 – Build a **visual Diagram engine** instead of scanned images – Consistent, duplicable, copyright-free shots whose trajectory is legible – New first-class Diagram on Shot, persisted as a versioned JSON blob; no new microservice – [ADR-005](../ADR/ADR-005-visual-diagram-engine.md)
- 2026-08-03 – Fix **table geometry and the Diamond Coordinate System** – Validate against the FIBiS regulation rather than the POC's guesses – All Ball/Path data persists in Diamond units, never cm or pixels – [ADR-006](../ADR/ADR-006-diagram-engine-table-geometry.md)
- 2026-08-04 – Treat **Grid and Sub-Diamond overlays as editor aids** – They are not part of the physical table – Ephemeral, projected components; never persisted as Diagram content – [ADR-007](../ADR/ADR-007-diagram-engine-editor-overlays.md)
- 2026-08-04 – Define the **Ball placement editor** – Balls are an open list, not a fixed 3-tuple – Role-driven colours, drag-to-position, transient editor state – [ADR-008](../ADR/ADR-008-diagram-engine-ball-placement.md)
- 2026-08-04 – Define the **Path drawing editor** – A Diagram without trajectories does not teach the shot – Paths are polylines with bend points, optional per Ball – [ADR-009](../ADR/ADR-009-diagram-engine-path-drawing.md)
- 2026-08-04 – Switch Path drawing to **click-to-draw** – Dragging was the wrong interaction for multi-bend polylines – Replaces drag placement; end point set by click – [ADR-010](../ADR/ADR-010-diagram-engine-path-drawing-click-to-draw.md)
- 2026-08-04 – Replace editor buttons with a **right-click contextual menu** – Buttons did not scale with per-element actions – One shared menu drives Ball, Path and label actions; PC only – [ADR-011](../ADR/ADR-011-diagram-engine-context-menu.md)
- 2026-08-04 – Define the **Diagram JSON schema** (`schemaVersion` 1) – The persisted shape had never been specified – Canonical polyline wire format, UUID Ball ids, loud failure on an unknown version – [ADR-012](../ADR/ADR-012-diagram-json-schema.md)
- 2026-08-05 – Add **Diamond Numbering v1** (`schemaVersion` 2) – A NUMBERING-topology Diagram could not convey its aiming system – Per-Diagram number labels on Diamonds; no reusable named system – [ADR-013](../ADR/ADR-013-diamond-numbering-v1.md)
- 2026-08-05 – Add **Pin Numbering v1** (`schemaVersion` 3) – Same need as Diamonds, for the Castello – Parallel component reusing the Diamond mechanism, gated on speciality – [ADR-014](../ADR/ADR-014-pin-numbering-v1.md)
- 2026-08-05 – Add **Diamond Aiming** for Path drawing – "Every shot is straight-across" is wrong for real bank/kick shots at an angle – Bend points anchor to the Diamond they were aimed through, and re-derive when the shot changes – [ADR-015](../ADR/ADR-015-diamond-aiming.md)
- 2026-08-10 – Add the **Hit: Ball Portion and Spin** (`schemaVersion` 4) – The table drawing shows where the balls go, not how to play the shot – One optional Hit per Diagram; Cue Ball identity derived, never stored – [ADR-016](../ADR/ADR-016-hit-ball-portion-and-spin.md)
- 2026-08-11 – Restructure a Diagram into ordered **Layers** (`schemaVersion` 5) – Teaching a shot needs where it ends and the routes there, not just where it starts – First breaking schema change: `balls` moves inside a Layer, and intermediate states become expressible – [ADR-017](../ADR/ADR-017-diagram-layers.md)
- 2026-08-11 – Make the **Ball a component** rather than a `<circle>` in its container – Everything a Ball is visually was a property of whoever drew it, so appearance had no owner – One component per Ball, unit-circle local space, configured by capability and never by context – [ADR-018](../ADR/ADR-018-ball-as-a-component.md)
- 2026-08-12 – **Freeze the Diagram `schemaVersion` at 6 while the engine is pre-production** – Six bumps all describe empty populations; no Diagram has ever been persisted, so the implied migration chain would be unreachable code – Wire format changes freely without bumping until scanned images are ported into Diagrams; that port is the named trigger that restarts versioning and makes migration handling owed – [ADR-012 (amended)](../ADR/ADR-012-diagram-json-schema.md)
- 2026-08-12 – **Model identity now, authenticate later** – Practice history is the first data that accumulates, and it belongs to somebody, but authorship is generated continuously and cannot be backfilled while access decisions under a single user are trivially recoverable – Singular Author on Shot plus a deferred many-to-many Access Grant; history outlives entitlement via FK + permitted projection; soft-delete becomes an invariant on Shot/Category/Book with one privileged read path; login, plans and groups deferred – [ADR-019](../ADR/ADR-019-identity-before-authentication.md)

---

## How to Add a New Decision
1. Create a new ADR under [`docs/ADR/`](../ADR/)
    - File name: `ADR-XXX-title.md` (increment number).
    - Use template in [`docs/templates/ADR.md`](../templates/ADR.md).
2. Add an entry in this log with date, summary, and link.

---

✅ This log helps us (and future contributors) quickly see **what was decided, when, and why**.
