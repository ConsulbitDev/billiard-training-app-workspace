# Billiard Training App — Domain Glossary

Shared vocabulary across `billiard-training-app-be` and `billiard-training-app-fe`. See
`CLAUDE.md` for the workspace map. This file is a glossary only — no implementation details,
no roadmap, no scratch notes. Scope/sequencing decisions live in `docs/ADR/`.

## Language

**V1**:
The Knowledge Base Explorer + Admin scope defined in `PRD.md`: browse/filter/view shots,
resources, comments, and admin CRUD for shots/categories/books/resources. Excludes
Authentication, Training Sessions, and Statistics — see ADR-003.
_Avoid_: "MVP" as a synonym — it has been used loosely for both V1 and V2-scoped features
(e.g. Practice Sessions) in some docs, which is exactly the ambiguity ADR-003 resolves.

**V2**:
The future scope in `MICROSERVICE_ARCHITECTURE.md`: real Authentication, Training Session
Service, and Statistics & Analytics Service. Not started; see ADR-003 for sequencing relative
to V1.

**Soft-deleted** (Shot):
Hidden from all normal queries after deletion, with no restore path in the current milestone.
A distinct, narrower mechanism from **Archived** — deliberately not the same field, so the
future active/archived design isn't constrained by this decision. See ADR-003.

**Archived** (Shot):
Not yet defined. Deferred to the milestone after this one — see ADR-003. Do not conflate with
Soft-deleted above.
_Avoid_: using "archived" or "soft-delete" interchangeably — they will very likely end up as
two different fields/behaviors.

**Diagram**:
The structured, coordinate-based representation of a Shot's balls and their paths, as JSON,
rendered by the billiard engine instead of a scanned image. Ball start/end positions and the
trajectory between them (paths) are both core — a Diagram without paths is not a meaningful
replacement for a scanned instructional diagram, since the path is what actually teaches the
shot. Exactly one Diagram per Shot (1:1) — a scanned page showing what looks like two unrelated
shots becomes two Shots, each with its own Diagram, rather than one Diagram holding both. A
`NUMBERING`-topology Shot (a system) still has exactly one Diagram, since a system's multiple
numbered positions are one coherent concept. Diagrams coexist with the existing image/video/PDF
Resource types but are **not** a Resource type themselves — a Resource is a pointer to an
external Drive-hosted file, while a Diagram is native structured data your own backend owns and
serves. It is a separate, first-class 1:1 relationship on Shot (`Shot.diagram`), not a member of
`Shot.resources[]`. Diagrams do not replace scanned-image Resources on existing Shots, and old
shots are not force-migrated.
_Avoid_: "layout" (UI-layout ambiguity), "scene" (unwanted graphics-engine connotation),
"system" (already means something narrower — see `topology: NUMBERING`), treating it as a
Resource subtype.

**Path** (Diagram):
A polyline (start point, zero or more bend points, end point) describing one ball's trajectory
within a Diagram. Bend points represent bank/kick rail contacts. A ball with no Path is static
(sits in place, e.g. as an obstacle) — Paths are optional per ball, not mandatory. A Path does
not store its own start point — that's always the owning Ball's current position, read live, so
dragging the Ball keeps the Path's start attached with no separate sync step. See ADR-009.
_Avoid_: treating a Path's start point as independent, persisted data — it is always derived from
`Ball.position`.
A bend/end point may additionally be **anchored** to a Diamond it was aimed through (Diamond
Aiming, `ADR-015`) — a permanent, persisted fact (not editor-session state), since it's part of
what the point actually means, not an incidental coordinate. An anchored point's `x`/`y` is a
*resolved* value, kept in sync (`resolvePath()`) whenever anything upstream in the same Path
changes — the owning Ball moving, or an earlier point in the chain moving or being removed — so
the point keeps tracking the Diamond it was aimed at rather than freezing at a stale coordinate.
**Being anchored is mandatory for every newly-placed or newly-repositioned bend point** — "the
aiming line is always aiming to a Diamond" (`ADR-015` Round 6): the editor refuses to commit a
bend that isn't currently aimed at a Diamond or Sub-Diamond. A **bend** with no anchor is a plain
free-placed point left over from before this invariant existed — it keeps working exactly as
before until someone repositions it, at which point the new position must be anchored like any
other bend commit.
**The Path's end point is exempt from this invariant** — it's wherever the ball actually ends up
(potted, come to rest, resting against another ball), not necessarily a rail contact the way a
bend is, so it can always be freely placed, aimed or not. This is a deliberate, permanent
exception, not a legacy-data allowance the way an unanchored bend is.
_Avoid_: assuming a free-placed *bend* can still be created going forward — that's only ever true
for data that predates `ADR-015` Round 6. A free-placed *end*, by contrast, is always valid.

**Cue Ball**:
The ball the current player is shooting/striking with, in a given Diagram. Always white or
yellow — the two player-owned ball colors — whichever the player in question is using; not a
fixed color.

**Object Ball**:
The ball the cue ball is initially directed at. In the practical 3-ball setup this is the
opponent's ball — always the *other* of white/yellow from whichever the Cue Ball is using, never
both the same color in one Diagram.

**Second Object Ball**:
The other ball on the table — not the cue ball's initial target — that may become involved
indirectly, via carom or combination, during the shot. Term borrowed from English-language
carom/three-cushion billiards terminology, chosen specifically to replace "pallino." Always red —
unlike Cue Ball/Object Ball, this role is not player-owned, so its color doesn't vary.
_Avoid_: "pallino" — colloquially just the Italian word for "ball" (as in "pallino bianco"),
not specific to this role, and ambiguous as a result.

**Annotation** (Diagram):
A short text note anchored to a specific point/ball/bend in a Diagram, meant to replace the
margin instructions found on scanned pages. Not yet structurally defined — deferred past the
initial engine milestone. The schema reserves room for it (an additive field), but the editor
does not support authoring it yet.
_Avoid_: assuming `Shot.description` alone covers this — some Diagrams are known to need
positioned, per-point notes that a single free-text field can't represent.

**Ball** (Diagram):
One entry in a Diagram's open list of balls. Has a role (Cue Ball / Object Ball / Second Object
Ball today, extendable later), an associated color, a start position, and an optional Path.
Not a fixed 3-tuple — the list can hold more or fewer than 3 if a shot needs it — but Italiana
and Goriziana are physically 3-ball games, so exactly one ball per role (Cue Ball, Object Ball,
Second Object Ball) is the practical default. Color follows role, not free choice: Second Object
Ball is always red; Cue Ball and Object Ball are each white or yellow, and must be the opposite
color from one another whenever both are present — see the role entries above for why (only Cue
Ball/Object Ball are player-owned, so only those two vary). Diameter is 61–61.5mm per the
official FIBiS regulation ("5 Birilli - 9 Birilli Goriziana - Tutti Doppi", in force from the
2022/23 season) — validated, not a guess.

**Layer** (Diagram):
One state of the table within a Diagram: a full collection of Balls with their Paths. A Diagram
holds an ordered list of them, and **the order is the chronology** — the first Layer is the table
before the shot, the last is where the balls come to rest, and anything between is a mid-shot state
(a carom passing through its first contact, for example). A Layer carries an optional `label` for
the editor's Layer switcher; it is display only and never competes with order as the source of
truth for sequence.
A Layer's Balls are ordinary Balls, so the same physical ball appearing in two Layers is represented
twice, independently — deliberately, since a starting ball and a resting ball are two marks doing
different jobs and a reader must tell them apart at a glance. There is no reference between them:
correspondence is carried by role and by position in the sequence. Nothing cross-validates one Layer
against another. See `ADR-017`.
_Avoid_: reading a Path's end point as "the ball's final position" — it is where one trajectory
stops, which is a different claim and cannot express a ball that finishes somewhere without a drawn
route. Also avoid treating Layers as a fixed start/end pair; the list is open precisely so
intermediate states fit.

**Hit** (Diagram):
How the Cue Ball is struck and which part of the Object Ball it takes — the close-up inset drawn
beside the table on instructional plates, showing two overlapping balls and a dot marking the cue
tip's strike point. Holds a **Ball Portion** and an optional **Spin**. At most one per Diagram, and
only where the Diagram has a Cue Ball, since an inset about striking the cue ball is meaningless
without one. It carries no table position: unlike a Ball, a Hit says nothing about where anything
sits on the Playing Surface, only how the two balls meet each other. Which colour is in front is
*derived* from the Diagram's Cue Ball, never stored on the Hit — so "bring to front" means swapping
which colour is the Cue Ball, not a display setting local to the inset. See `ADR-016`.
_Avoid_: "Contact" (taken by **Cushion Contact Point**, which is a computed geometric consequence of
a Path, not authored teaching content) and "Aiming" (taken by **Aiming Ray** / **Diamond Aiming**).

**Ball Portion** (Hit):
How much of the Object Ball the Cue Ball covers, and on which side — the fullness of the contact.
Measured in eighths of the ball's **width**: 8/8 is a full ball, 4/8 a half ball, 1/8 a thin edge.
Signed, where the sign is the side taken (negative left, positive right) from the shooter's view. A
portion of zero is invalid — no contact is not a shot — and at 8/8 the sign carries no meaning,
since a completely full hit has no side. Deliberately one-dimensional: both balls rest on the same
cloth, so their centres are always level and only the across-the-ball offset can vary. See
`ADR-016`.
_Avoid_: treating it as a 2-D offset between the two balls — a vertical displacement between them
depicts something that cannot physically occur.

**Spin** (Hit) — *effetto*:
Where the cue tip strikes the Cue Ball, as a 2-D offset from its centre, measured in eighths of the
ball's **radius** (not its width — Ball Portion is a span across the ball, Spin is a displacement out
from its centre; both are "eighths of the ball" as a player would say it, but anchored differently).
Constrained only by the ball's own circumference: there is **no miscue limit**, because a Diagram is
teaching content rather than a simulation. Optional, and its absence means "not specified" — never
"struck centre", which is a different and stronger claim. See `ADR-016`.

## Table geometry

Real-world dimensions and reference marks of the billiard table itself, as distinct from the
Diagram content placed on it (Balls/Paths). Sourced from the official FIBiS regulation
(`resources/Rgolamento_gioco_5birilli_goriziana.pdf`), not the original POC's guesses — where
the two disagreed, the regulation wins and the POC's numbers should be corrected accordingly.

**Playing Surface**:
The table's playable rectangle: 284cm × 142cm (±5mm tolerance). Confirmed identical for both
Italiana (cue-stick games) and Goriziana — same regulation table, different game rules on it.
No pockets on either — official rules moved to pocketless tables in 1983.
_Avoid_: "table" alone when precision matters — the full table (see Rail below) is larger than
just the playing surface.

**Rail**:
The outer, structural wood portion bordering the Playing Surface — what a player rests a cue or
chalk on, and where the Diamonds are embedded. Together with the Cushion (below), the two form
the raised border colloquially called "sponda" in the regulation; Rail specifically means the
wood part, not that whole assembly. 8cm wide (the 12.5–15cm regulation total for Rail + Cushion
combined, minus the Cushion's 5cm).
_Avoid_: "sponda" — keep the glossary in English like the rest of this file's terms. Also avoid
using "Rail" loosely for the combined Rail+Cushion assembly now that the two have distinct
names — say "Rail + Cushion" or "the border" when you mean both together.

**Cushion**:
The inner rubber rebound strip a ball actually bounces off ("sponda di gomma" in the regulation),
between the Rail and the Playing Surface. 5cm wide, with a rebound edge height of 37mm (±1mm).
The POC's `frameWidth`/`railWidth` split maps loosely onto Rail+Cushion vs. Cushion alone (frame ≈
Rail+Cushion combined, rail ≈ Cushion) but was numerically off (6cm where the regulation says 5cm)
— correct to the regulation's figures.
_Avoid_: "elastic strip"/"elastic rebound material" — earlier phrasing in this codebase before the
Rail/Cushion split was named; use Cushion.

**Diamond**:
A fixed reference mark on the outer edge of the Rail, spaced at intervals of 1/8 of the Playing
Surface's length (284cm ÷ 8 = 35.5cm), including the corners. This gives **9 Diamonds per long
side, 5 per short side**. A Diamond's *position* is physical and fixed and defines the axes of
the Diamond Coordinate System (below) — a Diamond's *number* under a given aiming system is a
separate, per-Diagram concern (see Numbering System).
_A Diamond's own position is not the same as where an aimed shot touches the Cushion_ — that's
the **Cushion Contact Point**: where the **Aiming Ray** (the straight line from a Path segment's
start point through the Diamond being sighted on) actually crosses the Cushion. The two coincide
only for a straight, zero-angle shot; for any other start position they diverge, since the
contact point depends on the ray's real geometry, not the Diamond's coordinate alone (e.g.
aiming at Diamond 20 from an off-center start can touch the Cushion at 19, 18, or another value
entirely, never 20). Diamond Aiming (`ADR-015`) computes this properly — an earlier, abandoned
version of that feature wrongly assumed the two were always equal.

**Diamond Coordinate System**:
What a Ball's position and a Path's points actually store in the Diagram JSON — **not**
centimeters. Deliberately chosen over raw cm because cm coordinates aren't reproducible by a
player on a real table without a ruler; counting Diamonds and estimating tenths between them is
how players actually locate positions.
Each Diamond interval (35.5cm) equals 10 units, giving continuous axes of **0–40 across the
short Rail and 0–80 across the long Rail** (a Diamond sits at every multiple of 10). Origin
(0, 0) is the bottom-left corner of the "quadrato inferiore" (the near/acchito square, where
play starts, per the regulation's own Tavola 1 diagram) — X grows across the short Rail, Y grows
up the long Rail, away from the player. Values are continuous, not restricted to whole tens —
this is the coordinate space a drag-to-position editor writes to, with Diamond marks simply
landing on round numbers within it. Example: (20, 40) is the exact center of the table.
_Avoid_: storing or persisting raw centimeter coordinates for Ball/Path positions — cm remain
the unit for the static Table geometry constants (Playing Surface, Rail, Diamond spacing) only.

**Pin** (Birillo):
A small cylindrical marker (25mm tall; 7mm diameter at top, 10mm at the widest point, 7mm base)
used in the birilli-scoring games (5/9 Birilli, Goriziana, Tutti Doppi). Arranged in a "Castello"
(rack) at 66mm inter-Pin spacing, positioned along the table's median line. In scope for the
engine as a visual element (rendered on the table) and, like Diamonds, some aiming systems use
Pins as numbered reference points too — same system-relative-numbering caveat applies.

**Numbering System**:
Deliberately built in two stages. **v1 (in scope for the initial engine milestone): ad hoc,
per-Diagram number labels** — a Diagram can attach a plain number to any Diamond/Pin it
references, entered by whoever authors that Diagram, with no reuse across Diagrams. This is what
makes a `NUMBERING`-topology Shot's Diagram actually convey a system at all, so it isn't deferred
entirely despite being simplified. A "system" in the sense a player means it (e.g. "the 50
diamond system") maps onto a single Diagram here — different systems are simply different
Diagrams, each with their own independently-entered numbers; there is no separate named-system
entity in v1. **Deferred:** a reusable, named scheme that multiple Diagrams could reference
instead of each re-entering the same numbers — real complexity, correctly pushed past the initial
milestone.

The numbered target is not limited to the 28 real, whole Diamonds — real systems commonly compute
fractional aim points (half- or tenth-Diamond) via subtraction, so a Sub-Diamond position is
numberable too (see Sub-Diamond). A number label's identity is its **position**, expressed in
Diamond Coordinate System units (not cm) for consistency with how Ball/Path positions already
persist — not an array index, since Diamond/Sub-Diamond positions are computed fresh each render
with no stable id otherwise.

In the editor, a label at a Sub-Diamond (fractional) position is only visible/editable while that
Sub-Diamond spacing layer is toggled on (the layer is how an author reaches that position to
click it — a whole-Diamond label has no such gating, since real Diamonds have no visibility
toggle). The read-only Diagram view (`fe`#22, Shot Detail) has no spacing-toggle UI at all, so it
auto-shows exactly the labels that exist, at whatever granularity, without needing the underlying
dot markers to be drawn.

_Consequence for persistence_: Diamond/Pin *positions* stay a static rendering constant (derived
from Table geometry, not stored per-Diagram) — but Diamond/Pin *number labels* are per-Diagram
data and must live in the Diagram JSON alongside Balls/Paths.

## Editor overlays

Optional visual aids for positioning Balls/Paths while authoring a Diagram — as distinct from
Table geometry (fixed, regulation-defined) and Diagram content (Balls/Paths/Numbering labels,
persisted). Overlays are **never persisted** — ephemeral editor session state only, reset each
time the editor opens. Both render as separate components projected into `BilliardTableComponent`
via its `<ng-content>` slot (it has no knowledge of either), reusing the Diamond Coordinate
System's spacing math (10 units = 1 Diamond interval) rather than centimeters.

**Grid** (Positioning Grid):
Lines spanning the Playing Surface interior — horizontal, vertical, or both — at one or more
simultaneous spacings, expressed in Diamond Coordinate System units (10 = a full Diamond
interval, 5 = half, 1 = a tenth; any value is valid, not a closed set of presets). Configured via
a `spacingUnits: number[]` list (e.g. `[10, 5]` shows diamond- and half-diamond-spaced lines
together) plus shared `showHorizontalLines`/`showVerticalLines` booleans applying to all active
spacings alike (not configurable per-spacing — revisit only if that proves genuinely limiting in
practice). Intended groundwork for a future snapping mechanism in the Ball/Path editor, though
snapping itself is not yet built.

**Sub-Diamond**:
An extra Diamond-styled dot marker along the Rail edges, at a finer spacing than the 28 real,
regulation-fixed Diamonds — e.g. a marker every half-Diamond or every tenth. Visually similar to
a real Diamond but **not** a physical, regulation-defined mark — a Sub-Diamond is a synthetic,
configurable, toggleable overlay, and the marker/overlay itself is never persisted (ephemeral
editor state, reset each time the editor opens, same as Grid). It is **not**, however, unrelated
to the Numbering System: real aiming systems routinely compute fractional aim points (e.g. a
subtraction landing on a half- or tenth-Diamond value), so a Sub-Diamond *position* is a
legitimate Numbering System target, not merely a drawing aid — see Numbering System. What's
persisted is the number label's value at that coordinate (Diagram content); the dot/overlay used
to click that position while authoring is not. Always rendered on all four edges together (no
long/short-rail split, unlike the Grid's horizontal/vertical split) since a real Diamond itself
has no such split either. A separate component from Grid, even though both are spacing-driven
overlays — a consumer may want Sub-Diamond markers without Grid lines, or vice versa.
_Avoid_: confusing with Diamond — a Diamond is a fixed, physical, regulation-defined mark; a
Sub-Diamond is a synthetic, configurable overlay used to reach positions between Diamonds. Also
avoid assuming Sub-Diamonds are irrelevant to Numbering just because the overlay itself isn't
persisted — the *label placed at* a Sub-Diamond position is real, persisted Diagram data.
