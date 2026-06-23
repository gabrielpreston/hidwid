# Engineering discipline: enforce the invariant, don't rely on the habit

> **In one line.** Convert each engineering discipline from a habit anyone can skip into
> a structure the build itself enforces — so a violation is a visible failure, not a
> silent omission.

**TL;DR — the mechanisms**
- Transitive ownership: a failure you see is yours, whoever caused it — and a success you claim needs fresh evidence, not confidence.
- A pre-commit gate (lint + typecheck + test) every commit must pass.
- Layered import boundaries enforced by lint, not by convention.
- Coverage floors that act as regression floors, set just below current.
- Forward-migrate, never wipe: old saves migrate up; a coverage census proves it.
- No single-player assumptions baked into the data model (the swap contract).
- Measure the blast radius before widening a shared type; never widen speculatively.

**Read this when** you're setting the invariants the codebase enforces mechanically.
**Big ideas:** [mechanical enforcement over habit](OVERVIEW.md#5-mechanical-enforcement-over-habit)

---

How to keep a codebase correct as it grows and as the people (and agents) editing
it turn over — by converting each engineering discipline from a *cultural habit*
("write tests," "don't couple the layers," "own your failures," "don't break old
saves") into a *structural invariant* the build itself enforces. The premise is
that any discipline that survives only as good intention will eventually be
skipped under pressure, by a newcomer, or by an agent that never absorbed the
culture — and the skip is silent until it costs something. The fix, applied seven
ways below, is always the same shape: **encode the discipline as a gate, a floor,
a boundary, or a census, so that violating it is a visible failure rather than an
invisible omission.**

This is project-agnostic doctrine extracted from a working implementation.
Wherever a concrete noun appears — a layer name, a lint rule, a CLI verb, a
coverage tool — it is called out as a **`«slot»`** you fill for your own project;
the [parameterization table](#parameterization-seams) near the end collects every
slot in one place.

It is the **engineering-rigor companion** to the
[session lifecycle doctrine](./session-lifecycle-doctrine.md): that doc owns the
temporal envelope and carries the validation gate as a **`«gate»` slot** — it fixes
*when* the gate runs (the same checks at both the open baseline and the close
merge) but leaves the slot's *contents* to be filled. This doc fills that slot —
the lint/type/test triad of §2 — and specifies the standing invariants the gate and
its CI backstop defend (§1, §3–6). Adopt the lifecycle for *when* work commits;
adopt this for *what must be true* before it does.

---

## Why this shape

A growing codebase accumulates correctness obligations: tests must exist, layers
must stay decoupled, old persisted data must keep loading, failures must get
owned. Every one of those is normally carried as a *norm* — something the team
agrees to and mostly remembers. Norms have three failure modes, and all three
compound over time:

- **Pressure.** Under a deadline the norm is the first thing dropped, because
  dropping it is invisible this week and only costs something later.
- **Turnover.** A new contributor — human or agent — never absorbed the norm.
  Nothing in the repo tells them; the knowledge lived in people who moved on.
- **Erosion.** Each individual skip looks harmless ("just this once"), so the
  norm degrades one reasonable-looking exception at a time until it's gone.

The defense against all three is the same: **make the obligation mechanical.** A
gate that fails the build, a coverage floor that reds CI, an import-boundary lint
that errors, a migration census that won't pass with a gap — these don't depend on
anyone remembering or caring. They convert "you should" into "you cannot proceed
until," which is the only form of discipline that survives pressure, turnover, and
erosion.

The seven rules below are seven applications of that one move. They are presented
roughly in dependency order: ownership (§1) is the posture; the gate (§2) is the
enforcement point; the boundary (§3), the floor (§4), the never-wipe contract
(§5), and the no-singleton contract (§6) are the standing invariants the gate and
its backstop defend; the speculative-pass gate (§7) protects the costliest change
of all — a widening of the shared type contract — by measuring its blast radius
before the work is scoped.

A recurring sub-theme worth naming up front: **a gate that can pass without
enforcing anything is worse than no gate, because the green result buys false
confidence.** A coverage gate that silently passes when no tests ran, a migration
census that counts a deleted test as covered, a budget check that a worktree can
bypass — each *looks* green while enforcing nothing. Several rules below spend
their hardest design effort not on the happy path but on closing the gate's own
escape hatches.

---

## Part 1 — transitive ownership of failures

### The posture

**Any failing check you encounter is your problem, regardless of who caused it.**
The moment a session *sees* a failure — at the opening baseline check, mid-work,
or at the closing gate — it has acquired ownership of the *decision* about that
failure, even if its own changes are blameless. "Not mine, it was already broken"
is not a legitimate response. The repo state when you finish is the baseline the
next session inherits; a failure you saw and left uncaptured silently degrades
that baseline for everyone after you.

This is the cultural keystone the structural rules rest on. Gates and floors catch
*new* breakage; transitive ownership is what keeps *inherited* breakage from
becoming permanent background noise that trains everyone to ignore red.

### The three legitimate responses

When a failure surfaces, exactly three responses are on the table. Pick one for
each failure — walking past is not on the list.

1. **Fix it.** If the failure is in scope, or cheap enough that triage would cost
   more than repair, fix it inline. If the underlying bug had no coverage, add a
   regression test. Commit it separately from the session's main work so the fix
   is independently reviewable and revertible.
2. **Capture it.** If it is genuinely out of scope (different subsystem, needs a
   decision this session can't make), file a tracked work item — *with the
   evidence*: the failing check and its assertion, actual-vs-expected output
   pasted from the runner, the commit where it was first observed, and a
   best-available hypothesis from the history of the relevant files. The ticket is
   the paper trail; without it the failure is invisible to the next session.
3. **Surface it.** If neither fix nor capture fits — the check itself looks wrong,
   the project is mid-pivot, the failure hints at something deeper — stop and
   escalate to a human. Don't proceed past a failure you neither understand nor
   documented.

The **forbidden fourth response** is the one this rule exists to kill: noting the
failure in passing ("there's a pre-existing failure in X") and moving on having
done none of the three. That is how a baseline rots.

### Why ownership has to be transitive

The reason it transfers to whoever *sees* it, not whoever *caused* it: causation is
often unknowable and always backward-looking, while the next session's experience
is concrete and forward-looking. Tying the obligation to *observation* makes it
actionable by the one person guaranteed to be present — the one looking at the red
right now. The opening baseline check exists precisely to *force* that observation
early, so the choice is made deliberately at the start rather than discovered at
the close.

### Evidence over confidence — the success-side mirror

Ownership governs the failures you *see*; its mirror governs the successes you
*claim*. **Make no "done" or "passing" claim without fresh evidence in the same
breath — run the check, read the output, then claim.** A remembered green from
earlier in the session, a confident "that should work," a subagent's report of
success — none of these is evidence; they are assertions about a build state nobody
just observed. Confidence is not evidence, and the gap between them is exactly where
a regression hides: the change that "obviously" couldn't break anything, claimed
done without a re-run, inherited as background red by the next session.

The two halves are one posture: an honest relationship to the build's actual state.
One half forbids ignoring red you saw; the other forbids asserting green you didn't.
Both fail the same way — substituting what you *believe* about the build for what you
just *watched* it do. The discipline is to close that gap at the moment of the claim,
not to be more careful in general.

This posture has two enforcement points elsewhere in the corpus, and naming them
keeps this from being mere exhortation. When the work is delegated, a subagent's
green report is a *claim* the lead re-verifies before accepting it — the
post-completion validation gate in the
[multi-agent coordination doctrine](./multi-agent-coordination-doctrine.md). When the
harness allows it, the claim is mechanically gated: a stop-hook loop that re-runs the
checks and refuses to let a turn end on an unproven "done" — the run-until-verifiable
loop in the [harness doctrine](./claude-code-harness-doctrine.md). The posture is the
general form; those two are where it is made to bind.

### Adapt this

The compliance condition for a session: at close, **every failure it encountered
has a corresponding fix commit, a tracked ticket, or an escalation on record.**
Zero invisible failures. If your work is agent-driven, every agent that runs
checks inherits this rule verbatim — an agent that reports "tests pass except for
some unrelated failures" without saying what it did about each one has violated it,
and the supervising context re-runs it with the rule cited. *Project nouns to
fill:* `«tracker»` (where captured tickets land), `«baseline-check»` (the opening
step that forces early observation).

---

## Part 2 — the pre-commit gate

### The posture

**Nothing commits until a fixed, ordered set of checks all pass.** The gate is the
single chokepoint where the standing invariants get enforced on every change. Its
power comes from being *non-optional and uniform*: the same checks, in the same
order, run before every commit, so "it compiles on my machine" and "I'll fix the
types later" stop being possible states to commit from.

A representative triad, in cost-ascending order so the cheapest feedback comes
first:

| Check | What it defends | Why this order |
|---|---|---|
| **Lint** (`«linter»`) | style, common-bug patterns, the layer boundaries of §3 | fastest; fix these first |
| **Type check** (`«typechecker»`) | interface contracts, null-safety, signature drift | slower than lint, faster than tests |
| **Tests** (`«test-runner»`) | behavioral correctness, the coverage floors of §4 | slowest; run last |

Run them cheapest-first because a lint error is often the *cause* of a downstream
type error, and fixing it first avoids chasing a phantom.

### Strict by default, disable by exception

The type checker and linter run in their **strictest** mode — full strict-type
flags, type-aware lint rules, not just syntax checks. Strictness is the point:
loose settings let whole categories of bug commit cleanly. Suppression
(`«disable-directive»`) is permitted only for a genuine false positive or an
irreducible test-mock case, **always with a comment stating why**, and **never**
file-wide. A suppression without a reason is itself a violation the reviewer
rejects. The standing rules a suppression may *never* dodge are the structural
ones — a layer-boundary violation (§3) is fixed by restructuring, not silenced.

### Landing a new strict rule on an already-dirty codebase

Turning a rule to its strictest setting is easy on a clean tree and infeasible on a
dirty one: a new rule pointed at existing code typically lights up hundreds of
pre-existing violations, and a single big-bang refactor to clear them all is exactly
the unreviewable mega-change the gate exists to prevent. The move that lets you
*adopt* strictness without a flag-day is a **warn-to-error ratchet**, the lint cousin
of the coverage ratchet (§4):

1. **Land the rule as a non-blocking `warn`.** The tooling merges immediately with no
   refactor attached; the warn channel makes the latent debt *visible and countable*
   without reddening the build. (Rules with a *small, all-genuine* violation count —
   often the correctness/security ones — can skip straight to `error`; landing those
   as `warn` just trains people to ignore the warn channel. The exception is when the
   initial hits are mostly *false positives*: stage even a low-count rule through
   `warn` for a triage pass first, then flip to `error` once each hit is dispositioned.
   Reserve the warn stage otherwise for the high-count complexity/duplication rules.)
2. **Burn the warn-set down to zero.** This is a coverage-add sweep in the backlog
   sense — its cost is the area's latent debt, so enumerate it at groom time and
   split it from any point fix (see the companion backlog doctrine).
3. **Flip the rule `warn → error`, gated at zero.** Once a rule is *globally* zero it
   becomes a blocking error, so its count can only ever stay flat or shrink — a new
   violation fails at the source. **A per-rule global gate cannot flip until that
   rule is globally zero**; a partial cleanup leaves the rule stuck in `warn` and the
   debt re-growable.

If a flip later surfaces a genuine false positive, disable it *inline at that one
site with a stated reason* — never relax the rule back to `warn` globally, which
re-opens the ratchet you just closed.

> **The lint cache lies after a config edit.** A cached linter run, right after you
> change rule config, can report a near-empty result instead of the true count — the
> stale cache short-circuits the run. When measuring violation counts or verifying a
> warn→error flip, clear the cache or run uncached; trusting the cached number
> silently under-reports the debt and mis-sizes the sweep.

### Two enforcement layers

The gate works best as two layers so neither a forgetful contributor nor a
misconfigured local hook can land unchecked code:

- **A local pre-commit/pre-push hook** gives fast feedback and blocks the obvious
  case. Hooks are fragile — they can be unset by an editor, lose their exec bit,
  or be deliberately bypassed — so they are necessary but not sufficient.
- **A CI backstop** re-runs the same checks server-side and is the
  *non-bypassable* layer. Automated lifecycle commits may carry an explicit,
  auditable bypass sentinel for the local hook (so tooling isn't blocked by an
  interactive gate), but **CI never honors the bypass** — it is the floor that
  holds when every softer layer is skipped.

If editing the gate's checks is itself substantial work, that batch of small,
same-layer fixes can run the gate once for the batch rather than per-fix — but only
when the changes are mechanical, within one or two layers, and touch no shared
contract. Feature work and any change to a shared type does not get the batch
exemption.

### What the per-commit gate deliberately leaves out — and the blind spot that creates

The triad is the *fast* gate. Heavy end-to-end tests (browser-driven, slow, flaky
under parallel contention) deliberately **do not** run per-commit — they run in CI,
often only a *subset*, and on demand. That keeps the commit loop fast, but it opens a
real blind spot: a change that alters UI selectors, navigation flow, or render order
can pass lint + types + unit cleanly while **reddening the full end-to-end suite**,
and because no per-commit check exercises that suite, the regression lands on the
trunk unnoticed and resurfaces later as "pre-existing CI red" nobody can date.

The mitigation is to make it a **gate, not a habit** — and to put the gate at the
*merge*, not the commit. Fuse the end-to-end suite into the merge-to-trunk step so the
merge itself runs it (at CI parity) and refuses on failure. The merge boundary is the
right home for two reasons: the per-commit triad has to stay fast, so e2e can't run on
every commit; and a post-merge CI run *can't block* a merge that already happened, so
catching the regression one step earlier — as the branch goes in — is the last moment
it can be stopped before it reaches the shared trunk. Auto-skip the gate for changes
that touch no UI (docs, config) so it costs nothing when it can't fire, and gate the
production-release boundary the same way. An earlier version of this was a habit —
"remember to run the affected specs before merging" — and that is exactly how a UI
regression reached the trunk: a habit is silently skippable, a gate fused into the
merge script is not.

A sharper version of the same trap is **dev-server-vs-production-bundle divergence.**
End-to-end specs run against a dev server (fast) behave differently from the same
specs run against a production-shaped bundle (what CI builds), because build-time flag
elimination differs between the two. A test-only affordance gated on a *dev-only*
build flag renders under the dev server — so the spec passes locally — but is stripped
from the production bundle, so the same spec times out in CI. Two consequences worth
internalizing: gate any end-to-end-reachable affordance on the **same condition the
test harness reaches under** — the dev flag *OR* a dedicated test-hooks flag
(`dev-flag || test-hooks-flag`), not a bare dev-only flag (the test-hooks flag is what
the production-shaped CI bundle sets, so it must be added alongside the dev flag, not
substituted for it);
and know that the **local dev-server run cannot catch this class of failure** — only
the production-bundle path exercises it, so run that path before merging a change to a
test-reachable affordance.

### Adapt this

*Project nouns to fill:* `«linter»` / `«typechecker»` / `«test-runner»` (your
three checks), `«disable-directive»` (your suppression syntax), `«hook-bypass»`
(the auditable sentinel automated commits set for the *local* layer only),
`«ci»` (the non-bypassable backstop). The non-negotiable: **the same checks gate
both the local commit and the server-side merge, and only the local layer is ever
bypassable.**

---

## Part 3 — layered import boundaries

### The posture

**Dependencies flow in one direction only; a layer may never reach upward or
sideways.** Pick a small number of layers with a strict ordering — a common shape
is `UI → application → domain ← content`, with persistence depending only on the
domain — and forbid every import that runs against the arrows. The payoff is
that each layer stays independently testable and replaceable: the domain core has
no dependencies and runs in isolation, the UI swaps without touching simulation
logic, content loads without booting the app.

This is not a style preference. An upward or sideways import creates coupling that
compounds — and once one exists, the next is easier to justify, until the layers
are a ball of mud. So the boundary is enforced as a **blocking lint error**
(via `«import-boundary-lint»`, part of the §2 gate), not a review convention. You
cannot commit across the boundary.

### The fix is always an abstraction, never a shortcut

When you find yourself wanting to import across a boundary, the import is the
symptom; the cause is a *missing abstraction at the correct layer*. The fix has a
fixed shape:

1. Identify what's actually missing — usually a method on a lower-layer service,
   or a shared type.
2. Add that abstraction at the layer that's *allowed* to provide it.
3. Import the abstraction, not the implementation.

Worked instance: a UI component needs catalog data that lives in the content
layer, which UI may not import. Wrong fix: import the content file directly. Right
fix: add a typed accessor to the application-layer service that already loads that
content, call it at the composition root, and pass the result down as data. The UI
receives typed values; the boundary holds. The same move resolves "domain needs
content" — content data flows *into* domain functions as plain arguments passed by
the application layer, because the domain imports nothing.

### Adapt this

*Project nouns to fill:* your layer set and their ordering, and `«import-boundary-lint»`
(the rule that makes a violation a build error with a clear message naming the
offending direction). The portable invariant: **a dependency that runs against the
arrows is a blocking error fixed by adding an abstraction at the right layer — never
a shortcut, never a suppression.**

---

## Part 4 — coverage floors as regression floors

### The posture

**Test coverage is enforced as a per-area floor that CI reds below, set
deliberately *below* current levels.** The counter-intuitive part is the second
half. A floor set *at* the current number turns every percent of noise into a red
build and trains people to game it; a floor set a few points *under* current lets
CI absorb measurement noise while still catching a real regression — a meaningful
drop in covered code. The floor is a *ratchet against backsliding*, not an
aspiration to chase.

Coverage is enforced per *area* (per layer or module glob), because a single
whole-repo number lets a well-tested core hide an untested edge. Pure-logic areas
(a domain core, persistence) carry the highest floors because they are cheap to
test and expensive to get wrong; presentation areas are often better served by
end-to-end or integration tests and may be excluded from the unit-coverage rollup
rather than gamed into it.

### Floors are a ratchet, and aspirations are separate

Two numbers, kept distinct:

- The **enforced floor** sits a few points below current and reds CI. When current
  values comfortably clear the next breakpoint, *raise the floor* and record the
  new number — the ratchet only ever tightens.
- The **aspirational target** is where you'd like the area to be. It is documented
  but *not* gated, so it can pull effort without failing builds on the way.

### When a test is owed

Tie new tests to events, not to a vague "add coverage" backlog:

- Every new unit of domain logic gets at least one happy-path and one edge-case
  test.
- Every change to persisted-data shape gets a round-trip test (see §5).
- **Every bug fix gets a regression test that proves the fix holds** — this is the
  one that compounds, because it converts each bug into permanent protection
  against its own recurrence.

Periodically (or before a major release) a dedicated hardening pass targets the
under-covered complexity the floors don't force — the parameter-combination
explosions and state-transition matrices where bugs actually hide.

### Adapt this

*Project nouns to fill:* `«coverage-tool»`, your per-area floors and the globs they
apply to, and which areas are unit-gated versus end-to-end-owned. The portable
invariant: **the floor is a ratchet set below current so CI absorbs noise but
catches regression; raise it as current rises; keep the aspiration documented but
ungated.** Beware the silent-pass blind spot — a coverage run that instruments
*zero* tests (a filter excluded them all) can report a vacuous green; the gate
should fail loud on "ran but measured nothing," not pass.

---

## Part 5 — forward-migrate, never wipe

### The posture

**Every change to a persisted-data shape ships a migration that carries old data
forward; saved data is never wiped to accommodate a schema change.** Persisted
data outlives the code that wrote it — a returning user loads a save written by a
months-old version. Treat that old shape as an input contract, not an
inconvenience. The discipline: append a migration step that transforms the
previous shape into the new one, run the full chain of steps on load, and the user
keeps everything.

The procedure for adding a field is fixed: add it to the type, give it a default
at the construction site, **append a migration step that backfills it on old
data**, and write a round-trip test (old shape in → new shape out). A single
version counter records the last migration applied; on load, every step from that
counter forward runs in order. New saves start fully migrated.

### The coverage census, and its blind spot

A migration chain needs its own coverage guarantee: **every step must be exercised
by a round-trip test.** The trap is making that guarantee *honest*. A naive
approach — scrape the test source for calls referencing each step — counts a test
that exists but never runs (skipped, or filtered out) as covered. That is the
exact blind spot a structural gate must close.

The robust form is **reporter-attributed coverage**: a test *declares at runtime*
which step it covers (a stamp set from inside the test body), and a custom reporter
aggregates those stamps across the full suite and fails the build on any step that
is neither exercised nor explicitly grandfathered. Because the stamp is set only
when the test actually executes, a deleted or skipped test silently drops coverage —
which is precisely the failure the gate now catches instead of masking. Guard rails:
enforce the census only on a *full* suite run (a filtered run skips cleanly), and if
the full suite runs but stamps *zero* steps, fail loud as a mis-wire rather than
passing vacuously — the same silent-pass defense as §4.

### The backfill ban

A sharp corollary that catches people: **never paper over a missing field with an
inline default at read time** (`field ?? default` inside a service method). If the
load path re-reads raw persisted data — as a tick loop or any periodic reload does —
an in-memory default never persists and reappears as a crash or a `NaN` on the next
read. The field must be backfilled *by a migration* so it is actually written to
storage. This ban is independent of the migration policy and holds regardless: the
migration is the only correct place to introduce a default for a new field. (A
read-time `?? 0` for a genuine map-lookup or search fallback is fine; a `?? default`
standing in for a *new schema field* is the banned case, and is worth catching with
a diff-scanning check.)

The wipe is the break-glass of last resort — reserved for catastrophic corruption,
documented in place, and never the easy way out of writing a migration.

### Adapt this

*Project nouns to fill:* `«migration-list»` (the ordered steps), `«version-field»`
(the last-applied counter), `«coverage-meta»` + `«coverage-reporter»` (the runtime
stamp and the aggregator that fails on a gap), `«backfill-check»` (the diff scan for
new read-time defaults). The portable invariant: **old saves migrate forward and are
never wiped; every migration step is covered by a test that must actually run to
count; and a new field's default lives in the migration, never in a read-time
fallback.**

---

## Part 6 — no single-player assumptions in the data model

### The posture

**The data model and storage layer are multi-tenant-aware from day one, even when
the product ships single-tenant.** This is the cheapest-now / most-expensive-later
invariant in the set: baking a singleton assumption into the data model costs
nothing today and forces a rewrite the day a second tenant arrives. The rule is not
"build multi-tenancy" — it's "don't foreclose it." (Substitute your axis of
multiplicity for "tenant": user, account, workspace, org.)

The concrete constraints, all mechanical and checkable:

- **No global "current entity" and no singleton state object.** Every piece of
  state is scoped by an explicit id.
- **Every state-accessing operation takes that id as its first parameter** —
  `getState(id)`, never `getState()`.
- **Storage keys are always namespaced by the id** — a structured key like
  `«namespace»:<id>`, never a bare global key.
- **No cache crosses the id boundary.** Two different ids must never share or
  overwrite each other's state; same-id write-through caching for performance is
  fine as long as it preserves read-after-write per id.

### The swap contract

The forward-looking payoff is a **swap contract**: a single storage *interface*
with a local/simple implementation now, replaceable later by a networked/shared
implementation **without changing the interface or any call site.** That contract
is what makes "single-tenant now, multi-tenant later" a swap instead of a rewrite.
Any data-model decision that would break the swap gets flagged in place with an
explicit marker (`«swap-risk-marker»`) naming why — so the cost is visible at the
moment it's incurred, not discovered at migration time.

What this explicitly does **not** demand: shared state, conflict resolution,
real-time sync, or any actual multi-tenant feature. The whole burden is *not baking
in assumptions that would require a rewrite* when multiplicity arrives. That is a
data-modeling discipline, not a feature.

### Adapt this

*Project nouns to fill:* your multiplicity axis and its id, `«namespace»` (the
storage-key prefix), `«storage-interface»` (the swappable adapter contract), and
`«swap-risk-marker»` (the in-code flag for a decision that would break the swap).
The portable invariant: **scope every operation and storage key by an explicit id,
let nothing assume a singleton, and keep the storage interface swappable — so the
move to real multiplicity is a contained adapter swap, not a model rewrite.**

---

## Part 7 — the speculative-pass gate for shared-type changes

### The posture

Before *committing a plan* to widen or weaken a high-blast-radius shared type in
`«shared-type-module»` — nullable a non-null field, make an interface field
optional, generalize a discriminator — **measure the real blast radius first.** Make
the one-line type change, run the type checker, count the errors and the number of
distinct files they span, then revert. Only then decide how to scope the work.

The reason to measure rather than estimate: hand-estimation of a type-widening
blast radius **systematically under-counts** it. The missed sites are always in the
indirect layer — test fixtures that spread the narrowed shape, service methods that
accept a derived sub-shape and see the narrowing collapse through the call graph,
pure helpers that read several fields off the type. None of these are visible from
a grep on the field name alone.

### The decomposition decision

The measured count drives a tiered decision:

| Blast radius | Action |
|---|---|
| Small (a few errors, a handful of files) | Fix call sites inline during the session |
| Medium (tens of errors, tens of files) | Fan out to an agent team, one per affected area |
| Large (hundreds of errors, many files) | **Decompose**: introduce a narrowing accessor first, migrate consumers through it with no type change, then widen — at which point only the genuinely null-capable call sites need handling |

The decomposition pattern deserves a sentence: the wrong move on a large blast
radius is to widen the type first and then fix the call sites one by one. The right
move is to introduce a `«require-accessor»` helper (e.g. `requireX(state): X` that
asserts non-null and throws at the one legitimate null site), migrate every consumer
to it with *no type change* (so the build stays green), and only then widen the
type — at which point the blast radius is the handful of call sites that genuinely
cannot hold the accessor.

### The prohibition: no speculative widening

The positive rule: **do not widen a shared type to be safe or in case it's needed
later.** Every nullable-a-field or make-optional change to `«shared-type-module»` is
a blast-radius event. Widen only when a concrete, in-hand requirement forces it, and
only after the speculative pass has measured the cost. "I might need this later" is
not a concrete requirement; it is the preamble to an undocumented, unmeasured type
change that the next session inherits as background red.

### Re-run the gate on downstream work, not just at planning

A prep session that introduces the narrowing accessor does not automatically make the
downstream widening cheap. Re-run the speculative pass on the downstream branch
before its first substantive commit — prep sessions routinely miss adjacent layers
outside their declared scope (tests, fixtures, harness tooling), and a "prep is
done" claim that hasn't been re-measured is a plan assumption, not a measurement.

### Adapt this

*Project nouns to fill:* `«shared-type-module»` (your central, high-blast-radius
type file — the one everything imports, lead-owned), `«require-accessor»` (the
pattern name for the non-null assertion helper that makes the large-blast-radius
decomposition tractable). The portable invariant: **measure the blast radius before
planning the work, use the measured count to choose inline / team / decompose, and
never widen a shared type speculatively.**

---

## Parameterization seams

| Slot | Source binding | Agnostic form |
|---|---|---|
| `«tracker»` | the backlog system | wherever captured-failure tickets land with their evidence |
| `«baseline-check»` | the start-session baseline step | your opening step that forces early observation of inherited failures |
| `«linter»` / `«typechecker»` / `«test-runner»` | ESLint / `tsc` / the unit runner | your three gate checks, run cheapest-first |
| `«disable-directive»` | the inline lint-suppression comment | your suppression syntax — permitted by exception, with a reason, never file-wide |
| `«hook-bypass»` | the automation bypass sentinel for local hooks | the auditable env flag automated commits set to skip the *local* gate only |
| `«ci»` | the CI workflow | the non-bypassable server-side re-run of the same checks |
| layer set + ordering | `UI → app → domain ← content`, persistence → domain | your layers and their one-directional dependency arrows |
| `«import-boundary-lint»` | the per-layer restricted-import rule | the lint rule that makes a cross-boundary import a build error |
| `«coverage-tool»` + floors | the coverage instrument and per-glob thresholds | your coverage measurement and per-area floors set below current |
| `«migration-list»` / `«version-field»` | the ordered migration array + last-applied counter | your forward-migration chain and its version pointer |
| `«coverage-meta»` / `«coverage-reporter»` | the runtime stamp + custom reporter | the runtime-attributed migration-coverage census that fails on a gap |
| `«backfill-check»` | the diff scan for new `?? default` reads | your guard against read-time defaults standing in for schema fields |
| multiplicity axis + id | `playerId` | your tenant/user/account id that scopes all state |
| `«namespace»` | the `prefix:<id>` storage key | your namespaced storage-key scheme |
| `«storage-interface»` | the persistence adapter interface | the swappable storage contract held stable across implementations |
| `«swap-risk-marker»` | the in-code `MULTIPLAYER-NOTE` flag | the marker for a decision that would break the storage swap |
| `«shared-type-module»` | the central type-contract file, lead-owned | your high-blast-radius shared-type file; widening requires a speculative-pass measurement first |
| `«require-accessor»` | the non-null assertion helper pattern | your accessor that lets consumers migrate away from a widened type without touching the type itself |

---

## When to adopt this

These seven rules pay off at different sizes, so adopt them as the relevant pressure
arrives rather than all at once:

- **Transitive ownership (§1)** the moment more than one session (or person, or
  agent) touches the repo over time — it is what stops inherited red from becoming
  permanent.
- **The gate (§2)** as soon as you have a lint, a type checker, and a test suite
  worth not breaking — it is the chokepoint the other rules hang off.
- **Layer boundaries (§3)** when the codebase is large enough that "everything can
  import everything" has started to hurt — and *before* it hurts is cheaper.
- **Coverage floors (§4)** once you have a test suite you want to defend against
  silent erosion.
- **Forward migrations (§5)** the first time real persisted data exists that a user
  would be upset to lose — which is usually sooner than you think.
- **Multi-tenant data model (§6)** from day one if there is *any* chance of a second
  tenant, because it is the only rule here that is dramatically cheaper to honor
  before the first line of model code than after.
- **Speculative-pass gate (§7)** the first time a shared-type change turns a
  one-session scope into an unplanned multi-session refactor — after which the gate
  becomes a standing discipline on every planned type widening.

The non-negotiable core, if you adopt nothing else: **a single pre-commit gate
that every change passes and that CI re-runs un-bypassably**, and the posture that
**every failure you see is yours to fix, capture, or escalate — and no success is
claimed without fresh evidence.** The gate is where
the standing invariants get enforced; transitive ownership is what keeps the gate's
own inherited failures from rotting. Everything else — the one-directional layer
lint, the coverage ratchet, the migration census, the id-scoped data model — is a
specific invariant bolted onto that frame. The unifying move never changes: find
the discipline you're relying on people to remember, and make the build remember it
instead.
