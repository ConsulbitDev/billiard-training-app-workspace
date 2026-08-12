# ADR-012: Diagram JSON schema (persisted shape)

## 📌 Context

`ADR-005` decided Diagram persistence is a single JSON column plus a `schemaVersion` field, with
the backend treating it as an opaque blob (`be`#72) — but never defined the actual JSON shape.
With `fe`#19/#20/#41 shipped (Ball placement, Path drawing, contextual menu), and `fe`#21
(persistence) next, this is the first time the shape actually needs to exist. Nothing has been
persisted yet, so there's no legacy data and no migration burden — this is a clean-slate design.

## 💡 Decision

**v1 covers only what's built: `speciality` + `balls`.** No placeholder fields for Numbering
System labels (`fe`#24) or Annotations — both are deliberately unbuilt today (see `ADR-006`/
`ADR-005`). When they ship, they become new fields under a bumped `schemaVersion`, which is
exactly what that field exists for — reserving empty slots now would be designing for a
hypothetical the versioning mechanism already handles.

**Shape:**
```json
{
  "schemaVersion": 1,
  "speciality": "goriziana",
  "balls": [
    {
      "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "role": "cueBall",
      "color": "white",
      "position": {"x": 10, "y": 20},
      "path": {
        "points": [{"x": 15, "y": 22}, {"x": 25, "y": 30}]
      }
    }
  ]
}
```

**All coordinates (`position`, and every point in `path.points`) are Diamond Coordinate System
units** (0–40 across the short rail, 0–80 across the long rail — `ADR-006`), never centimeters or
pixels. This must be stated explicitly here because a bare `{x, y}` number pair carries no unit
of its own — the existing TS `Point` type is reused verbatim, but the JSON alone doesn't convey
that without this record.

**`path.points` is a single ordered array — the whole polyline after the ball's own position, not
split into `bendPoints`/`endPosition`.** The last element is the path's end; everything before it
is a bend. This was a direct correction during grilling: the original proposal mirrored the FE's
internal field split (`bendPoints: Point[]`, `endPosition: Point`), which reads as "a bend list
plus a special end point" rather than what it actually is — a polyline. The wire format optimizes
for being unambiguous polyline data; the FE's internal editor model is a separate concern (below).
A ball with no Path omits the `path` key entirely (not `path: null`), matching the TS `path?:
Path` optionality already in place.

**The FE's internal editor model (`DiagramPathsComponent`, `path.ts`, the sandbox) keeps its
existing `bendPoints`/`endPosition` shape — no refactor of already-shipped, tested drag/
context-menu code.** A small conversion pair (`toDiagramJson`/`fromDiagramJson`) sits at the
save/load boundary, mapping `{bendPoints, endPosition} ↔ {points}`. The internal shape optimizes
for the UI mechanics already built around "the end is special" (its own draggable marker, its own
context-menu entry); the wire shape optimizes for being clean, canonical polyline data. Paying a
small, well-named conversion at the one boundary that needs it is cheaper than reshaping working
code for no functional gain.

**Ball IDs switch to `crypto.randomUUID()`, replacing the current module-level sequential counter
(`ball-0`, `ball-1`, ...).** The counter resets to 0 on every page load, which was harmless while
Balls only ever lived in transient in-memory editor state (`ADR-008`) but actively breaks once
Diagrams persist and reload: loading a saved Diagram's `ball-0`/`ball-1`/`ball-2`, then adding a
new ball in a *later* session (fresh page load, counter reset) would collide with already-persisted
IDs. `crypto.randomUUID()` needs no cross-session coordination.

**Loading enforces `schemaVersion === 1` exactly — an unrecognized version fails loudly (throws/
shown as an error), not a best-effort parse.** No migration logic exists yet because there's
nothing to migrate from; this is purely about what happens if a newer schema version is ever
encountered (e.g., mid-rollout). Silently misinterpreting a future field layout is worse than a
clear failure.

**`speciality` is required on every Diagram, with no default.** It determines the Castello/Pin
rendering (`ADR-006`) — not a cosmetic default worth guessing. Creating a new Shot's Diagram means
explicitly picking Italiana or Goriziana as part of setup. The sandbox's `signal<Speciality>
('goriziana')` default is dev-harness convenience only, not a statement about real defaults.

## 🔄 Consequences

**Positive:**
- Wire format is unambiguous, canonical polyline data — no reader needs to know an
  editor-mechanics backstory to understand `path.points`.
- Zero refactor cost to already-shipped `fe`#19/#20/#41 code — the internal/wire split contains
  the new shape to exactly one conversion boundary.
- Ball ID collisions across sessions become structurally impossible before they ever had a chance
  to cause a real data problem.

**Accepted gaps:**
- A `toDiagramJson`/`fromDiagramJson` conversion layer is genuine extra code that a fully-unified
  internal/wire shape wouldn't need — accepted specifically to avoid touching working drag/
  context-menu logic.
- No migration path exists for a future `schemaVersion` bump yet — deliberately deferred until a
  real v2 shape is known, per `ADR-005`'s versioned-blob approach.

**Follow-up needed:**
- When Numbering System labels (`fe`#24) or Annotations are built, add them as new fields under a
  bumped `schemaVersion`, and only then design the actual migration/back-compat handling for
  existing persisted Diagrams.
- Define the new-Shot-creation flow's speciality picker (not yet designed) — this ADR only
  establishes that the field is required, not the UI for setting it.

---

## 📎 Amendment (2026-08-12): the version is frozen while the engine is pre-production

**`schemaVersion` stays at 6 and does not move for further wire-format changes, until the Diagram
engine carries real data.** This amends the cadence of incrementing it, and nothing else.

### Why

The follow-up above says to design migration handling "for existing persisted Diagrams". There have
never been any. The version was bumped six times anyway — v2 for Diamond labels (`ADR-013`), v3 for
Pin labels (`ADR-014`), v4 for the Hit (`ADR-016`), v5 for Layers (`ADR-017`), v6 for Ball
appearance (`fe`#84) — and **every one of those numbers describes an empty population**. No document
has ever been written in any of them. `fe`#84 verified this rather than assuming it: the
`shots.diagram` column arrived with the backend's `V9` migration on 2026-08-07, there is no
deployment, and the development database predates the column.

The cost of the ritual is not the bumping. It is that six minted version numbers imply a five-link
migration chain, and a reader — or an agent — encountering `DIAGRAM_SCHEMA_VERSION = 6` alongside an
undesigned migration story reasonably concludes that a debt has been dodged five times and is now
overdue. It has not been dodged. **The trigger condition was never met.** Writing that chain would
produce code whose every branch is unreachable, to convert documents that do not exist.

There is a second, worse property of waiting for the deferred trigger to fire on its own: the
condition it waits for — the first real saved Diagram — is the same moment that makes the work
unavoidable. There is no interval in which you are told you need it while it is still free. Hence a
named trigger below rather than a condition to be noticed later.

### The policy

- **During development the Diagram wire format is unstable and effectively unversioned.** Change the
  shape freely, in either direction, additive or not. Do not bump.
- **The number stays 6.** Not reset to 1: resetting costs a code change, makes this ADR's documented
  v1 shape disagree with what the code would then write as v1, and buys nothing. Only the value that
  real data is first written with matters, and 6 is as good a starting number as any. A `0` or
  SNAPSHOT-style sentinel is not available — the backend's `DiagramJsonMapper` requires a positive
  integer, and changing that to express a temporary development state is not worth a backend release.
- **Everything else in this ADR stands.** The field remains required. Loading still enforces the
  version exactly and still fails loudly on a value it does not recognise. The mechanism is intact
  and untouched; only the cadence of incrementing it changes.

### The trigger

**When the scanned images start being ported into Diagrams, this policy ends.** That is the point at
which Diagrams stop being test fixtures and become content someone would be upset to lose.

At that moment: freeze whatever shape is current as the first real version, bump it deliberately,
and from then on **every** wire-format change bumps — and the first such bump is the one that must
carry the migration handling this ADR originally deferred. That migration is owed to real documents,
not to v1 through v6.

Until then the only Diagrams that exist are the author's own tests, which are disposable by
definition.

### Accepted cost

**A stale local Diagram now misparses silently instead of failing loudly.** If a shape changes
without a bump, a document saved before the change still declares version 6 and will be read as
though it matched. During development that is the right trade — the remedy is to discard the
document and redraw it, which is what test data is for.

**It is precisely the wrong trade after the trigger fires**, and that asymmetry is the whole reason
the trigger is written down here rather than left to judgement.
