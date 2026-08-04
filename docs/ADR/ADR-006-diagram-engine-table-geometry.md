# ADR-006: Diagram engine table geometry and coordinate system

## 📌 Context

`ADR-005` settled the *functional* shape of the Diagram engine (Diagram, Ball, Path, roles,
coexistence with Resources) but explicitly left the table's *geometric* shape unaddressed — the
POC at `C:\dev\learning\html\canvas-test` hardcodes real-world dimensions (table 179×324cm,
playable surface 142×284cm, frame 12.7cm, rail 6cm, ball diameter 6.15cm) that had never been
validated against any authoritative source, and no coordinate system had been chosen for what a
Ball's position or a Path's points actually store in the Diagram JSON.

The user provided the official FIBiS regulation for these games
(`resources/Rgolamento_gioco_5birilli_goriziana.pdf`, "5 Birilli - 9 Birilli Goriziana - Tutti
Doppi", in force from the 2022/23 season) as the authoritative source, superseding both the POC's
guesses and an initial web-research pass (which was itself corrected in one place — ball diameter
— once the official PDF was checked).

## 💡 Decision

**Table geometry, validated against the official regulation** (see `CONTEXT.md`'s "Table
geometry" section for the full glossary):

- Playing Surface: 284cm × 142cm (±5mm) — the POC's number was already correct.
- Ball diameter: 61–61.5mm — the POC's 6.15cm was already correct; an earlier Wikipedia-sourced
  claim of 63.2mm was wrong and is retracted.
- Rail + Cushion (combined): 12.5–15cm total width from the outer table edge to the Playing
  Surface, of which 5cm is the Cushion (the rubber rebound strip) — the POC's `railWidth: 6`
  should be corrected to 5cm. See `CONTEXT.md` for the Rail/Cushion terminology split.
- Pin (Birillo): 25mm tall, 7mm/10mm/7mm diameter profile, 66mm inter-Pin spacing, arranged in a
  "Castello" along the median line. Not modeled by the POC at all; now in scope as a visual
  element (see below).
- Diamond: 9 per long rail, 5 per short rail (corners included), at 35.5cm (1/8 of the Playing
  Surface's length) intervals. This does not match the POC's arbitrary 5-top/bottom,
  10-left/right decorative dots.

**No pockets** on the table for either Italiana (cue-stick games) or Goriziana — confirmed by the
regulation (rules moved to pocketless tables in 1983); this was already the working assumption
and required no correction.

**Pins are in scope**, as a visual element and as a second class of numberable reference point
alongside Diamonds (see Numbering System below) — not merely decorative.

**Numbering System, built in two stages:**
- **In scope for the initial engine milestone:** ad hoc, per-Diagram number labels on any
  Diamond or Pin, entered by whoever authors that Diagram, with no reuse across Diagrams. Without
  this, a `NUMBERING`-topology Shot's Diagram cannot convey a system at all, defeating that
  topology's purpose — so this could not be deferred entirely the way other engine features were.
- **Deferred:** a reusable, named numbering scheme that multiple Diagrams could reference instead
  of each re-entering the same numbers. Real complexity, correctly pushed past this milestone.

**Diamond Coordinate System — not centimeters — is what a Ball's position and a Path's points
persist in the Diagram JSON.** Raw cm coordinates were rejected because they are not reproducible
by a player on a real table without a ruler; counting Diamonds and estimating tenths between them
is how positions are actually communicated and reproduced in practice. Specifically:

- Each Diamond interval (35.5cm) equals 10 coordinate units.
- Axes run **0–80 along the long rail, 0–40 along the short rail** (continuous, not restricted to
  whole tens — a drag-to-position editor writes arbitrary values within this range; Diamonds
  simply land on round multiples of 10).
- Origin (0, 0) is the bottom-left corner of the "quadrato inferiore" (the near/acchito square
  where play starts, per the regulation's own Tavola 1 diagram) — X grows across the short rail,
  Y grows up the long rail, away from the player.
- Example: (20, 40) is the exact center of the table.
- Centimeters remain the unit for the *static* Table geometry constants (Playing Surface, Rail,
  Diamond spacing) — those don't vary per-Diagram and aren't affected by this decision. Only
  per-Diagram Ball/Path data moves to the Diamond Coordinate System.

**Persistence consequence:** Diamond/Pin *positions* are a static rendering constant derived from
Table geometry (not stored per-Diagram) — but Diamond/Pin *number labels*, being per-Diagram
authoring data, must live in the Diagram JSON alongside Balls/Paths.

## 🔄 Consequences

**Positive:**
- Ball/Path data is expressed in a form a player can actually use to reproduce a shot on a real
  table, which a cm-based or pixel-based system could not provide.
- The Diamond Coordinate System is a pure linear rescaling of physical space (1 unit = 3.55cm
  uniformly on both axes), so it costs nothing in precision or renderer complexity relative to
  cm — it's the same math with a more meaningful unit.
- Table geometry now matches an authoritative source instead of unverified POC guesses, reducing
  the risk of building the SVG renderer (`fe`#18) around wrong proportions.

**Accepted gaps:**
- The reusable, named Numbering System is explicitly not built yet — only ad hoc per-Diagram
  number labels. Diagrams that should logically share one system's numbering will each need their
  own numbers entered until that follow-up work happens.
- Pin-based numbering (as opposed to Diamond-based) is scoped as "in principle supported" but not
  worked out in the same detail as Diamonds in this ADR.

**Follow-up needed:**
- `fe`#18 (SVG table rendering) and the eventual editor stories should implement Diamond/Pin
  rendering and the Diamond Coordinate System conversion described here, not the POC's numbers.
- When the reusable Numbering System is eventually designed, it should build on top of the ad hoc
  per-Diagram labels defined here rather than replacing the coordinate system itself.
