# ADR-019: Identity before authentication — Shot authorship, access, and the practice record

## 📌 Context

The Practice Session engine is the next feature, and it is the first thing this project builds that
**accumulates data the author would be upset to lose**. Everything persisted so far is either
re-derivable (the catalogue, imported from Google Sheets) or disposable (Diagrams, still test
fixtures — see `ADR-012`'s 2026-08-12 amendment). Practice history is neither: it accrues from the
first session and cannot be reconstructed.

That matters because a Practice Session belongs to somebody, and **this application has no concept of
a person.** There is no user, account, player or session entity anywhere in the backend, and no
Spring Security dependency. `ADR-003` foresaw exactly this, recording that auth, sessions and stats
"all depend on having a real user concept that doesn't exist yet".

The existing plan answers it badly. `be`#12 ("MVP Auth") states its success metric as *"Training plan
+ shots saved under correct user account"* — which makes Shots per-user, siloing a shared catalogue.
`ADR-003` already found that issue stale on V1/V2 scope; it is stale on the domain model too.

The forcing question is not "should we build login". It is: **which facts are being generated right
now that cannot be reconstructed later?** Login can arrive whenever a second human does. Provenance
cannot be backfilled from memory.

The project is a learning lab and a personal tool today, with a stated ambition — *"niche tool for
players and billiard halls"* — that implies multiple users, shared catalogues and, eventually,
subscription tiers. The model must not foreclose that, and must not be built for it yet.

## 💡 Decision

**Model identity now; authenticate later.** Identity is data and accrues. Authentication is a
mechanism that proves it and accrues nothing. Only the accruing half is urgent, and conflating the
two is what makes `be`#12 look like a prerequisite when it is not.

### Access is many-to-many; authorship is singular

Two different questions, deliberately answered by two different structures:

- **"Whose shot is this?"** — exactly one answer, always. A **singular author** on the Shot. It
  carries provenance and edit rights. Catalogue content is authored by the platform account; content
  drawn in the Diagram engine is authored by its author.
- **"Who may see this shot?"** — a set. An **access association**, granting to a user, a role, or a
  group. A hall subscribing to a beginner pack grants its members access through the group, not by
  copying shots.

Authorship could have been folded into the access association as a grant *type*. It is not, because a
`NOT NULL` author makes zero-author and two-author Shots impossible **by construction**, where a row
in a many-to-many table makes both representable and leaves the invariant to be defended in
application code forever.

Note what follows: **a Shot is not owned by the people who can see it.** A public Shot may have
thousands of grants and exactly one author. Access never confers the right to modify.

### The catalogue is shared; the record is personal

Shots, Books, Categories, Resources and Diagrams describe *the game* and are shared. A Practice
Session and everything in it describe *a person* and are theirs. The dividing line is **reference
content versus record content**, and ownership follows that line and nothing else.

This kills `be`#12's "shots under a user account" framing explicitly: per-user shot silos would force
each player in a hall to maintain a private duplicate of the same published shot, destroying the
shared-library value that makes the hall case worth serving at all.

### History outlives access

**A user must always be able to read their own practice history, and enough Shot information to make
it intelligible, regardless of whether they can still access the Shot itself.** Entitlement gates
discovery and access; it never gates the survival or readability of a record. A lapsed subscription
must not evaporate six months of practice history.

Mechanically this is a **foreign key plus a permitted projection**, not a copy:

- The **foreign key is the identity and the grouping key** for statistics, so renaming or
  re-categorising a Shot never fragments history into two groups.
- A **permitted projection** — a defined, minimal subset of Shot fields — is readable for any Shot the
  user has history against, as an explicit exception to entitlement.

A snapshot copied onto the session record was considered and rejected: see Alternatives.

### Records that are referenced are never destroyed

Soft-delete becomes an invariant rather than a convention, scoped by a rule with a reason attached:
**never hard-delete anything another record references historically; entities owned by a parent may
still be deleted outright.**

- **Shot, Category, Book — soft-delete.** Statistics group by Category; a Shot is a Session's
  subject; a Book is a Shot's provenance.
- **Comment, Resource — hard delete is fine.** Owned by a Shot, referenced by nothing outside it, and
  aggregated over by nothing. Making them immortal would add a `deleted_at IS NULL` to every query
  and buy nothing.

Enforcement is at the persistence layer, not by convention, **with exactly one deliberate exception**:
a single, narrowly-named privileged read path that sees soft-deleted rows, used by history and
nothing else. The default must be safe because no finder can be trusted to remember; the exception
must be singular because an exception scattered across a codebase stops being auditable.

### What is built now

Only what cannot be reconstructed later. Everything else waits.

**Now — groundwork, before any session data exists:**

1. Soft-delete as a real invariant on **Shot, Category, Book**, plus the one privileged read path.
2. An **app user** concept with a role, seeded with two rows: the platform account, and the author.
3. **A singular, non-null author on Shot**, with existing rows backfilled.
4. **One seam that answers "who is the current user"**, returning the seeded author — the single
   place authentication later replaces.

Then the Practice Session engine, with a non-null owner from its first migration.

**Deferred entirely:** the access association, principals, roles as grantees, groups, halls, packs,
plans, entitlement, Shot visibility, and all of authentication — login, passwords, tokens, Spring
Security.

The test applied to every item on that list: **an access decision made under a single user is
trivially recoverable** — one person, who can see everything — whereas authorship is being decided
continuously, every time a Shot is created, and is lost the moment it is not recorded.

## 🔄 Consequences

**Positive**

- **The Practice Session engine is unblocked without building auth.** The answer to "does this need
  authentication first" is no: it needs one column and a seeded table.
- Catalogue-versus-personal provenance stops being lost. Asked in a year, it would be archaeology —
  book-derived Shots are inferrable from `book_id`, but Shots drawn in the Diagram engine, the exact
  content that engine exists to produce, would be reconstructable only from memory.
- The hall/pack ambition stays reachable without being built. Grants attach later to a model that
  already knows who authored what.
- Soft-delete stops being a convention honoured by two code paths out of nine.
- `be`#12 gets a correct target instead of a misleading one.

**Negative / accepted**

- **History is a privileged read path**, deliberately able to see what nothing else can. That is a
  standing invitation to misuse, mitigated only by it being singular, named and documented.
- **Backfilling authorship on existing Shots is a judgement call**, made once, by hand. There is no
  rule that recovers it — which is the whole argument for not deferring it further.
- A non-null author on Shot is a schema change to a table the frontend already consumes, though the
  API contract need not expose it yet.
- **Soft-deleted Categories and Books remain visible to queries that ask for them explicitly**, so
  admin UIs must decide what to show. Not addressed here.

**Follow-up needed — deliberately unbuilt, recorded so the shape is not re-derived**

- **The polymorphic principal.** "User *or* role *or* group" in one association is either several
  nullable foreign keys with a check constraint, or a `principal_type`/`principal_id` pair with no
  referential integrity. Both have real costs; choose deliberately rather than discover.
- **A grant must record what conferred it.** The deciding scenario: a player belongs to a hall
  subscribed to the Beginner Pack *and* has bought that pack personally. **The hall's subscription
  lapses.** They must not lose access. A grant stored as `(shot, principal)` with no source cannot
  express which rows to revoke, so revocation would either strip access that was paid for or leave
  the hall's access alive after it stopped paying.
- **Visibility** on user-authored Shots — private, shared, public — when there is a second human to
  share with.
- **Erasure versus immortal records.** "Referenced rows are never destroyed" and a future right to
  be forgotten will collide if this becomes a commercial product. Not urgent, but the collision is
  structural and worth facing before, not after.
- **`ADR-003`'s V1/V2 knot is still untied.** It scoped delete-shot as V1 close-out; `ADR-004` then
  removed the delete UI pending authorization, so a V1 item waits on V2 auth while `ADR-003` says V2
  waits on V1. This ADR does not resolve that, but it weakens it: identity now exists without
  authentication, and a delete UI may need less than full auth to be defensible.

## ⚖ Alternatives

**Snapshot Shot fields onto the session record.** Copy name, category, type, topology and priority
into the practice record at the time of practice, so history never reads the Shot at all. Rejected
on two counts. It is **redundant given soft-delete** — the row never disappears, so the reference
never dangles. Worse, **it is actively wrong for aggregation**: rename a Category and history
fragments into two groups, one under each name, which breaks the statistics the feature exists to
produce. The foreign key must remain the grouping key. Note the honest caveat: the snapshot's real
job was never surviving *deletion* but surviving *unreachability*, and it would return if
soft-deleted rows were globally hidden with no exception — which is exactly why the privileged read
path exists instead.

**Authorship as a grant type in the access association.** One table, `(shot, principal, type)` with
`OWNER` alongside `VIEW`. Rejected: it makes zero-owner and two-owner Shots representable and moves a
structural invariant into application code. A `NOT NULL` column enforces it for free.

**Ownership on the Shot instead of an access association.** A single `owner_id` and nothing else.
Rejected: access is genuinely many-to-many. A shot is seen by many users, by whole roles, and by
groups such as a hall subscribing to a pack — and even general-catalogue shots are not visible to
everyone. One column cannot express that.

**Per-user shot catalogues** (`be`#12's implied model). Rejected: it duplicates published material per
person and destroys the shared library that makes the multi-user case valuable.

**Build authentication first.** Rejected as a false prerequisite. It would delay the first feature
that produces data worth keeping, in order to add a mechanism that protects nothing while there is
one user — and it does not answer the question that is actually urgent, which is provenance.

**Defer everything until a second user exists.** Rejected for the single reason this ADR turns on:
authorship is generated continuously and is unrecoverable, while access decisions under a single user
are trivially recoverable. Deferring the recoverable half is correct; deferring the unrecoverable
half is not.
