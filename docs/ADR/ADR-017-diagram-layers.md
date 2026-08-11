# ADR-017: Diagram Layers — representing a shot's start, end, and intermediate states

## 📌 Context

The Diagram engine was built to teach **systems**: static ball positions plus a Path showing the
Cue Ball's trajectory against a numbered Diamond/Pin scheme. That is enough for a system, where the
question is "where do I aim".

Teaching a **shot** needs more. Once the Cue Ball is struck and reaches the Object Ball, the balls
move: they finish somewhere else, and the routes that took them there are the point of the lesson.
Instructional plates draw this as extra balls — unfilled outlines at the resting positions — with
their own trajectories. A Diagram that can only draw one set of positions can show where a shot
begins but not how it ends.

`ADR-012` established the persisted shape and `ADR-013`/`ADR-014` extended it additively. This
change is not additive: it restructures where balls live.

Grilling corrected three assumptions along the way, and the corrections are the substance of this
record:

1. **A Path's end is not "the final position".** It is where one trajectory stops. Reading it as the
   ball's resting place seemed to make the whole request a rendering change — draw the end marker at
   ball size instead of `PATH_MARKER_RADIUS_CM` — but that conflates "this line ends here" with
   "this ball ends here", and cannot express a ball that finishes somewhere without a drawn route.
2. **Duplicating a ball across states is the point, not a defect.** The first design derived the end
   ball's role and colour from the start ball, on the principle that a fact should be stated once.
   That was wrong about what the fact is: a start ball and an end ball are two marks with different
   jobs, and a reader must tell them apart at a glance. Deriving appearance would render them
   identically, which is precisely the confusion to avoid.
3. **Two states is the wrong bound.** A carom passes through an intermediate state — after the first
   contact, before the second — and that state is worth drawing. A `start`/`end` pair cannot hold it.

## 💡 Decision

**A Diagram holds an ordered list of Layers, bumping `schemaVersion` to 5.** Each Layer is a full
collection of Balls with their Paths — the same structure the engine already edits. `balls` moves
inside a Layer.

```json
{
  "schemaVersion": 5,
  "speciality": "goriziana",
  "layers": [
    { "label": "Start",        "balls": [ /* id, role, color, position, path? */ ] },
    { "label": "End position", "balls": [ /* same shape */ ] }
  ],
  "diamondLabels": [ ... ],
  "pinLabels": [ ... ]
}
```

**Order is chronology, and the only source of truth for it.** The first Layer is the table before
the shot; the last is where the balls come to rest; anything between is a mid-shot state. An
explicit `kind` field was rejected: it can contradict the order — nothing would stop a Layer marked
`start` sitting at index 2 — and then a reader must decide which signal wins. `label` exists for the
editor's Layer switcher and carries no meaning beyond display.

**A Layer's Balls are real Balls, with their own role, colour and appearance.** The same physical
ball appearing in two Layers is represented twice, deliberately. There is **no cross-Layer
reference**: an earlier design had end Balls reference their starting Ball so role and colour could
be derived, which is exactly what must not happen. With appearance independent, the reference
carried only correspondence — thin, and able to dangle when a Ball is deleted — so it was dropped.
Correspondence is conveyed by role and by the Layer's place in the sequence.

**Nothing cross-validates Layers.** A Diagram can hold an end state its geometry does not support.
Accepted, consistent with `ADR-016`: a Diagram is authored teaching content, and the same latitude
already exists between Paths and Ball positions.

**One Layer type, not two similar ones.** Everything already built — Diamond Aiming (`ADR-015`),
click-to-draw (`ADR-010`), bend removal, the contextual menu (`ADR-011`), Ball roles — applies to
every Layer unchanged, and every future addition lands in all of them without being ported. That
guarantee is structural only because the Layers share a type; two types sharing a component today
would drift tomorrow.

**The editor gains an active Layer.** Layers overlap on one table — an end position may sit under
another Ball's start position — so pointer interaction must target one Layer while the others stay
visible but inert. This is part of the design, not an implementation detail: without it the editor
becomes ambiguous the first time two Balls coincide, and the ambiguity is silent.

**Per-Layer appearance is deliberately not specified here.** Outline versus fill, dashes, and any
customisation of how a Layer's Balls are drawn are their own requirement. This ADR only guarantees
the model leaves room for it, which a per-Layer structure does.

## 🔄 Consequences

**Positive:**
- A Diagram can teach a shot, not only a system: where it starts, where it finishes, and the routes
  between.
- Intermediate states cost nothing — a carom's middle state is another entry, not a schema change.
- Every existing editing capability applies to every Layer for free, and so will every future one.
- Per-Layer appearance becomes expressible without further structural change.

**Accepted gaps:**
- **This is the first genuinely breaking schema change.** v2, v3 and v4 were additive; moving `balls`
  into `layers` is not. It is free only because no Diagram has ever been persisted — verified, the
  `diagrams` table is empty — and that window closes as soon as the Diagram work in flight reaches
  real use.
- The same ball is represented independently per Layer, so its role or colour can differ between
  them with nothing objecting. This is the deliberate consequence of wanting the states to look
  different.
- "The setup is Layer 0" is a positional rule rather than a declared one. Chosen over a field that
  can contradict the order, but it is implicit and must be stated wherever Layers are read.

**Follow-up needed:**
- Per-Layer appearance (outline, fill, dashes) — its own requirement, unblocked by this. **Taken up
  by `ADR-018`**, which gives appearance an owner (the Ball component); persisting it is that ADR's
  own committed follow-up.
- **`ADR-012`'s migration story is still unwritten, and this is the third bump to defer it.** Every
  version since v1 has been free because nothing was persisted. The first bump after real data
  exists will not be, and this one restructures rather than appends — the migration it would have
  needed is exactly the one still owed.
- Sequencing against `ADR-016`: the Hit and this change both bump the version and both touch the
  same conversion pair. Whichever lands first takes the lower number; they should not be built in
  parallel.
