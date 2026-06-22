# Frontend anti-slop: look before you change, and know what your model defaults to

> **In one line.** Keep agent-produced UI from converging on the model's competent,
> on-distribution default — by breaking that default and forcing every visible change to
> be a looked-at one.

**TL;DR — the mechanisms**
- Capture before/after screenshots: make every UI change a looked-at change.
- Name your model's default house style so you can deliberately break it.
- A two-surface font policy: distinctive display type, legible body type.
- A throwaway verdict agent reviews the pixels so the lead never reads them.

**Read this when** an agent is producing user-facing UI you don't want to look generated.
**Big ideas:** [defend the experience in review](OVERVIEW.md#7-defend-the-experience-in-review)

---

How to keep agent-produced or AI-assisted UI from converging on a generic,
"designed-by-committee" look — and how to make every user-visible change a
*looked-at* change rather than a blind one. The premise is that a capable model,
left to its own judgement, produces interface work that is competent, plausible,
and **on-distribution** — which is exactly the problem. It looks intentional while
being the average of everything the model has seen. The defenses below are not
about taste; they are about **breaking the default and forcing observation**, so
that the pixels that ship are ones someone (or some process) actually evaluated.

This is project-agnostic doctrine extracted from a working implementation.
Wherever a concrete noun appears — a palette direction, a font name, a viewport
list, a port script — it is called out as a **`«slot»`** you fill for your own
project; the [parameterization table](#parameterization-seams) near the end
collects every slot in one place, and a [strip-these-bindings](#what-to-strip)
section lists the source project's expression so you don't inherit it by accident.

It is the **player-facing-surface companion** to the
[engineering-discipline doctrine](./engineering-discipline-doctrine.md): that doc
converts correctness norms into build-enforced invariants (the lint/type/test
gate, layer boundaries, coverage floors). This doc covers the one class of change
that gate *cannot* catch — a UI that compiles, passes every test, and still looks
generic or silently regressed. Visual quality has no green checkmark; it has a
disciplined looking-loop instead. Adopt the engineering doctrine for *what must be
true* before code commits; adopt this for *what must be looked at* before UI
ships.

---

## Why this shape

Four observations drive the four parts below.

1. **A model's weak point is not generic slop — it is its own house style.**
   Anti-slop guidance written against an older model forbids *that* model's
   tells (a named sans-serif, a particular gradient). The model you are using now
   has a *different* default, and a checklist of yesterday's tells gives zero
   protection against it. Worse, the current default usually *looks* deliberate,
   so it passes a casual review. Anti-slop must be **model-relative**, and the
   only reliable levers are concrete, not prohibitive.

2. **The build gate is blind to appearance.** Lint, types, and tests verify
   behavior and structure. None of them can tell you a layout got denser, a
   hierarchy inverted, or a ceremony moment was flattened. The only instrument
   for visual regression is **a human or agent reading the before and the after**
   — which means the looking has to be a required step, not a hoped-for habit.

3. **Iteration without a stop condition burns budget for diminishing return.**
   A capture-edit-recapture loop is the right shape for visual polish, but it has
   no natural terminus — there is always one more nitpick. It needs a **hard cycle
   cap** so it converges and hands unresolved calls back to a human.

4. **Verification is expensive in the wrong context.** Reading full-page images
   into the orchestrating agent's context to answer "does this look right?" is the
   single most wasteful thing a UI session does — the answer is one word, but the
   image costs thousands of tokens and pollutes the driver's working set. The
   verdict belongs in a throwaway context that returns a string.

The four parts are four applications of one move: **make the model observe rather
than assume, and put each observation where it is cheapest.**

---

## Part 1 — anti-slop is relative to the model you are using

### The trap

Generic-AI-aesthetic guidance is usually a *blocklist*: avoid these overused
fonts, avoid this clichéd gradient, avoid these predictable layouts. That list
was written against a specific model's distribution. The model in front of you
has its own distribution, and therefore its own tells — a preferred background
warmth, a default display typeface, a signature accent color, a habitual layout
rhythm. **A blocklist of the previous model's tells does not constrain the current
model at all.** It will dutifully avoid the banned items and land on its *own*
default, which the list never named.

The trap has a second jaw: the current model's default tends to look
**intentionally designed**. It is not obviously broken or obviously cheap — it is
a coherent, tasteful, *generic* style. That makes it survive a casual "looks fine"
review, which is the exact moment slop ships.

### Why prohibitions fail and concretes work

Telling a model "don't use «its-default-palette»" or "make it less generic" does
not produce variety — it produces *a different fixed default*. Negative,
directional instructions push the model from one attractor to another; they never
push it off attractors entirely. Two levers actually work, both concrete:

- **Specify the spec.** Give exact values — background and accent as literal color
  tokens, display and body typefaces by name, spacing/density targets in real
  units. A capable model follows a concrete spec precisely. The specificity is the
  point: every degree of freedom you leave to "judgement" is a degree the default
  reclaims.

- **Propose-then-pick.** For a new surface with no locked direction, have the
  agent propose 3–4 *distinct* directions — each a one-line rationale plus its
  concrete tokens (background hex, accent hex, type pairing) — then a human picks
  one and the agent builds only that. The act of generating genuinely different
  candidates breaks the single-default pull; the human pick supplies the taste the
  model cannot.

> The corollary that bites: **if your own design direction happens to resemble the
> model's default, you must specify it, never imply it.** A direction left to
> "judgement" will be reconstructed *as the generic default* and read as
> deliberate. Cite the live token source as ground truth; do not let the model
> re-derive the look from an adjective.

### Keep a minimal verbatim anti-slop block — but know what it is for

A short, model-current anti-slop instruction (ban the *current* model's actual
tells; demand committed, dominant-color-plus-accent palettes over timid evenly-
distributed ones; reserve motion for a few high-impact moments; build atmosphere
rather than defaulting to flat fills) is still worth carrying in any UI-producing
agent's brief. Its job is to set a floor. But the **higher-signal half is the
model-default warning above** — when the block and the default-warning compete for
space, keep the warning. Re-audit the block whenever the model tier changes; a
stale anti-slop block is worse than none because it creates false confidence.

### The two-surface font split — distinctiveness vs. legibility

Anti-slop guidance that says "avoid system fonts" is right for *display* surfaces
and wrong for *body* surfaces. System fonts are the operating system's
reading-optimized choice for long-form and dense text; overriding them everywhere
trades legibility for distinctiveness on surfaces that gain nothing from it — an
accessibility regression dressed as a design choice. Split the ban by surface
class:

| Surface class | Examples | Font policy |
|---|---|---|
| **Display** | titles, headings, modal/section headers, narrative prose blocks, ceremony lines | The anti-slop ban applies in full — a distinctive, chosen typeface, no generic default sans. |
| **Body / data** | paragraphs, table cells, numeric data, tooltips, form inputs, dense rows | Relax to a **reading-optimized fallback chain** for legibility — prefer the platform's serif/reading stack over a generic `system-ui`/`sans-serif` keyword, but never override into something less legible for the sake of distinctiveness. |

The split is enforced at design-time, not by a linter. The tie-breaker for an
ambiguous surface: **what does the user look at for more than a few seconds at a
time?** If the answer is the text itself, it is a body/data surface and legibility
wins.

---

## Part 2 — look at the pixels before you change them

### The rule

Before the **first edit to a user-facing UI file** in a session or task, capture
and analyze the before-state. UI is a first-class concern; editing it blind
destroys the ability to do honest before/after comparison and silently flattens
informed choices into "looks fine I guess." The looking is a *required step that
precedes the first edit*, not a review you do afterward.

### What counts as user-facing

A change is user-facing when **someone returning to the product would notice the
difference within ~30 seconds of use**: component swaps, layout shifts
(list→grid, route→modal), color/hierarchy/token changes, copy rewrites that
change density or hierarchy, new panels or affordances. A change is *not*
user-facing when the rendered output is pixel-identical (pure refactor, file
moves, type-only edits), when it is test- or fixture-only, or when it is data that
flows through an existing unchanged render path. When uncertain whether a refactor
is truly pixel-identical, prove it: capture, change, recapture, diff.

### The protocol

1. **Run the app** against a seeded state that surfaces the affected screen(s).
   Use whatever test/dev bridge injects fixture state deterministically.
2. **Capture the before-state.** Prefer a **structural/accessibility-tree
   snapshot** for any *functional* question (is the element present? is the role
   right? does keyboard nav work?) — it is far cheaper than an image. Escalate to
   a **screenshot** only for questions that require visual judgement (layout,
   color, hierarchy). For design comparison, capture the relevant `«viewport-set»`
   and both themes when a color/contrast/hierarchy decision is in scope.
3. **Read each capture and write a sentence or two of observation notes** — what
   currently works that the change must preserve, what is weak that the change is
   trying to fix, and anything the pixels show you that the spec did not mention.
4. **Stash the evidence ephemerally** (a scratch/`«tmp»` path), never in the repo.
   Before/after diff artifacts are session evidence, not committed assets — the
   repository stays free of them.
5. **Then edit.** After the change lands, recapture at the same viewports/themes
   and compare. This catches the regressions the test suite cannot measure —
   hierarchy inversion, ceremony loss, an accidental density spike — and it
   happens *before* the work is reported complete, not after.

### A portable gotcha: trust disk, not the rendered frame

If a post-change capture looks identical to the before-capture and you suspect the
hot-reload never fired, **verify the served asset reflects disk before trusting
the pixels** — fetch the transformed source from the dev server and confirm your
edit is in it. A stale-transform / caching layer can make a real change invisible
and a real session chase a phantom. The instant stopgap is restarting the dev
server; the definitive check is confirming the served bytes changed.

---

## Part 3 — iterate in a bounded loop

When a surface needs polish rather than a single specified change, the right shape
is a closed loop: capture → evaluate against a rubric → edit → recapture →
re-evaluate, ending with a multi-viewport / multi-theme verification pass. Three
properties make the loop trustworthy:

- **Evaluate against an explicit rubric, not vibes.** Score each pass on the same
  named dimensions (hierarchy, semantic color, density/scannability, touch
  ergonomics, text quality, aesthetic alignment, motion/feedback) and tag each
  finding by severity. A fixed rubric makes "better" falsifiable and keeps
  successive passes comparable.

- **Prefer the cheap instrument first.** Re-check structure with an
  accessibility-tree snapshot every cycle; only spend a screenshot when a *visual*
  finding (color, aesthetic) actually needs the pixels. Most cycles resolve on the
  structural check alone.

- **Cap the cycles — hard.** Set a maximum (a small number — three is a sound
  default) of edit cycles. If high-severity findings remain after the cap, **stop,
  report them, and let a human decide** — do not enter another cycle. Responsive
  fixes discovered during the final verification pass are scoped repairs, not new
  cycles, but they get their own small bound too. The cap is what converts an
  open-ended polish loop into a terminating process.

The output is a structured report: what was evaluated, the per-cycle finding
counts, what got resolved with the specific change made, what remains (with
severity and reason), and a viewport×theme verification table. The report is the
artifact that lets a human ratify or override without re-running the loop.

---

## Part 4 — keep verification out of the driver's context

The orchestrating ("lead") context is the most expensive place to answer a visual
yes/no question, and the most harmful to pollute. Reading a full-page image into
it to confirm "does the border render at mobile width?" costs thousands of tokens
for a one-word answer and leaves the image resident in the driver's working set
for the rest of the session.

The rule: **a verification verdict is produced in a throwaway context and returned
as a string.** Spawn a disposable subagent (in its own isolated workspace if it
needs to run the app) that captures, reads, judges, and returns one line —
"contrast passes" / "border missing at mobile" / "looks correct." The driver
receives the verdict, never the pixels; the subagent's context is discarded.

Two refinements:

- **Use the structural snapshot first even here.** If an accessibility-tree
  assertion answers the question, no screenshot is needed at all.
- **The one exception** is a genuinely ambiguous regression where a human must
  make a *design* call — then a single targeted crop (not a full-page image) read
  into the lead context is acceptable, because the cost buys an irreducible
  judgement. Spot-checks that a one-word verdict would settle never qualify.

This is the visual-work corollary of a general agent-economy principle: large
intermediate results stay out of the orchestrator; only conclusions flow upward.

---

## Parameterization seams

| Slot | Source binding | Agnostic form |
|---|---|---|
| `«model-tell-block»` | the current model's verbatim anti-slop snippet | a short, *model-current* instruction banning today's actual tells — re-audited each tier change |
| `«house-default»` | the warm-cream / serif-display / terracotta default of the source model | whatever your current model converges on unprompted — discover it, then warn against it by name |
| `«token-source»` | the live CSS variable ramp + redesign artifacts | your authoritative, concrete source of palette/type tokens the model must cite, not reconstruct |
| `«display-font»` / `«body-font-chain»` | the project's chosen display face + reading-optimized fallback chain | your two-surface type policy (distinctive display, legible body/data) |
| `«player-facing»` | "a returning playtester notices within 30s" | your own threshold for what counts as a user-visible change |
| `«run-the-app»` | the dev-server + `dev-port` / `ensure-dev-port` scripts | however you launch a seeded instance of the running product |
| `«fixture-bridge»` | the in-page test bridge / seed-URL params | your deterministic state-injection mechanism |
| `«viewport-set»` | design-comparison screenshots: desktop 1920×1080 + mobile 390×844 (visual-regression baselines, a separate artifact, add a mid-size viewport e.g. 768×1024) | the viewports your audience actually uses |
| `«theme-set»` | light + dark | the themes you ship |
| `«snapshot»` / `«screenshot»` | accessibility-tree snapshot vs. PNG capture | your cheap-structural vs. expensive-visual capture tools |
| `«tmp»` | the `/tmp/...-ui-before/` scratch path | any out-of-repo location for ephemeral before/after evidence |
| `«rubric»` | the 7-dimension design-audit rubric | your named, severity-tagged evaluation dimensions |
| `«cycle-cap»` | max 3 edit cycles | your hard iteration ceiling before handing back to a human |
| `«verdict-subagent»` | a throwaway isolated subagent returning one line | your disposable verification context that returns a string, not pixels |
| `«stale-transform-check»` | curl the dev-server transform + restart as stopgap | your way to confirm served bytes reflect disk when a change looks invisible |

---

## What to strip

This doctrine is the *method*. The source project's *expression* is not portable —
do not carry these across, and replace them with your own where a slot calls for
it:

- **A specific palette direction** (the source uses a "parchment-and-ink,
  ledger-first" warm/bookish look). This is expression, and — pointedly — it
  nearly coincides with the source model's generic default, which is precisely why
  the source must *specify* it rather than imply it. Your project supplies its own
  `«house-default»` warning and `«token-source»`.
- **Genre / IP visual-vocabulary clauses** (the source bans a particular
  tabletop-system's iconography and pastiche). Adopt the *discipline* — invent your
  own visual grammar rather than borrowing a recognizable one — but the specific
  banned vocabulary is the source's domain, not yours.
- **Named token systems and design-artifact references** (specific CSS variable
  names, redesign-handoff docs). Point your `«token-source»` slot at your own
  authoritative tokens.
- **Concrete tooling nouns** (`dev-port` scripts, particular MCP screenshot tools,
  exact `/tmp` paths). These are the `«run-the-app»`, `«screenshot»`, and `«tmp»`
  slots — fill with your stack's equivalents.

---

## When to adopt this

- **Part 1 (model-relative anti-slop)** the moment a model produces *any*
  user-facing UI on your behalf — it is the cheapest immediate defense, and the
  one most likely to be skipped because the default *looks* fine.
- **Part 2 (look before you change)** as soon as UI changes matter enough that a
  silent visual regression would embarrass you — which is essentially always for a
  product with users.
- **Part 3 (bounded iteration)** when you delegate polish to an agent and need it
  to converge instead of fiddling indefinitely.
- **Part 4 (verdict out of context)** the first time an agent session's cost is
  dominated by reading images to answer questions a string could answer.

The non-negotiable core, if you adopt nothing else: **discover what your current
model defaults to and specify against it concretely, and never let a user-facing
change ship without someone looking at the before and the after.** The model will
always pull toward the on-distribution average, and the build gate will always be
blind to how things look. This doctrine is the two habits that counter exactly
those two facts — make the model observe instead of assume, and put every
observation where it is cheapest.
