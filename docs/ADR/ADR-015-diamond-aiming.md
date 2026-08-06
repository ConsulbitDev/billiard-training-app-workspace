# ADR-015: Diamond Aiming for Path Drawing

## 📌 Context

Drawing a Path today (`ADR-010`'s click-to-draw mechanic) places a point wherever the cursor
happens to be on the Playing Surface — free-form, no assistance. In real play, a Diamond is the
actual reference point a player aims at ("aim for a 10" means aiming at Diamond 10), and a system
Diagram (`NUMBERING` topology, `ADR-013`) is only as useful as how precisely its Paths actually
point at the Diamonds it references. Free-hand mouse placement can't reliably land on the exact
point a real aimed shot would touch the Cushion.

**This ADR replaces an earlier, abandoned attempt (previously numbered ADR-015, "Diamond
Snapping") that was never merged.** That version assumed a drawn point's Cushion contact value
always equals the Diamond's own along-rail coordinate — i.e. "aiming at Diamond 20" always means
the ball touches the Cushion at value 20. Live correction from the reporter (after two rounds of
live-testing on the earlier version, and one dedicated grilling session) established that this is
physically wrong except for a straight, zero-angle shot. The two are the same point only when the
segment's start position lines up exactly opposite the Diamond; for any other start position, the
real Cushion contact point is determined by the actual straight-line geometry of the shot, and
will land at a different value — the earlier version's core assumption baked in an implicit
"every shot is straight-across" simplification, which real bank/kick shots at an angle violate
constantly.

**Vocabulary, settled during grilling:**
- **Diamond** — fixed, physical position on the Rail. Never moves; doesn't depend on the shot.
- **Aiming Ray** — the straight line from wherever the current Path segment starts (the Ball's
  own position, for the first segment; the previous bend point, for any later one) through the
  Diamond being sighted on.
- **Cushion Contact Point** — where that Aiming Ray actually crosses the Cushion (the Playing
  Surface edge). This is the point that gets placed as the bend/end point. It depends on *both*
  the segment's start point and which Diamond the ray is aimed through — the Diamond itself never
  moves, only the Ray and its Cushion crossing do.

**Round 2 addition — a Cushion Contact Point's dependency on its segment's start point cuts both
ways.** The first round only computed the Cushion Contact Point once, at the moment a point was
placed, then stored it as a plain coordinate — exactly like every other Point in this engine.
Live testing surfaced the gap: repositioning the Ball afterward (or dragging an earlier bend
point) left every later point frozen at its old Cushion coordinate, even though the *reason* that
point was there — "aimed at Diamond X" — hadn't changed, only the Ray's start had. A point placed
via Diamond Aiming isn't really "a coordinate that happened to result from a ray computation" —
it's "aimed at Diamond X", full stop, and that fact needs to persist and keep being honored as
everything upstream of it moves. Separately, dragging an already-placed marker had no Diamond
Aiming applied at all in round 1 (an explicit accepted gap) — once a point could carry a
permanent Diamond anchor, leaving drag as pure free movement became inconsistent: you could aim a
point at a Diamond while drawing it, but never again once it existed.

## 💡 Decision

**Scope: the click-to-draw flow only** (`onDrawPointerMove`/`onDrawClick`/`onDrawDblClick` in
`DiagramPathsComponent`) — same scope as the abandoned attempt. Dragging an already-placed
end/bend marker, and Ball dragging, are both untouched.

**Mechanism: real ray-cushion geometry, not a static per-Diamond projection.** A new pure
`aimAtDiamond(cursorPoint, startPoint, radiusCm)` (`core/utils/diamond-aim.ts`) takes the
segment's actual start point as a required input (unlike the abandoned version, which needed no
such input — a direct symptom of its flawed model). For each of the 28 Diamonds, it computes the
real Cushion Contact Point for a ray from `startPoint` through that Diamond (`rayExitPoint`, a
standard AABB "ray exit" / slab-method computation against the radius-inset Playing Surface
rectangle), then measures how close the already-on-Cushion cursor is to *that specific Diamond's*
contact point. Within `DIAMOND_AIM_RADIUS_UNITS` of the nearest one, that Diamond's real contact
point is what gets placed; otherwise the raw cursor position is used unchanged. This was a
deliberate simplification over an angle-based "closest aiming direction" comparison floated
during grilling — comparing cursor-to-contact-point distance reuses the exact same
nearest-within-radius structure the abandoned version already had, needs no new plumbing for raw
unclamped cursor coordinates, and was judged close enough in practice; verified live with a
genuinely off-axis shot (Ball at Diamond Coordinate x=10, aiming near the Diamond at x=20) landing
at x≈19.55, not 20 — confirming the fix actually changes behavior for angled shots, not just in
the straight-across case the abandoned version also got right by accident.

**Round 3 — the tolerance itself was wrong, not just generous.** `DIAMOND_AIM_RADIUS_UNITS` was
initially set to 2 units (~7cm), deliberately wide, on the reasoning that a forgiving zone removes
the need for pixel-precise aim. Live use found that reasoning backwards: a ~7cm catchment around
Diamond 20 meant any cursor position within ~20% of a whole diamond interval snapped to 20,
making it *impossible* to place a point at a nearby non-Diamond value like 19 while hovering
anywhere near 20. The rule is now "the cursor's on-Cushion position intersects the Diamond's own
rendered mark" — `DIAMOND_AIM_RADIUS_UNITS` derives from `DIAMOND_RADIUS_CM / CM_PER_DIAMOND_UNIT`
(≈0.34 units, ≈1.2cm) instead of a hardcoded, independently-chosen number, so the tolerance always
matches whatever size the Diamond is actually drawn at. The proximity *metric* itself (distance
from cursor to each Diamond's dynamic Cushion Contact Point) is unchanged from round 1 — only the
radius shrank.

Because the start point now matters, `DiagramPathsComponent` had to start passing it explicitly at
every call site — critically, `onDrawDblClick` must use the already-deduped `bendPoints` list (not
the raw, still-duplicated `session.committedPoints`) to compute the correct start for the final
segment. See `ADR-010` for why that dedup exists.

**Highlight: the aimed-at Diamond's own marker changes fill/radius, no new visual element.**
Unchanged in substance from the abandoned version's design — `DiagramPathsComponent` reports the
matched Diamond's SVG position via a `snappedDiamondChanged` output (null when unmatched or
disabled); the host forwards it to `BilliardTableComponent` (which already renders the Diamond
markers) as a `highlightedDiamond` input, applying `.diamond--highlighted` (accent fill, ~1.5×
radius) to the matching marker. This remains valuable — arguably more so now, since the placed
point is often visibly *not* at the Diamond's own position, so confirming *which* Diamond you're
sighted on matters more than ever.

**Toggle and cursor:** unchanged from the abandoned version's design — right-click the drawing
surface mid-session to toggle "Enable/Disable Diamond Aiming" (defaults enabled, ephemeral editor
state, `ADR-011`'s host-builds-the-menu convention), and crosshair cursor across the whole table
during an active drawing session (via the sandbox's existing `drawingRequest`/`session` signals).

**Round 2 — the anchor is a permanent, persisted fact, not editor-session state.** A bend/end
point's schema (`PathPoint`, `core/models/path.ts`) gains an optional `aimedDiamond: Point` (the
Diamond's own SVG position) alongside its always-present resolved `x`/`y`. This was decided as a
schema change now, deliberately, while it's nearly free: there's no real backend Diagram support
yet (`be#72` isn't built; persistence is still the `fe#21` localStorage stub), so there's no
production data to migrate. A point without `aimedDiamond` is a plain free-placed point, exactly
as before — this is additive, not a breaking change to the wire format
(`DiagramJson.balls[].path.points`, `ADR-012`).

A new pure `resolvePath(ballPosition, path)` (`core/models/path.ts`) walks a Path's points in
order, treating each resolved point as the next one's segment start — so it re-derives every
anchored point's Cushion Contact Point in one pass, cascading correctly through a chain of
multiple anchored points. It's called wherever something could invalidate an anchored point's
segment start: the sandbox's `onBallMoved`, `onPathPointMoved` (dragging any marker), and
`onBendRemoved` (removing a bend changes what the *next* point's start is). It reuses
`cushionContactPointForDiamond` — the same single-Diamond ray-cast `aimAtDiamond` already used
internally, now extracted so both call sites share one geometry implementation.

**Dragging an already-placed end/bend marker now goes through Diamond Aiming too**, gated by the
same `snapToDiamondsEnabled` toggle, using the same nearest-within-radius search — the accepted
gap from round 1 is closed. The segment start for a dragged marker comes from the Ball's *current*
Path data (`ball.path!.bendPoints`/`ball.position`), not from `session`, since dragging happens
with no drawing session active — `DiagramPathsComponent`'s `segmentStart()`/`snapped()` helpers
take the Ball's position as an explicit parameter for this reason, rather than reading
`drawingBall()` (which returns `undefined` outside an active session).

**Round 4 — the proximity metric itself was measuring the wrong thing.** Every prior round
compared the *clamped* cursor position against each Diamond's computed Cushion Contact Point — a
metric that seemed reasonable but has a structural flaw: the clamped cursor can never actually
reach a Diamond (it's always pulled back onto the Playing Surface, by design), so "distance to the
contact point" is not the same question as "is the cursor over the Diamond". Live feedback caught
this directly: a snap could fire while the mouse was visibly nowhere near the Diamond on screen,
because the *contact point* — not the Diamond — happened to be close to wherever the clamped
cursor was. The fix replaces the trigger with a literal hit-test: `aimAtDiamond()` now takes the
**raw, unclamped** cursor position (`pointerToSvgPoint()`, new in `draggable-point.ts`) and
compares it directly against each Diamond's own true SVG position, within `DIAMOND_AIM_RADIUS_CM`
(simplified to just `DIAMOND_RADIUS_CM` now that the comparison happens in the same SVG space the
Diamond is actually drawn in — no unit conversion needed). The *placement* step is unchanged: once
a Diamond is hit, its real Cushion Contact Point (ray from the segment's start through that
Diamond) is still what gets placed — the hit-test and the placement are deliberately two separate
steps using two different points. This needed raw-cursor plumbing through all five call sites in
`DiagramPathsComponent` (three drawing handlers, two drag handlers), since only the clamped
position existed there before.

**Round 5 — Diamond Aiming closes the Sub-Diamond gap accepted back in round 1.** With the
tolerance now tied to each mark's own rendered radius, extending aiming to Sub-Diamond positions
(half/tenth spacing, ADR-007) is no longer a separate design question — the same hit-test model
already generalizes, it just needs a bigger candidate list. `aimAtDiamond()` now takes that list
as an explicit `candidates: SubDiamondPosition[]` parameter instead of internally calling
`getDiamondPositions()`, and each candidate hit-tests against its *own* `radiusCm` (a tenth-mark's
tiny dot needs a tighter hover than a real Diamond's, exactly matching how it's actually drawn).
`DiagramPathsComponent` builds this list via a new `aimablePositions` getter — real Diamonds plus
whichever Sub-Diamond layers `subDiamondConfig` currently has enabled — mirroring
`DiagramDiamondLabelsComponent`'s own `clickablePositions`, which already solved this exact
"aim/click targets should match what's actually visible" problem for Diamond Numbering (`ADR-013`).
The persisted schema needs no change: `PathPoint.aimedDiamond` is just a `Point`, indifferent to
whether it came from a real Diamond or a Sub-Diamond, so an anchor to a half-diamond mark already
persists and re-resolves correctly through the existing `resolvePath()`. The highlight extends too
— `DiagramSubDiamondsComponent` gets its own `highlightedPosition` input, fed the same
`highlightedDiamond` signal `BilliardTableComponent` already uses; since real-Diamond and
Sub-Diamond positions never coincide, at most one of the two ever shows a highlight for a given
match.

**Round 6 — dragging was never actually the problem; the affordance was.** Live feedback: "if I try
to move the end point of the line, it moves along the cushion and I can't point to a diamond any
longer." Round 2 already ran drag-to-reposition through the exact same `aimAtDiamond()` hit-test as
click-to-draw — the aiming math itself was never broken for drag. The real gap was the *interaction
affordance*: click-to-draw gives a live rubber-band preview and a crosshair cursor while you hunt
for a ~1.2cm hit-test radius; a press-hold-drag gives neither, so the marker stays visually pinned
near the Cushion for the whole gesture and only jumps once the raw cursor happens to land exactly on
a Diamond, with no visual guidance getting you there. The fix removes the drag mechanic entirely:
clicking an already-placed end/bend marker now starts the **same kind of session** click-to-draw
already uses, scoped to that one point (`DrawingSession.repositioning`) — live preview, crosshair,
a single commit click finalizes it via the existing `pathPointMoved` output and closes the session.
There is no more `setPointerCapture` anywhere in `DiagramPathsComponent`; `capturePointer()`
(`draggable-point.ts`) is now exclusive to Ball dragging (`DiagramBallsComponent`), which is
untouched — Balls move freely on the Playing Surface and were never subject to Diamond Aiming.
Repositioning a point that has segments *after* it keeps the tail attached rather than reopening it
for redraw: the point's own commit is the only thing that changes, and every later anchored point in
the chain re-cascades automatically via the existing `resolvePath()` (already wired to every
`pathPointMoved` consumer since Round 2) — no new plumbing needed for this, it already worked this
way for drag.

**Round 6 — "the aiming line is always aiming to a Diamond" is now a hard invariant, not an
assist — for bend points.** Grilled directly: free (non-Diamond) placement is removed for every
bend, whether placed via a plain click-to-draw click or a `kind: 'bend'` reposition click. A bend
commit that isn't currently aimed at a Diamond/Sub-Diamond is a no-op: the session simply stays
open, previewing wherever the cursor currently clamps to (shown with a distinct dimmed/dashed
style, `.path-line--preview-unaimed`), until the cursor lands on a mark. This is a real behavior
change, not just a UX polish, so it's documented here rather than folded silently into Round 6's
interaction fix.

**Round 6 correction — the Path's *end* is exempt.** Live feedback caught an overreach in the
above: "the final double click to end the polyline MUST remain even if it is not pointing to a
diamond because the end of the polyline is a special case." A bend is by definition a rail contact
— aiming at a Diamond is exactly what a bend *means*. A Path's end is different: it's wherever the
ball actually ends up (potted, come to rest, resting against another ball) — there's no domain
reason it has to land on a Diamond, and the earlier version wrongly applied the same mandatory-aim
gate to it. Fixed: the `dblclick` that finalizes a draw/continue session, and a reposition click
for a `kind: 'end'` ref, both commit unconditionally now — `endPosition`/the emitted point simply
carries no `aimedDiamond` when the cursor wasn't over a mark. Bends remain mandatory; only the end
is exempt.

**Round 6 — mandatory aiming broke the click-pair dedup hack (ADR-010) in a provable, universal way,
not just a rare geometric coincidence.** `onDrawDblClick` used to unconditionally strip the last 2
`committedPoints` before finalizing, on the assumption that a native dblclick's two composing
`click` events always append two near-duplicate points. Under mandatory aiming that assumption is
now **always false once the first composing click succeeds**: that click's placement is, by
construction, exactly on the Cushion boundary; the second composing click re-aims the exact same
mark from that new position, which is a ray already sitting on its own exit edge — provably
degenerate (`tExit <= 0`), so `onDrawClick`'s own aimedDiamond gate silently rejects it every time.
At most one of the two composing clicks ever appends anything now, never both — a fixed
`slice(0, -2)` would sometimes delete a real, unrelated preceding bend along with it (caught by a
test with a real preceding bend before the final double-click; the simpler single-segment case
happened to mask the bug, since `slice(0, -2)` on a committedPoints array of length ≤ 2 gives `[]`
either way, whether 1 or 2 points were really appended). Fixed with a new `hitTestDiamond()` helper
— the hit-test half of `aimAtDiamond()`, extracted so it's usable *without* a placement computation,
since placement is exactly what could be degenerate here. `onDrawDblClick` now strips the trailing
`committedPoint` only if it's actually a hit-test match for the mark the cursor is over right now,
capped at exactly one.

**The `snapToDiamondsEnabled` toggle (and its "Enable/Disable Diamond Aiming" context-menu
item, `onDrawingSurfaceContextMenu`, `drawingSurfaceContextMenu` output) is removed entirely** — it
contradicted the new invariant; aiming isn't optional to switch off anymore. **Pre-existing
free-placed points are left exactly as-is** — this invariant is enforced only at the moment of a new
commit, never retroactively; a legacy Path saved before this round keeps rendering/behaving
unchanged until someone actually clicks one of its points to reposition it, at which point the new
position (like any commit now) must land on a Diamond/Sub-Diamond.

## 🔄 Consequences

**Positive:**
- No changes to `pointerToClampedDiamondPosition()` or its other two callers (ball drag,
  marker-reposition drag) — zero regression risk to already-shipped, tested interactions.
- `aimAtDiamond()` is a small, pure, independently-testable function; `diamond-aim.spec.ts`
  includes both a straight-shot sanity case (contact point == Diamond value) and an explicitly
  angled case (contact point != Diamond value, matching the reported bug) as a regression guard
  against silently reintroducing the abandoned version's flawed assumption.
- The context-menu toggle, cursor fix, and highlight all reuse existing patterns/state exactly —
  no new architectural surface beyond `aimAtDiamond()` itself and the start-point plumbing it
  requires.
- `resolvePath()` is a small, pure, independently-testable function too; `path.spec.ts` includes a
  dedicated cascading test (moving the Ball changes an aimed bend point, which changes what a
  *later* aimed point resolves to) as a regression guard — the whole point of persisting the
  anchor is that this cascade keeps working, not just the single-point case.
- The `PathPoint` schema change is additive only — every existing reader of `bendPoints`/
  `endPosition` as plain `{x,y}` keeps working unchanged, since `aimedDiamond` is optional and
  nothing strips it.
- Sub-Diamond support (round 5) needed zero schema or `resolvePath()` changes — the hit-test/
  placement split from round 4 already generalized cleanly to "any candidate with a position and
  a radius", and `aimablePositions` reuses `DiagramDiamondLabelsComponent`'s exact pattern rather
  than inventing a new one.

- Round 6's click-to-reposition mechanic needed no new geometry or schema at all — it reuses
  `aimAtDiamond()`, `resolvePath()`, and the `pathPointMoved`/`pathSessionEnded` outputs exactly as
  they already existed; only the *trigger* (click vs. drag) and the *commit gate* (must be aimed)
  changed. `diagram-paths.component.spec.ts` drops the old drag-based tests (there is no drag left
  to test) in favor of click-to-reposition tests, plus explicit reject-when-unaimed tests for both
  drawing and repositioning.

**Accepted gaps:**
- None specific to the aiming toggle anymore — it's gone (Round 6). Sub-Diamond spacing layers
  (`subDiamondConfig`) remain sandbox-only ephemeral state, same as Grid — a separate concern from
  aiming itself (which layers are *visible to aim at*, not whether aiming happens at all).

**Superseded (kept for history, no longer true):**
- "Proximity is distance-to-dynamic-contact-point, not angle-based" (round 1–3) — round 4 replaced
  this entirely with a direct hit-test against the Diamond's own position; there is no longer a
  distance-to-contact-point comparison anywhere in the trigger path.
- "Dragging an already-placed end/bend marker" (round 1–5) — round 6 removed the drag mechanic
  entirely in favor of click-to-reposition; there is no `setPointerCapture` left in
  `DiagramPathsComponent`.
- "A point without `aimedDiamond` is a plain free-placed point" as something a *new* commit can
  still produce (round 1–5) — round 6 makes this impossible going forward for **bend** points; it
  remains true for bends only as a description of data saved before round 6. A Path's **end**
  point can still be freely placed today, by design (round 6 correction) — that was never
  supposed to be prohibited, only bends.

**Follow-up needed:**
- None specific to this ADR. Pin Aiming (aiming at a numbered Pin rather than a Diamond) would
  need its own treatment — Pins aren't on a rail, so "Cushion Contact Point for a ray through a
  Pin" is a different, not-yet-solved geometry problem (a Pin sits *inside* the Playing Surface,
  not beyond the Cushion, so the ray wouldn't necessarily exit through a Cushion edge at all).
