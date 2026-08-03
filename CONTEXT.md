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
(sits in place, e.g. as an obstacle) — Paths are optional per ball, not mandatory.

**Cue Ball**:
The ball the current player is shooting/striking with, in a given Diagram.

**Object Ball**:
The ball the cue ball is initially directed at.

**Second Object Ball**:
The other ball on the table — not the cue ball's initial target — that may become involved
indirectly, via carom or combination, during the shot. Term borrowed from English-language
carom/three-cushion billiards terminology, chosen specifically to replace "pallino."
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
and Goriziana are physically 3-ball games, so 3 (white cue, yellow cue, red object) is the
practical default. Diameter is 61–61.5mm per the official FIBiS regulation ("5 Birilli - 9
Birilli Goriziana - Tutti Doppi", in force from the 2022/23 season) — validated, not a guess.

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
The cushion bordering the playing surface ("sponda" in the regulation). 12.5–15cm total
horizontal width, of which 5cm is the elastic rebound material ("sponda di gomma"), with a
rebound edge height of 37mm (±1mm). The POC's `frameWidth`/`railWidth` split maps loosely onto
this (frame ≈ the Rail's total width, rail ≈ the elastic rebound strip) but was numerically off
(6cm where the regulation says 5cm for the elastic strip) — correct to the regulation's figures.
_Avoid_: "sponda" — keep the glossary in English like the rest of this file's terms; use Rail.

**Diamond**:
A fixed reference mark on the outer edge of the Rail, spaced at intervals of 1/8 of the Playing
Surface's length (284cm ÷ 8 = 35.5cm), including the corners. This gives **9 Diamonds per long
side, 5 per short side**. A Diamond's *position* is physical and fixed and defines the axes of
the Diamond Coordinate System (below) — a Diamond's *number* under a given aiming system is a
separate, per-Diagram concern (see Numbering System).

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
entirely despite being simplified. **Deferred:** a reusable, named scheme (e.g. "the 3-diamond
system") that multiple Diagrams could reference instead of each re-entering the same numbers —
real complexity, correctly pushed past the initial milestone.
_Consequence for persistence_: Diamond/Pin *positions* stay a static rendering constant (derived
from Table geometry, not stored per-Diagram) — but Diamond/Pin *number labels* are per-Diagram
data and must live in the Diagram JSON alongside Balls/Paths.
