# ADR-009: Diagram engine — Path drawing editor (fe#20)

> **Superseded in part by `ADR-010`.** The "button-driven placeholders + drag" creation-UX
> decision below (and "Add Bend" specifically) is replaced by a click-to-draw interaction before
> `fe`#20 ever shipped — real usage during implementation surfaced the gap faster than expected.
> Everything else in this ADR (the `Path` data model, no-duplicate-start-point decision, shared
> drag-logic helper, post-hoc point dragging, `BallListComponent` placement) is unaffected and
> still current.

## 📌 Context

`fe`#20 lets a user give a Ball (`ADR-008`) a Path: a polyline from its start position to an end
position, with optional bend points representing bank/kick rail contacts. A Ball with no Path is
static (an obstacle). Several decisions were needed: whether a Path duplicates its own start
point, how a user actually creates/edits a Path given no drawing/canvas interaction exists yet in
this codebase, how bend points are ordered and removed, whether to duplicate or share the drag
mechanics built for Balls, and where the editing controls live in the UI.

## 💡 Decision

**A Path does not store its own start point.** `interface Path { bendPoints: Point[]; endPosition:
Point; }`, and `Ball.path?: Path`. The path's start is always read live from the owning `Ball`'s
`position` (`ADR-008`) rather than duplicated — a separate stored start point would risk going
stale the moment the ball is dragged after the path exists, with no mechanism forcing the two to
stay in sync. This is a straightforward single-source-of-truth call, not close.

**Creation/editing is button-driven placeholders + drag, reusing the exact Ball interaction model
from `fe`#19 — not a click-to-draw-on-the-table mode.** "Add Path" creates a `Path` with a
placeholder `endPosition` offset from the ball's current position by `(+5, +5)` Diamond
Coordinate System units (half a Diamond in each direction — enough to not render exactly under
the ball, still close enough to be obviously associated with it), zero bend points. The user then
drags the end marker into place with the same raw-pointer-events mechanism as Balls. "Add Bend"
appends a new bend point at the midpoint of the segment it's being inserted into (previous point
↔ end), which the user then drags into place the same way.

This was a deliberate scope trade-off, made explicitly aware it's provisional: a click-to-place
drawing experience (toggle a "draw" mode, click points directly on the table, click to finish)
would read more naturally for actually *drawing* a shot, but requires new interaction machinery
(mode state, click-vs-drag disambiguation on the SVG, cancel/escape handling) that doesn't exist
anywhere in this codebase yet. Reusing the already-built, already-tested Ball drag pattern ships
this story with zero new interaction paradigms. **Expected to be revisited** once the editor
sees real use — see Follow-up needed.

**Bend points are ordered, append-only, no mid-sequence insertion or reordering in this story.**
"Add Bend" always appends to the end of the sequence (just before `endPosition`), matching how a
path is naturally authored (bends get added in the order the ball actually contacts them: draw
the path, add the first bend, place it, add the second, place it, ...). Removal is a per-bend
delete control; if a user needs a bend earlier in the sequence, they remove and re-add rather
than reordering in place.

**Drag mechanics are extracted into a shared helper, not duplicated.** The pointer-capture drag
dance (`pointerdown`/`pointermove`/`pointerup` + `setPointerCapture`, `getScreenCTM().inverse()`
conversion, Playing-Surface clamping, continuous emit) that `DiagramBallsComponent` built for
Balls is exactly the mechanics the new end/bend markers need too. Rather than copy it into a
second component, it moves into a shared helper (a plain function wiring the three pointer
handlers onto a given element and emitting clamped Diamond Coordinate System positions) that both
`DiagramBallsComponent` and the new `DiagramPathsComponent` call. This is a direct lesson from
the real `trackBy` bug found while shipping `fe`#19 (`ADR-008`) — duplicating this logic doubles
the surface for that class of bug; extracting it means the fix (and any future fix) exists in
exactly one place.

**`DiagramPathsComponent` renders the polyline + end/bend markers, following the same
attribute-selector + `<svg:g>` projection pattern as every prior overlay/content component.** It
takes the same `balls: Ball[]` input as `DiagramBallsComponent` (filtering internally for balls
with a `path`) rather than a separate parallel `paths` structure keyed by ball id — avoids a
second data shape that could drift out of sync with the balls list.

**Path editing controls extend each ball's existing row in `BallListComponent`**, rather than a
separate panel: "Add Path" / "Remove Path", and once a path exists, a small nested list of its
bend points (each removable) plus "Add Bend" — all under/beside that ball's row. Keeps everything
about one ball's configuration in one place. **Also noted as provisional** — see Follow-up
needed.

## 🔄 Consequences

**Positive:**
- Zero new interaction paradigms — Path creation/editing reuses Ball's proven drag mechanics
  entirely, both conceptually and (after the extraction) in actual shared code.
- Single source of truth for a Path's start point removes an entire class of stale-data bug
  before it could exist.
- Shared drag-logic helper means the `fe`#19 `trackBy` lesson is encoded once, not duplicated
  into a second component that could independently regress the same way.

**Accepted gaps:**
- Button-driven placeholder creation is a coarser UX than directly drawing a path on the table —
  explicitly acknowledged as provisional, not a final design.
- No mid-sequence bend insertion or reordering — a wrong-order bend must be removed and re-added.
- Path controls living inline in the ball list row work for the current small feature set but
  will get cluttered as more per-path options are added.

**Follow-up needed:**
- Revisit path creation as a real click-to-draw-on-the-table interaction once the editor sees
  real use and the current placeholder-and-drag flow's limitations are concretely felt — the
  maintainer already expects this to change, not just tolerates it as-is.
- Revisit `BallListComponent`'s inline path controls as a contextual menu (e.g. right-click on a
  ball) instead of always-visible inline controls, per the maintainer's stated direction, once
  more per-ball/per-path actions exist to justify it.
