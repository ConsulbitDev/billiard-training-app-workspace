# ADR-021: V2 is consultation on mobile — and Authoring stays desktop-shaped

## 📌 Context

The app is used on a phone and a tablet. It was built on a desktop, and it shows: the layout is
usable at 1280px and awkward at 390px. That is not a cosmetic complaint — **the device the app is
actually used on is the one it serves worst.**

`ADR-003` set the milestone order as V1 (Knowledge Base Explorer + Admin) then V2 (real
Authentication, Training Sessions, Statistics). That order assumed the thing already built was
pleasant to use. It is not, on the device it is used on, and building Practice Sessions on top of a
layout that does not fit the screen means building them twice — session logging is the most
mobile-bound feature in the product, done at the table with a phone in hand between shots.

Three parked tickets already say so out loud. `fe`#67 and `fe`#68 carry a `blocked` label whose
stated reason is the coming layout rework, and `fe`#93 sits behind them. The rework is not a detour
from the roadmap; it is what part of the roadmap is queued on.

### What was already true, and what was actually broken

A survey before deciding anything found the app is **not** a desktop-only application needing a
rebuild. The shell already has a hamburger and a `p-drawer`; the viewport meta is correct; the shot
list is a responsive card grid rather than a table. Breakpoint coverage is thin but real — 19 usages
in `shot-detail`, 11 in `shot-list`, 6 in `system-form`, and **zero** in `resource-viewer`,
`diagram-view`, `shot-comments` and `shot-resources-editor`.

What is broken is more specific, and more serious, than "the layout is not responsive":

- **A zoomed scanned image cannot be panned on a touch device at all.** `resource-viewer` gates
  panning behind the space bar (`@if (isSpacePressed)`) and then rejects any pointer that is not a
  mouse (`event.pointerType !== 'mouse'`). Both conditions are unsatisfiable on a phone. Zoom into a
  scanned page and you are stuck looking at the middle of it. This is the primary use case, blocked
  by a keyboard-and-mouse interaction, and no amount of responsive CSS reaches it.
- **The first screen of the library is filters, not shots.** Seven controls stack into one column
  below `sm:`, costing roughly 545px before the first result on a 667px viewport.
- **The first screen of a Shot is metadata, not content.** Four `dt`/`dd` pairs stack to ~320px, and
  the Risorse panel — already `[value]="['resources']"`, already the intended focus — renders fourth.
- **The app opens on a launcher.** `/home` offers two cards routing to destinations already one tap
  away in the drawer, one of which is an authoring action never performed on mobile.
- **The shell is called `DashboardComponent`**, and a `{path: "dashboard"}` route points at it — so
  `/dashboard` renders a second complete shell, with its own toolbar and drawer, inside the first
  one's `<router-outlet>`. Nothing links to it, so it has never been noticed.

## 💡 Decision

### 1. The milestones renumber

**V2 is consultation on mobile.** Authentication, Training Sessions and Statistics — `ADR-003`'s V2 —
become **V3**.

`ADR-003`'s ordering argument is untouched by this: it insisted V1 close before that work began, and
V1 does. This inserts a milestone; it does not reopen one.

### 2. The scope rule: Consultation and Authoring

> **Every screen is either Consultation or Authoring. Consultation works everywhere. Authoring is
> desktop-shaped.**

| Consultation | Authoring |
|---|---|
| Shot list and its filters | Shot form (add / edit) |
| Shot Detail | Diagram editor, Hit editor |
| Resource viewer | Comment composer, Resource editor |
| Diagram view (read-only) | Categories / Books admin |

The rule is **descriptive, not enforced.** There is no viewport check anywhere in the code, no
device-conditional capability, and nothing is hidden or disabled on a small screen. Every route stays
reachable and every action stays available. "Read-only on mobile" describes what a person will *do*
there, never what the app *permits*.

Enforcement was considered and rejected. Any breakpoint is a guess about intent that the viewport
cannot support — a 1024px tablet with a keyboard is a genuine authoring device, a small laptop window
is not a phone — and the first time the guess is wrong, it is wrong by locking the author out of
fixing a typo on their own tablet. It would also introduce a device-conditional capability that
`CONTEXT.md` would have to name and every authoring component would have to consult, permanently.

The rule earns its place by deciding scope questions without a device check: a screen's category says
whether a given fix is in this milestone. It survives the awkward case — Shot Detail is Consultation
and contains an Authoring comment composer; the composer renders and works, it simply does not get
responsive attention.

### 3. The phone is the only new target

**Below 640px** is the new target. Tablets inherit today's layout.

`sm:` (640px) is already the de-facto line in this codebase — 19 usages, driving both the filter grid
and the card grid, and swapping the add-shot button between icon and label. There is no Tailwind
config, so these are the defaults. Adopting the existing line costs nothing and contradicts nothing.

A tablet at 768–1024px gets two-column filters and two-column cards. That is not broken, only
untuned, and untuned is an acceptable state under the rule above. A tablet is also the device propped
up while practising, where seeing more at once beats a phone-optimised single column — giving it the
phone layout would be a downgrade.

A phone in **landscape** (844×390) clears 640px and gets the desktop layout with 390px of height,
where `h-[70vh]` and `min-h-80` fight each other. Verified and patched where egregious; not designed
for.

### 4. What this milestone changes

- **Touch gestures in the image viewer** — pinch-to-zoom and direct drag-to-pan, removing the
  space-key gate and the mouse-only rejection. Direct panning is better on desktop too: the current
  modifier is undiscoverable. **PDFs are the exception** and delegate to Drive, which handles them
  properly and is not the primary content.
- **Filters behind a sheet below `sm:`**, search staying inline, with an **active-filter count on the
  button**. The count is not optional: hidden filters that silently constrain the list are how a
  person lands on "Nessun tiro corrisponde ai filtri" and concludes the catalogue is broken.
  `hasActiveFilters()` already exists.
- **Shot Detail** — metadata becomes wrapping chips below `sm:` (~320px to ~40px); the Risorse panel
  moves to the top of the accordion **for all viewports**, because a panel that opens by default and
  renders fourth is a small lie about what the page is for.
- **The app opens on `/shot/list`.** `/home` stays as a route for V3 to grow into, when Practice
  Sessions give it something to hold.
- **The shell is renamed off `DashboardComponent`** and the `/dashboard` route is deleted.
- **`UI_GUIDELINES.md`** already mandates mobile-first responsive layouts and explicit overflow
  behaviour. This milestone makes that true rather than aspirational.

### 5. Read-only Diagrams render, but get no gestures yet

`billiard-table` is already `width: 100%; height: auto` with `touch-action: none`, and its canvas is
portrait — which suits a phone better than a desktop. Panning is gated only on `event.button !== 0`,
which touch satisfies. **Only pinch-zoom is genuinely missing**, since zoom is wheel-only.

The diagram panel is verified to render correctly and fixed if it does not. Pinch-zoom waits, and it
waits behind **`ADR-012`'s existing trigger** rather than a new one: *when scanned images start being
ported into Diagrams.* That is the same event, for the same reason — it is when real Diagrams first
exist to consult. No Diagram has ever been persisted, so designing gestures now means designing
against imagined content, which is the argument `ADR-012`'s freeze already made and won.

### 6. V1 closes on two tickets, not four

`ADR-020` recorded V1 closing on `fe`#8, #101, #102 and #107. **V1 closes on `fe`#8 and `fe`#107**;
`fe`#101 and `fe`#102 (Categories and Books admin screens) follow V2.

The deciding factor is which pain is real. The mobile pain is live and daily. Categories and Books
are slow-changing reference data seeded by the Sheets import, and the project has run its whole life
without an admin screen for either without it blocking anything. #8 and #107 are small and
layout-independent, so V1 gets a real close-out rather than an abandoned one — and #107 is a bug that
silently writes duplicate data, which should not sit through a milestone about making the app
pleasant.

Building them after V2 also means they are built with the patterns this milestone establishes,
instead of being the last two screens built the old way.

`ADR-020` is **not** amended. Its subject is archive/unarchive and it was correct when written; this
is the record of the re-sequencing.

## 🔄 Consequences

**Positive**:
- The app becomes good on the device it is actually used on, before more is built on top of a layout
  that does not fit it.
- Unblocks `fe`#67, #68 and #93, which are parked on precisely this rework.
- The Consultation/Authoring split gives every future screen a scope answer without a device check,
  and without a concept the code has to carry at runtime.
- One survey replaced an assumption: the app was not desktop-only, and the real defects — an
  un-pannable image, a screen of filters, a nested shell — were specific and findable.

**Accepted costs**:
- Authoring on a phone stays awkward, by choice. If that becomes painful, it is a decision to revisit
  with evidence, not a gap to fill pre-emptively.
- Three documents are wrong the moment this is decided. They are corrected in the same change rather
  than as follow-ups — see below.

**Follow-up needed**:
- `MICROSERVICE_ARCHITECTURE.md`'s "V2+" scoping becomes V3+, in this change.
- The frontend's `CLAUDE.md`/`AGENTS.md`/`GEMINI.md` "Project Focus (MVP)" section stops listing
  Practice Sessions and Basic Stats as current, in this change. This finally discharges an `ADR-003`
  follow-up that has been open since that ADR was written.
- `be`#12 ("MVP Auth") is re-labelled V3, discharging the other `ADR-003` follow-up.
- `CONTEXT.md` gains **Consultation** and **Authoring**, and reserves **Dashboard** for the V3
  statistics screen so the next person to reach for the word finds it defined rather than merely
  taken.

**Deliberately left open**:
- The image viewer's **250% zoom cap** may be too low to read fine print on a scanned page at phone
  width. That needs a real scan looked at on a real phone — an observation, not a decision that can
  be reasoned to.
- Whether V2 ends with the **code-implementation retrospective** agreed during the speed phase. This
  milestone is what makes the app usable, which was the stated trigger.
