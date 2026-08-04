# ADR-011: Diagram editor — right-click contextual menu replaces buttons (PC only)

## 📌 Context

`fe`#18–20 shipped the Diagram editor's core (table, Balls, Paths) using `BallListComponent` as a
side panel with inline buttons/labels for every action (Add Ball, remove, color toggle, Add/
Continue/Remove Path, per-bend remove). Reviewing this first-release-quality implementation, the
maintainer flagged the panel as "basic and ugly" for what will almost exclusively be PC usage, and
separately raised a need for tablet ("on-the-fly") shot creation, which the current UI wasn't
designed for at all. These are different problems — a PC interaction upgrade vs. a genuinely
different interaction model for touch (click-to-draw's rubber-band preview depends on hover, which
doesn't exist on a touchscreen) — and were deliberately split. This ADR covers the PC contextual
menu only; tablet support is explicitly deferred to a separate later story.

## 💡 Decision

**A right-click contextual menu replaces every button currently in `BallListComponent`, mapped to
the target that was right-clicked, not one menu everywhere:**

| Right-click target | Menu offers |
|---|---|
| Empty table surface | "Add Ball" → submenu (Cue Ball / Object Ball / Second Object Ball) |
| A ball marker | "Remove Ball"; "Toggle Color" (only if Cue/Object Ball); "Add Path" (only if it has no path yet) |
| A path's end marker | "Continue Path"; "Remove Path" |
| A path's bend marker | "Remove Bend" |

**"Add Ball" via right-click creates the ball at the clicked position directly**, reusing the same
clamp-to-Playing-Surface conversion (`pointerToClampedDiamondPosition`, `ADR-009`) already used for
dragging and path drawing — no more placeholder-then-drag for ball creation. This was never quite
right (it's the same pattern `ADR-010` already dropped for Paths, for the same reason), and
right-click-to-place fixes it for free as part of this change.

**Right-clicking a specific marker removes the need to label/number bends in the UI.** The old
panel needed "Bend 1"/"Bend 2" text labels purely so a flat button list could disambiguate which
bend a remove-click meant. A right-click directly targets one marker, so that disambiguation
problem disappears along with the buttons that needed it.

**`BallListComponent` becomes a minimal read-only panel, not removed entirely.** One row per ball:
role label, a color swatch (no toggle — right-click a ball marker for that now), and a path
indicator with bend count (e.g. "Path · 2 bends") if it has one. No buttons. Kept because an
at-a-glance overview of what balls/paths already exist is useful information a context menu can't
passively provide — you'd otherwise have to right-click every marker just to remember what's
there.

**Tablet/touch support is out of scope for this ADR, on purpose.** Right-click is a PC-only mouse
gesture; no long-press-to-right-click emulation is being added here. The maintainer's tablet need
is real but secondary ("almost exclusively" PC, tablet only "on-the-fly"), and solving it properly
requires rethinking click-to-draw's core mechanic (which depends on a hover state touch doesn't
have), not just adding touch event handlers to this menu — that's real, separate design work,
deliberately not bundled into this pass.

## 🔄 Consequences

**Positive:**
- Every action lives on the target it acts on, rather than a flat button list disconnected from
  the table itself.
- Fixes ball creation's placeholder-then-drag pattern as a side effect, for free, bringing it in
  line with how Path creation already works.
- Removes a whole class of UI bookkeeping (bend numbering) that only existed to compensate for the
  button-list format.

**Accepted gaps:**
- PC-only. Tablet/touch users get no working shot-creation UI from this ADR — that remains a
  known, separate follow-up (see Follow-up needed), not solved incidentally here.
- The read-only panel duplicates some information already visible on the table itself (which ball
  has a path). Accepted as a reasonable trade-off for at-a-glance overview.

**Follow-up needed:**
- Design tablet/touch support as its own story. At minimum this needs a different mechanic for
  Path creation (click-to-draw's rubber-band preview requires hover, which touchscreens don't
  have) — likely tap-to-place without a live preview, or a different gesture entirely — not just
  larger touch targets or long-press-for-context-menu on top of the existing mouse-first design.
