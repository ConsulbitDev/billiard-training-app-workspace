# ADR-013: Diamond Numbering v1 (fe#24)

## 📌 Context

`ADR-006` scoped Numbering System v1 as "ad hoc, per-Diagram number labels on any Diamond or Pin,"
deferring only a reusable, named scheme shared across Diagrams. `ADR-012` reserved the schema for
this: "no placeholder fields for Numbering System labels — when they ship, they become new fields
under a bumped `schemaVersion`." Neither ADR worked out the actual mechanics, because `fe`#24
hadn't been designed yet. This ADR does that, for Diamonds only — Pin numbering follows later as
its own small story, reusing the same mechanism.

Grilling this surfaced two things that weren't obvious from the issue text alone:

1. **"System" already means "Diagram" here, not a new entity.** The request described "the same
   diamond may have different values for different systems," which sounds like the deferred
   reusable-System concept from `ADR-006`. It isn't: a real-world "system" (e.g. "the 50 diamond
   system") maps onto a single Diagram in this app. Different systems are simply different
   Diagrams, each independently numbered — no new persisted entity needed, `ADR-006`'s scoping
   stands unchanged.
2. **Numbering targets aren't limited to the 28 real, whole Diamonds.** `CONTEXT.md`'s existing
   Sub-Diamond entry framed it as "purely a positioning aid... no regulation basis," implying it's
   unrelated to Numbering. That was wrong: real aiming systems routinely compute fractional aim
   points (a subtraction landing on a half- or tenth-Diamond value) as legitimate reference
   positions. A Sub-Diamond *position* must be numberable, even though the Sub-Diamond *overlay*
   used to reach it while authoring stays the ephemeral, non-persisted editing aid `ADR-007`
   describes. `CONTEXT.md`'s Diamond, Sub-Diamond, and Numbering System entries were corrected
   inline as part of this grilling session.

## 💡 Decision

**A number label's identity is its position, in Diamond Coordinate System units — not an array
index.** `getDiamondPositions()`/`getSubDiamondPositions()` compute positions fresh on every
render with no stable id; the position itself is the only thing that means anything across
renders. Using Diamond Coordinate System units (not the cm those functions return internally)
keeps the entire Diagram JSON in one consistent, player-reproducible coordinate system, matching
how Ball/Path positions already persist (`ADR-006`, `ADR-012`) — convert via the existing
`svgToDiamond()`/`diamondToSvg()` at the same save/load boundary `diagram.ts` already owns.

**Schema (`ADR-012`, bumped to `schemaVersion: 2`):**
```json
{
  "schemaVersion": 2,
  "speciality": "goriziana",
  "balls": [...],
  "diamondLabels": [
    {"position": {"x": 0, "y": 40}, "value": 2},
    {"position": {"x": 4, "y": 12.5}, "value": 1.5}
  ]
}
```
`value` is a number (decimals allowed), entered freely, not validated against any real-world
numbering convention (`ADR-006`). A Diamond/Sub-Diamond position with no assigned label simply has
no entry — labeling is opt-in per position. Per `ADR-012`'s own stated policy, no migration path
exists for existing `schemaVersion: 1` data (there is none of real consequence yet — only
localStorage-stub dev data from `fe`#21's harness); loading continues to enforce an exact
`schemaVersion` match and fails loudly otherwise.

**Rendering and click-to-label interaction live in a new `DiagramDiamondLabelsComponent`**
(`[appDiagramDiamondLabels]`, projected into `<app-billiard-table>` exactly like
`DiagramBalls`/`DiagramPaths`/`DiagramSubDiamonds`), not in `BilliardTableComponent` or
`DiagramSubDiamondsComponent` themselves. This holds `ADR-007`'s separation: `BilliardTableComponent`
stays ignorant of per-Diagram editing state, and `DiagramSubDiamondsComponent` stays a pure visual
overlay unaware that Numbering exists. `DiagramDiamondLabelsComponent` takes the same
`SubDiamondLayer[]` spacing config already passed to `DiagramSubDiamondsComponent`, so its set of
clickable/labelable positions — real Diamonds (always) plus whichever Sub-Diamond spacings are
currently toggled — exactly matches what's visibly rendered as dots.

**Editor-mode label visibility is gated by the same spacing-layer toggle that makes a position
reachable.** A label at a Sub-Diamond position only shows/is editable while that spacing layer is
on; a whole-Diamond label has no such gating, since real Diamonds have no visibility toggle at
all. The toggle is purely how an author reaches a position to assign or edit its number — it is
not a condition for the label's *existence*, only for interacting with it in the editor.

**The read-only Diagram view (`fe`#22, Shot Detail) has no spacing-toggle UI, so it auto-shows
every assigned label** at whatever granularity it exists, without drawing the underlying dot
markers — computed from which positions actually have labels in this Diagram, not from any
toggle state (there is none to have).

**Interaction:** left-click a Diamond/Sub-Diamond marker opens a small PrimeNG overlay/popover
anchored to it, with a single auto-focused numeric input; Enter or clicking away confirms, Escape
cancels. Right-click a labeled position opens a context menu with "Remove number" — the same
pattern already used for removing Balls and bend points in this editor, not a new interaction
paradigm.

**Label placement** is derived purely from each marker's own coordinates, no per-corner special
casing: long-rail markers (left/right) place their label just outside the table on that side,
vertically centered on the marker's y; short-rail markers (top/bottom) place their label above/
below, horizontally centered on the marker's x. A fixed small gap separates label from marker.

## 🔄 Consequences

**Positive:**
- No rework of `BilliardTableComponent`, `DiagramSubDiamondsComponent`, or their tests — Numbering
  is purely additive, one new component.
- Reusing `svgToDiamond()`/`diamondToSvg()` and the `SubDiamondLayer[]` config means no new
  coordinate-conversion or spacing-computation logic; the existing, tested functions are the
  single source of truth for both the dots and the labels.
- A real aiming system's fractional aim points are representable, not just whole-Diamond numbers —
  the actual motivating use case from grilling.

**Accepted gaps:**
- A label at a Sub-Diamond position is only reachable for editing while its spacing layer happens
  to be toggled on in the editor — mildly inconvenient if an author forgets which layer a given
  fractional label needs, but avoids a second, parallel "always show every possible fractional
  position" UI that nothing else in this editor has.
- `schemaVersion: 2` breaks the (dev-only, localStorage-stub) diagrams saved under `fe`#21's
  harness. Zero real consequence — no backend-persisted data exists yet (`be`#72 remains unbuilt).

**Follow-up needed:**
- Pin numbering, deferred here, should reuse this same position-keyed, layer-gated mechanism once
  it's designed — Pins have their own placement-rule question (clustered around the Castello, not
  rail-relative) that this ADR deliberately doesn't answer.
- When the reusable, named Numbering System (`ADR-006`'s deferred stage) is eventually built, it
  sits on top of this per-Diagram mechanism rather than replacing it, per `ADR-006`'s own
  follow-up note.
