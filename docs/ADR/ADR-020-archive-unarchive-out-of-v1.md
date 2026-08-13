# ADR-020: Drop archive/unarchive from V1

## 📌 Context

`ADR-003` deferred archive/unarchive out of the V1 close-out pass while explicitly keeping it inside
the milestone — *"a real V1 scope item, just not part of this close-out pass."* Everything else it
named has since shipped. Archive/unarchive is now **the only undelivered V1 item, and the only one
with no ticket**, because it cannot be ticketed: nothing anywhere states what it would do.

### The requirement has no requirement behind it

`ADR-003` attributes it to *"the `status: active/archived` concept from the PRD's original data
model"*. **`PRD.md` contains no such field.** It has no occurrence of "archive", "archived" or
"status" at all. Whatever original data model that sentence referred to is not the one in the repo.

What survives is a phrase, in two kinds of place:

- Four derived documents repeating the bullet *"Shots & Systems CRUD — list, filter, view, edit,
  archive"*: both READMEs and the frontend's `CLAUDE.md`/`AGENTS.md`/`GEMINI.md` trio, which are
  copies of one another. One line, propagated five times, sourced from nothing.
- `CONTEXT.md`'s own entry, whose entire body is *"Not yet defined. Deferred to the milestone after
  this one."*

So there is no statement of what archiving hides a Shot from, who it is hidden from, what
un-archiving restores, or how any of it differs from what the app already does. There is a name and
a promise to define it later.

### Everything it might have been for already has an owner

- **Taking a Shot out of the catalogue** — soft-delete, enforced at the persistence layer since
  `be`#81, with the row preserved for practice history (`ADR-019`).
- **"I am not working on this right now"** — `Priority` already carries `NOT_NEEDED`,
  `NICE_TO_HAVE`, `INTERESTED`, `MUST_HAVE`. This is the archive use case, already modelled, already
  filterable, and already the field the catalogue was imported with.
- **Finding things in a large catalogue** — search and filtering, in place since the start of V1.

### And the cost of building it anyway is not nothing

It puts a **second lifecycle field on `Shot`** whose meaning nobody has written down. Every read path
then has to decide whether to respect it, and every future one inherits that decision — the exact
class of problem `be`#81 was spent removing, where exclusion was honoured by two finders out of nine.
`CONTEXT.md` already warns the two will be confused: *"avoid using 'archived' or 'soft-delete'
interchangeably."* A field that needs a warning before it exists is a field to be sure about.

The volume argument settles it. There is one user, a catalogue imported from a spreadsheet, and
nothing cluttered enough that filtering fails to find it.

## 💡 Decision

**Archive/unarchive is out of V1** — removed from scope, not deferred to a later milestone. A
deferral implies a commitment with a date attached, and there is no design to commit to.

V1 therefore closes on four tickets, all of them mechanical and all of them written:

- `fe`#8 — delete-shot (unblocked; see `ADR-004` and the `be`#81/`be`#85 work that superseded it)
- `fe`#101 — Categories admin CRUD
- `fe`#102 — Books admin CRUD
- `fe`#107 — resource retry duplication

**Soft-delete does not absorb archiving.** `ADR-003` kept them as deliberately separate fields so
this decision would stay open; dropping archive does not merge them. It stops reserving design space
for something that may never exist, and it leaves soft-delete meaning exactly what it means today.

### The named trigger

Following `ADR-012`'s freeze, this is reopened by a specific event rather than by a date: **a second
person using the app, or a catalogue large enough that filtering stops finding things.** Until one of
those happens, an archived state is a field with no user. If it is reopened, it starts from a stated
need — which is more than it has ever had.

## 🔄 Consequences

**Positive**:
- V1 closes on written, estimable tickets with no open design question in front of them.
- Removes the only V1 item that could not be assigned or finished, and which had therefore blocked
  the milestone indefinitely without anyone deciding to block it.
- Keeps a second, undefined lifecycle field off `Shot`. The asymmetry matters: adding a status field
  later is a migration, while removing one that queries have come to depend on is not.
- Five documents stop advertising a feature that does not exist and is not coming. Agents read those
  documents as instruction.

**Accepted costs**:
- If archiving is wanted later it begins from nothing. This is a smaller loss than it sounds — it
  begins from nothing today too. *"Not yet defined"* was never a head start.
- Someone may eventually want the distinction between "deleted" and "put away", and will have to
  argue for it. That argument is the point.

**Follow-up needed**:
- `CONTEXT.md`'s **Archived** entry records this decision instead of promising a milestone. The
  glossary keeps the term, because the *warning* against conflating it with soft-delete is still
  worth having.
- The *"list, filter, view, edit, archive"* bullet is corrected in `billiard-training-app-be/README.md`,
  `billiard-training-app-fe/README.md`, and the frontend's `CLAUDE.md`/`AGENTS.md`/`GEMINI.md`.
- `ADR-003`'s deferral list is superseded on this point. Its text is left as written — the deferral
  was correct when made — and this ADR is the amendment.
- Two `ADR-003` follow-ups remain open and are deliberately **not** addressed here, since both wait on
  V2 scope actually being kicked off: `be`#12 ("MVP Auth") still implies V1 scope it does not have,
  and the frontend's "Project Focus (MVP)" section still lists Practice Sessions and Basic Stats,
  contradicting the PRD.
