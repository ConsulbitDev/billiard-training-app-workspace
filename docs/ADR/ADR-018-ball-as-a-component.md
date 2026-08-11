# ADR-018: The Ball is a component, not an SVG circle in someone else's template

## 📌 Context

A Ball is currently a `<circle>` tag written directly into whichever component happens to be
drawing. It appears twice: in `DiagramBallsComponent` (on the table, in centimetres) and in
`DiagramHitComponent` (in the inset, in ball radii). Everything a Ball *is* visually — its fill, its
outline, whether it offers a right-click, what decorations sit on it — is therefore a property of
its container, and every container that draws a Ball re-states it.

The cost is already visible rather than theoretical:

- The Hit inset clips its spin mark to the cue ball with a `clipPath`, and needs a per-instance id
  because SVG ids are document-global. That machinery exists in the inset only because the inset
  draws a mark on a ball it does not model.
- Stroke width is declared twice, in two units — `0.15` cm on the table, `0.05` radii in the inset —
  which are the same proportion expressed unrelatedly. Nothing keeps them in step.
- `ADR-017` deferred per-Layer appearance (outline, fill, dashes) as "its own requirement". There is
  nowhere for that requirement to land: appearance has no owner.

Grilling settled the shape, and corrected the request in two places:

1. **"A context parameter" would have undone the change it was part of.** The requirement was
   described as a Ball behaving differently by context — a contextual menu on the table but not in
   the inset, a spin mark in the inset but not on the table. Expressed as `context: 'table' |
   'inset'`, each new usage adds a branch inside the Ball, each branch is exercised in one situation
   and rots in the others, and the Ball ends up knowing about its callers. Expressed as
   capabilities, the same two examples are a flag and a decoration.
2. **"The Ball owns its contextual menu" splits in two.** The menu's *items* are domain actions —
   start a Path, change which colour is the Cue Ball, delete this Ball — each needing knowledge the
   Ball is specifically meant not to have.

## 💡 Decision

**One component renders one Ball.** It uses an **attribute selector** applied to a real SVG element
(`<svg:g appBall …>`), not an element selector. This is not stylistic: an element-selector component
inside an SVG silently fails to paint — the SVG renderer refuses to lay out anything inside an
unrecognised element, with no console error and a structurally correct DOM. See
`BilliardTableComponent`'s doc comment for the empirical finding and `DiagramGridComponent` for the
worked example.

**The Ball draws a unit circle and carries its own scale.** The caller passes a centre and a radius
in the caller's own units; the component sets `transform="translate(cx cy) scale(r)"` on its host
`<g>` and draws everything internally against `r = 1`. Every internal length — stroke, axes, eighth
guides — is written once, in radii, and never converted. This is what makes the eighth-portion guides
literals (`±0.25`, `±0.5`, `±0.75`) rather than products, and it is why a spin offset, already
measured in eighths of the radius (`ADR-016`), needs no conversion at all: `{x: 4, y: 0}` is
`x = 0.5` in Ball-local space.

Note the consequence: **scaling the host scales the stroke**, which is wanted — a ball drawn small
should have a proportionally thin outline. A stroke that must stay constant regardless of size would
need `vector-effect="non-scaling-stroke"`, which is then a per-appearance choice rather than a
property of this approach.

**Configuration is by capability, never by identity.** The Ball is told *what to do*, never *where it
is*. A `context` enum naming the situation was explicitly rejected for the reason in the Context
section above. The two motivating examples land as:

| Requirement | Expressed as |
|---|---|
| Context menu to start a Path, on the table only | `interactive` capability + the container's menu |
| Spin mark, in the inset only | a Ball-local decoration at a point in radii |

A third caller wanting a menu without a mark, or a mark without a menu, needs no change to the Ball —
including one neither of today's two callers anticipates, such as a Shot Detail inset showing where
the tip struck.

**Extension has three tiers, in increasing openness:** named appearance properties for the common
cases; a guides configuration for the built-in decorations (axes, eighth-portion marks); and
**content projection into the Ball's local coordinate space** for anything nobody has thought of yet.
Because the host is already scaled so `r = 1`, projected SVG is positioned in radii and works at any
drawn size. `BilliardTableComponent` already takes overlays this way (`ADR-007`), so this is an
existing mechanism applied one level down, not a new one.

**The Ball owns what needs only itself; the container owns what needs the drawing.**

| The Ball owns | The container owns |
|---|---|
| Its appearance — fill, outline, dashes | Which appearance to use, and why |
| Its own drag gesture and `dragging` state | What a moved Ball means to the Diagram |
| *Offering* a context menu | The menu instance and its items |
| Decorations positioned in its own space | The data those decorations show |
| Clipping its own decorations to itself | — |

**A Ball owns its own pointer gesture**, with an `interactive` capability. `ADR-016` rejected a
`readonly` flag on a drawing component, and that decision stands for the case it was made about — see
the amendment below for why this is a different case and not a reversal.

**Appearance is an input, and for now the container computes it. Nothing is persisted yet.** The
component's contract is identical either way: it receives an appearance and draws it. Only the source
changes. This keeps a structural refactor free of a schema change — but see the follow-up, which is
committed work, not an option.

## 🔄 Consequences

**Positive:**
- Appearance finally has an owner, which is what `ADR-017`'s deferred per-Layer appearance was
  waiting for.
- `fe`#67 gets "visible but inert" and "visually distinguishable" for free — both are per-Ball
  capabilities — where otherwise it would have invented per-set interactivity and per-set appearance
  inside `DiagramBallsComponent` and had them moved afterwards.
- The spin mark's clipping, and its document-global id problem, become one component's private
  concern instead of something the inset must know about.
- Stroke and proportion are stated once instead of twice in two units.
- Diamonds, Pins, Sub-Diamonds and path markers have a worked example to follow.

**Accepted gaps:**
- **Adopting one stroke constant changes the table's stroke by about 2%** — from `0.15` cm to
  `0.153` on a `3.0625` cm radius, since `0.15 / 3.0625 = 0.049` and the shared constant is `0.05`.
  Invisible in practice, but it is a change and not a no-op.
- `DiagramBallsComponent`'s `trackBy` stays load-bearing, and becomes *more* so: pointer capture now
  depends on the child Ball's host `<g>` surviving a re-render, so the reason for `trackBy` is no
  longer visible in the component that declares it.
- The Layers chain (`fe`#67, `fe`#68) waits behind a refactor with no user-visible change.
- Scope is one primitive. Until the others follow, the codebase holds both patterns at once.

**Follow-up needed — committed, not optional:**
- **Persisted appearance.** Ball appearance must survive save and reload; the decision to defer it
  here is about sequencing, not about whether it happens. It carries a second requirement with it:
  `BallColor` is a closed union of `white | yellow | red`, and balls or objects whose colour is not
  derived from a role will not fit it. Colour keeps its current semantics for Cue Ball, Object Ball
  and Second Object Ball.
- Diamonds, Pins, Sub-Diamonds and path markers as components, once this pattern has proved itself.
- `ADR-012`'s migration story remains unwritten, and persisted appearance is the change most likely
  to need it — it would be the fourth deferral, and the first with real data in the way.

## 📎 Amendment to ADR-016

`ADR-016` rejected "a single component behind a `readonly` flag", on the grounds that threading the
flag through drag handlers and pointer state accumulates conditionals exercised in one mode and
rotting in the other. **That decision stands for the Hit editor**, which is a stateful gesture
machine with genuinely distinct modes — click versus drag versus mark-drag, decided on `pointerdown`
and fixed for the gesture.

A Ball's `interactive` capability is not the same thing: it guards event bindings and a cursor
affordance on a single unconditional gesture, with no second path through a state machine. The
codebase already contains this distinction unremarked — `DiagramBallsComponent` has carried
`@Input() interactive` since `fe`#22 — and this ADR is where it is written down rather than left for
a reader to infer that `ADR-016` was ignored.

The renderer/editor split `ADR-016` established is unchanged, and the Hit editor keeps it.

## 📎 Amendment to ADR-017

`ADR-017` deferred per-Layer appearance as "its own requirement, unblocked by this". That requirement
is this one, and it arrives as an architecture change rather than a feature: the Ball component is
where appearance becomes expressible. `ADR-017`'s guarantee that "the model leaves room for it" is
now discharged on the rendering side; the persistence side is the committed follow-up above.
