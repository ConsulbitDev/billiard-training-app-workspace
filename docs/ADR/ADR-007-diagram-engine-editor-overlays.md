# ADR-007: Diagram engine editor overlays (Grid, Sub-Diamond markers)

## 📌 Context

Ahead of building the Ball/Path placement editor (`fe`#19, #20), the user raised a need for
optional visual positioning aids: grid lines connecting Diamonds (for lining up Balls/Paths, and
later a snapping mechanism), and extra Diamond-styled markers at finer-than-regulation spacing
(half-Diamond, tenth-Diamond, "and so on"). Neither is part of the physical table (`ADR-006`) or
the Diagram's actual content (`ADR-005`) — both are purely editing aids.

## 💡 Decision

**Overlays are separate components, not `BilliardTableComponent` features.** `BilliardTableComponent`
stays scoped to the fixed, regulation-defined table (Rail, Playing Surface, Diamonds, Pins) and
remains ignorant of both. Overlays render as consumer-projected content via the `<ng-content>`
slot already built for this purpose (`fe`#18), using the same `diamondToSvg()` conversion. This
also means the eventual read-only Shot Detail view (`fe`#22), which has no reason to ever show a
grid, carries none of this concept.

**Two separate overlay components, not one combined config** — `Grid` (lines) and `Sub-Diamond`
(edge dot markers) are visually and positionally different (lines across the interior vs. dots on
the four edges) and a consumer may want either without the other. Despite sharing the same
underlying "spacing in Diamond Coordinate System units" idea, they are not forced through one
shared config object.

**Grid**: `spacingUnits: number[]` (any value, not a closed set of named presets — 10 = a full
Diamond interval, 5 = half, 1 = a tenth, arbitrary values also valid) lets multiple resolutions
render simultaneously (e.g. `[10, 5, 1]`). `showHorizontalLines`/`showVerticalLines` are shared
across all active spacings, not configurable per-spacing — simpler, and nothing described so far
needs per-spacing orientation; revisit only if that proves genuinely limiting.

**Sub-Diamond**: same `spacingUnits: number[]` idea, but always renders on all four edges
together — no long/short-rail split the way Grid has a horizontal/vertical split, since a real
Diamond has no such split either and a Sub-Diamond is meant to read as "the same thing, finer."

**Neither overlay is ever persisted.** Both are ephemeral editor session state, reset every time
the editor opens — not saved per-Diagram (would pollute actual shot content with tooling state,
the same line ADR-006 already drew for Diamond/Pin number labels) and not saved as a per-user
preference (no such concept exists anywhere in this app yet, and would be meaningful added scope
for something trivially re-toggled each session).

**Snapping is out of scope for this ADR.** Grid is explicitly intended as groundwork a future
snapping mechanism in the Ball/Path editor will use, but snapping behavior itself is not
designed or built here — only the visual/config shape it will eventually read from.

## 🔄 Consequences

**Positive:**
- `BilliardTableComponent` stays exactly as focused as ADR-006 intended — no growth in scope from
  a feature that isn't part of the physical table.
- `spacingUnits: number[]` composes arbitrary simultaneous resolutions for free — "diamonds and
  tenths together" is just `[10, 1]`, no new code path.
- Ephemeral-only state means zero backend/schema impact from this ADR.

**Accepted gaps:**
- No per-spacing orientation control on Grid (all active spacings share one H/V toggle).
- No persistence of overlay preferences across sessions, even for a user who always wants the
  same setup.

**Follow-up needed:**
- When the Ball/Path editor (`fe`#19/#20) is built, it should consume Grid's spacing/orientation
  state as the basis for snap-to-grid behavior, per the intent recorded here.
- If per-spacing orientation or persisted preferences turn out to be genuinely needed once the
  editor is in real use, revisit — both are described above as deliberately deferred, not ruled
  out.
