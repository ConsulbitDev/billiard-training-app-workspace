# ADR-010: Diagram engine — Path drawing switches to click-to-draw (fe#20)

## 📌 Context

`ADR-009` shipped Path creation as button-driven placeholders + drag, reusing the Ball drag
pattern from `fe`#19, as a deliberate scope trade-off — explicitly flagged there as provisional,
expected to be revisited once the editor saw real use. That happened immediately: reviewing the
implementation before it was ever pushed/PR'd, the maintainer found the gap in practice, not in
theory. A real shot's Path very often touches multiple rails (bank/kick shots), and `ADR-009`'s
"Add Bend" only ever inserted a point at the midpoint of the *existing* last segment — there was
no way to extend a Path's endpoint outward to represent a second or third rail contact, only to
subdivide what was already there. Worse, doing so required a button click per point, which the
maintainer specifically called out as the real pain versus the (rare, "nice to have") mid-segment
insertion capability itself.

## 💡 Decision

**Path creation/extension becomes click-to-draw: a live rubber-band preview, no button held.**
"Add Path" starts a drawing session from the ball's actual position (no more `(+5, +5)`
placeholder offset — that concept is gone entirely). While drawing, a preview segment tracks the
pointer on every `pointermove` — the mouse button is **not** held down, unlike every other drag
interaction in this codebase. A single click commits the current pointer position as a bend
(a rail contact) and continues the preview from there. A double-click commits the final point as
`endPosition` and ends the drawing session, at which point the committed points become the Path's
real `bendPoints`/`endPosition`.

**"Add Bend" (`ADR-009`'s midpoint-insertion action) is removed, not kept alongside the new
mechanic.** The maintainer explicitly called it low-priority ("very rarely used... but nice to
have") against the real pain point (no outward extension). Rather than maintain two different
point-insertion semantics in the same story, mid-segment insertion is dropped for now — it can
come back later if it turns out to be genuinely needed (see Follow-up needed).

**"Continue Path" re-enters drawing mode from the current end, for an already-finalized Path.**
Without this, fixing "I forgot a rail contact" after finishing would require deleting and
redrawing the whole Path — a real usability regression from what prompted this ADR in the first
place. Continuing always extends from the current `endPosition`, appending further; it cannot
insert into the middle of an already-finalized sequence (same append-only reasoning as `ADR-009`).

**Escape cancels the active drawing session and discards everything not yet committed.** For an
initial "Add Path" session, canceling reverts the Ball to having no Path at all. For a "Continue
Path" session, canceling reverts to the Path exactly as it was before Continue was invoked —
already-existing segments are untouched, only the in-progress extension is discarded. Nothing
uncommitted is ever silently kept.

**Already-placed points remain individually draggable after finalizing, using the existing
press-hold-drag-release mechanism and shared `draggable-point.ts` helper from `ADR-009` —
unchanged.** Click-to-draw only replaces how points get *created*; repositioning an existing
point is still the same tested drag mechanic, reused as-is. This is why `ADR-009`'s data model,
shared drag-logic extraction, and `BallListComponent` placement all remain valid — only the
creation-time interaction changes.

**Implementation note (not a decision requiring sign-off, but load-bearing for whoever touches
this next):** capturing `pointermove`/`click`/`dblclick` during a drawing session needs a
full-surface hit-testing layer (e.g. a transparent `<rect>` covering the Playing Surface, active
only while drawing), since SVG has no hit-testing over empty space — a bare `<g>` with no filled
geometry underneath the cursor never receives pointer events. This interaction-catcher must sit
on top of the Ball/other markers while a drawing session is active, so accidental clicks during
drawing land on the drawing surface rather than triggering a Ball drag underneath. Drawing state
(which ball, if any, is actively being drawn) stays local to `DiagramPathsComponent`, the same way
`draggingKey` already is — only the finalized Path is emitted to the host, not intermediate
drawing state.

## 🔄 Consequences

**Positive:**
- Matches how bank/kick shots are actually authored — draw straight through every rail contact
  in one continuous motion, not one button click per point.
- Removes the button-per-segment friction that was the maintainer's specific, named complaint.
- `ADR-009`'s data model and post-hoc drag mechanics needed zero changes — only the creation-time
  interaction was wrong, not the underlying representation.

**Accepted gaps:**
- Mid-segment insertion (the old "Add Bend") is gone. If a genuine need for it resurfaces, it
  needs its own design pass rather than resurrecting the old midpoint-insertion behavior verbatim,
  since it would now need to coexist with click-to-draw rather than being the only mechanism.
- Click-to-draw introduces a real new interaction paradigm (a "drawing session" state machine,
  full-surface hit-testing) that this codebase didn't have before — exactly the complexity
  `ADR-009` originally traded away, now taken on because real usage showed it was worth it sooner
  than expected.
- `BallListComponent`'s inline path controls (Add Path / Continue Path / Remove Path / per-bend
  remove) are unchanged from `ADR-009` and still carry that ADR's "expected to become a contextual
  menu" follow-up.

**Follow-up needed:**
- If mid-segment insertion turns out to be genuinely needed later, design how it coexists with an
  already-drawn, click-to-draw-authored Path rather than assuming the old flow.
- `ADR-009`'s contextual-menu follow-up for `BallListComponent` still stands, unchanged by this
  ADR.
