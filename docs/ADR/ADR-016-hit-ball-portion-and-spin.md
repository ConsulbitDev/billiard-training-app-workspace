# ADR-016: Hit — Ball Portion and Spin (Diagram `schemaVersion` 4)

## 📌 Context

Instructional plates in *Il fascino del biliardo* carry, alongside the table drawing, a small
close-up inset: two balls overlapping, drawn one in front of the other, with a dot marking where the
cue tip strikes. It encodes what the table drawing cannot — **which part of the object ball to take**
and **with what spin** — and without it a reader knows where the balls go but not how to play the
shot.

`ADR-005` established Diagram persistence as a versioned blob and `ADR-012` defined its shape,
noting as follow-up: "add them as new fields under a bumped `schemaVersion`, and only then design the
actual migration". `ADR-013` and `ADR-014` already followed that route for Diamond and Pin labels
(v2, v3). This does the same for v4.

Nothing in `CONTEXT.md` covered either fact — there was no term for spin, and none for the fullness
of a ball-to-ball contact. The vocabulary here is new.

**No migration is required.** At the time of writing, zero Diagrams have ever been persisted, so v4
is still a clean-slate change exactly as v1 was.

Grilling surfaced four things that were not obvious from the request:

1. **The inset is not table geometry.** The existing `balls` array places balls at Diamond
   Coordinate System positions on the Playing Surface. The two balls in the inset have no table
   position at all — the entire content is their relationship to each other. Reusing the same
   structure would have been a category error.
2. **The overlap is one-dimensional.** Both balls rest on the same cloth, so their centres are
   always level; only the across-the-ball offset can vary. A 2-D offset would let an author store a
   vertical displacement that cannot physically occur, and the read-only renderer would then be
   obliged to faithfully redraw a diagram that misstates the shot.
3. **"Eighths" lands differently on the two quantities.** Fullness is a span *across* the ball; spin
   is a displacement *out from its centre*. One shared unit would have made the spin grid so coarse
   that "a touch of left" and "plenty of left" collapse to the same stored value.
4. **The Diagram already knows which ball is the Cue Ball.** "Bring the cue ball to the front" is
   not a new fact, it is a restatement of one `CONTEXT.md` already defines.

## 💡 Decision

**A new optional `hit` field on the Diagram, bumping `schemaVersion` to 4.** The backend is
unaffected: it treats the Diagram as an opaque blob and reads only `schemaVersion` (`ADR-005`,
`be`#72), so this needs no migration, no column, and no API change.

**Schema (`ADR-012`, bumped to `schemaVersion: 4`):**
```json
{
  "schemaVersion": 4,
  "speciality": "goriziana",
  "balls": [...],
  "diamondLabels": [...],
  "pinLabels": [...],
  "hit": {
    "ballPortion": -4,
    "spin": {"x": -3, "y": 2}
  }
}
```

**`ballPortion` is a signed integer in eighths of the ball's width.** Its magnitude is how many
eighths of the Object Ball the Cue Ball covers — `8` a full ball, `4` a half ball, `1` a thin edge.
Its sign is which side is taken, negative for the left and positive for the right, from the
shooter's view. `0` is invalid: no contact is not a shot. At `8` the sign carries no meaning, since
a completely full hit has no side.

**`spin` is optional and 2-D, in eighths of the ball's radius**, offset from the Cue Ball's centre,
`{x, y}` with x positive to the right and y positive upward. Its only constraint is geometric — the
point stays within the ball. **There is deliberately no miscue limit.** This is a teaching diagram,
not a simulation; restricting the author to physically strikeable offsets would be enforcing a rule
the source material itself does not observe.

**An absent `spin` means "not specified", never "struck centre".** Those are different claims, and a
Diagram that silently asserted a centre-ball strike whenever the author declined to say would teach
something it never meant.

**Cue Ball identity is derived from the Diagram, never stored on the Hit.** `CONTEXT.md` already
defines the Cue Ball as the ball being struck in a given Diagram, and fixes that Cue Ball and Object
Ball are each white or yellow and must be opposite colours. The inset renders that; it does not
restate it. "Bring to front" is therefore not an inset-local control but a shortcut for swapping
which colour is the Cue Ball, which edits the Diagram's balls and is reflected on the table at the
same time. **A Diagram with no Cue Ball cannot have a Hit** — an inset about how the cue ball is
struck is meaningless without one.

**Exactly one Hit per Diagram.** Spin happens once per shot; fullness happens once per ball-to-ball
contact, so a carom's second contact has one too. Modelling a single Hit therefore declares that one
contact is worth describing. That matches every plate in the source and keeps the authoring gesture
to one widget. Multiplicity is deferred, not rejected — see the follow-up below.

**A presentational renderer, with an editor wrapping it.** This mirrors `DiagramEditorComponent`,
which owns interaction and delegates all drawing to the `diagram-*` components. Shot Detail gets the
renderer with no drag handling, and exactly one piece of code decides what a Hit looks like, so the
read-only view cannot drift from the editable one. A single component behind a `readonly` flag was
rejected: threading that flag through drag handlers and pointer state accumulates conditionals that
are exercised in one mode and rot in the other.

## 🔄 Consequences

**Positive:**
- The stored value is the fact, not a picture of it — "half ball on the right" rather than a pair of
  pixel offsets — so it stays legible, comparable, and renderable at any size.
- Zero backend work, and no migration, because the change lands entirely inside the opaque blob
  while nothing is yet persisted.
- One place decides which colour is the Cue Ball, so the table drawing and the inset cannot
  contradict each other.
- The renderer/editor split is the same shape `fe`#22 needs for read-only Diagram rendering, so Shot
  Detail ends up composing renderers uniformly rather than special-casing one feature.

**Accepted gaps:**
- A Diagram can carry a `hit` describing a contact its `balls` and `paths` do not support — nothing
  cross-validates the two. Accepted: the Diagram is authored teaching content, and the same latitude
  already exists between Paths and Ball positions.
- One Hit cannot describe a carom's second contact, which is a real limitation for exactly the class
  of shot this ADR's source plate belongs to (*angolo con propria a girare*).
- Eighths of the width for portion and eighths of the radius for spin is two units under one word.
  Defensible per quantity, but it must be stated wherever either is read or written, or someone will
  eventually assume they match.

**Follow-up needed:**
- If per-contact fullness is wanted, promote `hit` to carry a list of portions under a bumped
  `schemaVersion` — keeping `spin` singular, since a shot is struck once.
- Read-only rendering on Shot Detail is `fe`#22's, which already has to read `shot.diagram`.
- The first real migration story still does not exist. It becomes necessary the moment Diagrams are
  persisted in anger, and every version bump until then is borrowing against it.
