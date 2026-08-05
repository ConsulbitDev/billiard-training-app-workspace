# ADR-014: Pin Numbering v1 (fe#24 follow-up)

## 📌 Context

`ADR-013` built Diamond number labels for the same v1 Numbering scope `ADR-006` originally
described ("ad hoc, per-Diagram number labels" on "any Diamond or Pin"). Pins were deliberately
deferred out of that pass — `ADR-013`'s own follow-up note flagged that Pins have "their own
placement-rule question (clustered around the Castello, not rail-relative)" that it didn't answer.
This ADR answers it and completes the v1 scope.

Grilling confirmed the interaction/persistence mechanics genuinely are "exactly the same as
Diamond" (the requester's own words) — position-keyed identity, the same popover/context-menu
pattern, the same schema-versioning approach. The only real open question was placement, because
Pins don't sit on a rail the way Diamonds do.

**Pin geometry** (`table-geometry.ts`): a Castello — one center Pin plus 4 arms (up/down/left/
right), each with `PINS_PER_ARM[speciality]` Pins (1 for Italiana, 2 for Goriziana) spaced
`PIN_SPACING_CM` (6.6cm) apart, radius `PIN_RADIUS_CM` (0.5cm). This spacing is tight enough that
it directly ruled out reusing Diamond's placement approach unmodified: a radially-outward offset
(push the label further out along the same direction the Pin already sits from center) would land
a Goriziana arm's inner Pin's label on top of that same arm's outer Pin.

## 💡 Decision

**A new `DiagramPinLabelsComponent`, parallel to `DiagramDiamondLabelsComponent`, not a
generalization of it.** Matches this codebase's established convention of keeping conceptually
distinct overlays separate even when the mechanics rhyme (`ADR-007`: Grid vs Sub-Diamonds, "even
though both are spacing-driven overlays"). Diamonds and Pins are different domain concepts with
different placement rules; shared math (position identity, the popover/context-menu interaction
shape) lives in shared utilities both components call, not in a merged component.

**Placement, arm Pins: perpendicular to the arm, not radial.** Rotate each arm's direction vector
90° clockwise (SVG space) and offset the label along that perpendicular, by a fixed gap. Every Pin
on the same arm gets its own lane beside the arm's line, so how many Pins are on it (1 vs 2) never
matters — no collision is possible by construction, unlike the radial approach this replaces.

**Placement, center Pin: a fixed 45° diagonal (up-right), not derived from any arm.** The center
Pin has no arm direction to rotate — a diagonal was chosen deliberately over a cardinal direction
specifically so it reads as visually distinct from the arm-relative Pins, not because it was
derived from anything. Arbitrary among the four diagonals; up-right was picked for consistency,
nothing more.

**Gap magnitude is smaller than Diamond's**, sized against `PIN_SPACING_CM` (6.6cm) rather than
the Diamond's much larger rail-relative spacing (35.5cm) — large enough to clear the Pin marker
(`PIN_RADIUS_CM` 0.5cm) and read clearly, small enough not to encroach on an adjacent arm's Pins
or the central cluster. No canvas-margin concern here at all (unlike Diamonds, `ADR-013`): Pins
sit near the table center, nowhere near the rail or canvas edge, so nothing new is needed from
`table-viewbox.ts`.

**Visibility gating mirrors the Sub-Diamond pattern exactly, with `speciality` as the gate instead
of a spacing layer.** A Pin label is only clickable/visible while its position is part of the
*currently selected speciality's* Castello — e.g. a 2nd-arm-Pin label (Goriziana-only) persists in
the Diagram's data but isn't shown while Italiana is selected, the same way a Sub-Diamond label
persists while its spacing layer is toggled off. Switching speciality never loses data, only
current visibility.

**Identity, schema, and interaction are identical to `ADR-013`, unchanged in kind:** a label's
identity is its position (Diamond Coordinate System units, via the same `svgToDiamond`/
`diamondToSvg` boundary), left-click opens the same PrimeNG `Popover` numeric-entry pattern,
right-click opens the same context menu with "Remove number". The Diagram JSON schema gains a
`pinLabels` field, bumping `schemaVersion` to 3 (`ADR-012`'s reserved mechanism, used a second
time).

## 🔄 Consequences

**Positive:**
- Completes the v1 Numbering scope `ADR-006` originally described for both Diamonds and Pins.
- No changes to `BilliardTableComponent`, `table-viewbox.ts`, or `DiagramDiamondLabelsComponent` —
  purely additive.
- The perpendicular-offset rule is speciality-agnostic by construction — Italiana's 1-Pin arms and
  Goriziana's 2-Pin arms both just work, no special-casing per speciality in the placement math.

**Accepted gaps:**
- The center Pin's diagonal direction is a one-off placement rule with no reusable pattern behind
  it (unlike the arm Pins' rotation rule) — acceptable since there's exactly one center Pin, ever.
- `schemaVersion: 3` breaks the (dev-only, localStorage-stub) diagrams saved under `fe`#21's
  harness, same as `ADR-013`'s bump did. Zero real consequence — no backend-persisted data exists
  yet (`be`#72 remains unbuilt).

**Follow-up needed:**
- None specific to Pins — this closes out the placement question `ADR-013` deferred. The
  reusable, named Numbering System `ADR-006` deferred for both Diamonds and Pins remains future
  work, unaffected by this ADR.
