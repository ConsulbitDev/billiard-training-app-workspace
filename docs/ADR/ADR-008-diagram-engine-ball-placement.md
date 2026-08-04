# ADR-008: Diagram engine — Ball placement editor (fe#19)

## 📌 Context

The next slice of the Diagram engine (`fe`#19) lets a user add Balls to a Diagram and drag them
into starting positions on the table built in `ADR-006`, using the Grid/Sub-Diamond overlays from
`ADR-007` as positioning aids. This is greenfield: no `Ball`/`Diagram` model exists yet anywhere
in the frontend, no drag-and-drop library is in the repo, and the backend has no Diagram endpoint
to persist to (`ADR-005` describes Diagram as a future JSON-blob field on `Shot`, not yet built).
Several decisions were needed before implementation: the shape of the Ball data model (including
a domain correction to the role→color mapping), how dragging works inside an SVG table with no
existing drag library, where editor state lives, and how far this story's scope should reach.

## 💡 Decision

**Editor-only, in-memory state — no backend persistence in this story.** `fe`#19 builds the Ball
data model and the drag-to-position editor as pure frontend/editor state (a local
`signal<Ball[]>` owned by whichever component hosts the editor — the dev sandbox today, a real
Diagram editor page later). Saving a Diagram to a Shot is deferred to a later story, once the
backend actually exposes a Diagram endpoint — matches how `fe`#35/#36 shipped as pure
rendering/editor primitives with zero backend involvement.

**Ball role/color mapping — corrected from the issue's original wording.** The three Ball roles
(`Cue Ball` / `Object Ball` / `Second Object Ball`, per `CONTEXT.md`) are not each pinned to one
fixed color. Only Cue Ball and Object Ball are player-owned, so only those two vary:
- **Second Object Ball is always red** (the "pallino") — not player-owned, so its color never
  changes, and it is not user-editable.
- **Cue Ball and Object Ball are each white or yellow**, independently toggleable per ball. New
  Cue Balls default to white, new Object Balls default to yellow, so the default 3-ball setup is
  correct without any user action — but the two are **not programmatically enforced to stay
  opposite colors**. An open list already permits multiple balls of the same role (e.g. two Cue
  Balls in an unusual drill), which makes "the opposite of which ball" ambiguous; enforcing the
  constraint is real added complexity for a case that's editor state, not a game-legality
  checker. Left as an accepted gap — see Consequences.

**Default new-Diagram setup**: exactly one ball per role (Cue Ball white, Object Ball yellow,
Second Object Ball red), starting near the table center but offset to avoid overlapping the
Castello pins (e.g. `(10, 20)`, `(20, 20)`, `(30, 20)` in Diamond Coordinate System units) — not
the balls' real regulation starting spots, since the user repositions them immediately by drag
for the actual shot being diagrammed.

**Dragging: raw pointer events, not Angular CDK DragDrop.** `@angular/cdk` is not a dependency of
this repo. CDK's `cdkDrag` positions elements via `transform: translate(...)` in pixel space,
built for HTML layout — it fights rather than helps a `viewBox`-scaled SVG (where 1 user-space
unit ≠ 1 screen pixel), and using it here would still require writing the pixel→Diamond
coordinate conversion by hand, on top of a new dependency. Instead, `pointerdown`/`pointermove`/
`pointerup` handlers on each ball's `<svg:circle>` convert `clientX`/`clientY` through
`getScreenCTM().inverse()` into SVG user-space, then through the already-existing
`svgToDiamond()` into Diamond Coordinate System units — the same conversion path either approach
would need, with no new dependency and full control given the SVG-specific rendering gotchas
already hit building `ADR-007`'s overlays (attribute-selector-only components, no "unknown tag =
generic container" fallback).

**Component API: plain `@Input() balls`, continuous `@Output() ballMoved` during drag.**
`DiagramBallsComponent` follows the same attribute-selector + `<svg:g>` projection pattern as
`DiagramGridComponent`/`DiagramSubDiamondsComponent` (`ADR-007`), taking `balls: Ball[]` in. Drag
position updates emit on every `pointermove`, not only on `pointerup` — the host's signal is the
single source of truth throughout the drag (no internal transient drag-state to reconcile), and
OnPush + signals make the per-frame round-trip cheap. This also sets up cleanly for a future
snap-to-grid mechanism (which `ADR-007` already earmarks the Grid overlay's spacing config for)
wanting to see live positions mid-drag.

**Clamping: radius-aware, to the Playing Surface.** A dragged ball's center is clamped so the
ball's visual edge never crosses the Playing Surface boundary — clamped by `BALL_RADIUS_CM`
inset from each edge, not just the raw center point, so a ball can't visually hang half off the
rail. `BALL_RADIUS_CM` is added to `table-geometry.ts` as the midpoint of the regulation's
61–61.5mm diameter range (radius ≈ 3.0625cm), consistent with how `DIAMOND_RADIUS_CM`/
`PIN_RADIUS_CM` were derived.

**Add/remove UI**: a list panel next to the table (one row per ball: role, color toggle for
Cue/Object Ball only, remove button), plus an "Add Ball" control. No floor on the list size — it
can go to zero balls, since this is transient editor state, not a validated final Diagram.

## 🔄 Consequences

**Positive:**
- Zero new dependencies and zero backend/schema impact from this story.
- Reuses `diamondToSvg()`/`svgToDiamond()` and the attribute-selector overlay pattern from
  `ADR-006`/`ADR-007` without modification — the coordinate and rendering groundwork already
  proved out.
- Continuous-emit drag positions are ready for snap-to-grid to consume later without an API
  change.

**Accepted gaps:**
- Cue Ball/Object Ball colors are not enforced to stay opposite — a user can manually create an
  invalid-looking combination (e.g. two white balls), and multiple balls of the same role are
  permitted with no color-conflict validation at all.
- Default ball start positions are arbitrary placeholders, not the balls' real regulation
  starting spots — acceptable since the user repositions them by drag immediately.
- No backend persistence — a Diagram built in this editor is lost on page reload until a later
  story adds save/load.

**Follow-up needed:**
- When a real Diagram editor page/save flow exists, revisit whether the opposite-color
  constraint needs enforcing after all, once real usage shows whether the gap is actually
  confusing in practice.
- Path drawing (`fe`#20) will extend the Ball model with an optional `Path`, per the `CONTEXT.md`
  Ball entry — deliberately not added to the `Ball` TypeScript interface yet, since `Path` isn't
  implemented and an unused placeholder field would be speculative.
