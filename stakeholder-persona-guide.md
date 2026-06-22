# How to instantiate and dispatch a stakeholder-persona reviewer

> **In one line.** Build a roster of read-only persona reviewers and deploy them in a
> review — when to spawn, what they may see, how to keep them honest, what to run them on.

**TL;DR — the mechanisms**
- Build a roster of archetypes, each a narrow read-only reviewer.
- Three spawn modes: design review, pre-release sweep, targeted check.
- Read-scope discipline is load-bearing — a persona sees only the surface, not the code.
- Anti-noise rules and a refuter keep them from rubber-stamping.
- A clear boundary with craft reviewers: they ask "would I care?", not "is it well-built?"

**Read this when** you want engagement and UX defended in review, not just correctness.
**Big ideas:** [defend the experience in review](OVERVIEW.md#7-defend-the-experience-in-review)
**Depends on:** [persona template](stakeholder-persona-template.md) (the form each persona fills)

---

Companion to `stakeholder-persona-template.md`. The template tells you how to
*author* one persona; this guide tells you how to build a **roster** of them
and *deploy* them in a review — when to spawn, what to let them see, how to keep
them honest, and what to run them on.

This is project-agnostic. Substitute your own domain's surfaces wherever a slot
appears.

---

## The shape of the pattern

A stakeholder-persona reviewer is a **read-only agent** that reviews an artifact
in the first-person voice of one archetypal user. You run two or three of them
*beside* your craft reviewers. Craft reviewers protect quality; personas protect
engagement. The deployment has five moving parts:

1. **A roster** — one persona per archetype that matters to your product.
2. **A selection rule** — which personas to spawn for a given artifact.
3. **A read-scope discipline** — each persona sees only what its user sees.
4. **A refuter pairing** — a check that the persona found real friction, not
   invented friction.
5. **A model/cost tier** — personas are cheap, stable lenses; size them so.

---

## 1. Building the roster

Author one persona (via the template) per **distinct user archetype whose
failure mode your craft reviewers structurally cannot see.** A good roster is a
*menu*, not a committee — you will never run all of them at once.

Keep archetypes **mutually exclusive by concern.** Each persona's "what you
ignore" section should hand off, by name, to another persona's lane. Overlap is
where drift breeds: if two personas both review "the first-run experience," they
collapse into one generic voice. Some archetypes worth a seat in most products:

| Archetype | Hunts for | Quits when |
|---|---|---|
| **Impatient newcomer** | "What do I do right now?" | Opacity in the first minutes |
| **Power user / optimizer** | Hidden levers, dominant strategies, edge cases | Outcomes feel arbitrary or capped |
| **Returning/lapsed user** | "Where was I?" re-entry cues | Has to relearn after time away |
| **Completion-driven user** | Legible "have I done everything?" surfaces | Progress is opaque or hidden |
| **Voice/quality-sensitive user** | Tone, coherence, authenticity | Filler, inconsistency, generic phrasing |
| **Accessibility-dependent user** | Keyboard path, contrast, screen-reader order | A surface is reachable only one way |

Pick the archetypes that match *your* product's actual risk surface. A
checkout flow needs the impatient newcomer and the returning user; a developer
API needs the power user and the voice/quality reviewer; a clinical form needs
the accessibility-dependent and completion-driven users. The roster is yours to
define — the *generator* is what's reusable.

---

## 2. When to spawn — three modes

### Design-lock review (the default seat)

Before committing to a design, your review wave is some mix of craft reviewers.
**Replace one or two craft seats with persona seats.** A typical wave: 3 craft
specialists + 1–2 personas whose lens matches what the artifact risks. Both
lenses feed the synthesizer (you).

Match personas to the artifact:

| Artifact under review | Persona seats to add |
|---|---|
| First-run / onboarding | Impatient newcomer |
| Mechanics / pricing / levers | Power user / optimizer |
| Content / copy / narrative | Voice/quality-sensitive user |
| Persistence / accounts / re-entry | Returning user, completion-driven user |
| Identity / personalization | The archetype whose identity is on the line |

### Pre-release sweep (recurring, cheap)

Before sealing a release, cycle two or three personas across the surfaces you
changed, each matched to its lens. Generate a friction list; triage it into your
backlog. No infrastructure, repeatable every release.

### Targeted spawn (asymmetric)

When a single piece of real user feedback flags a perspective gap, spawn the one
matching persona on the implicated surface. A "this felt arbitrary" report →
power user. A "I bounced immediately" report → impatient newcomer.

---

## 3. Read-scope discipline (the load-bearing rule)

**A persona sees only what its real user would see.** This is not a politeness
constraint — it is what keeps the output valid. The moment a persona reads the
source code, the internals, or the other specs, it stops reacting like a user
and starts reasoning like a builder, and its feedback collapses into craft
critique.

So every dispatch gives a persona:

- the artifact / surface under review;
- a **rendering** of it — a screenshot, a transcript, a sample output. If your
  artifact is visual and you can't show it a rendering, the persona cannot
  review it; produce one first. A persona reacting to a *description* of a
  screen instead of the screen is guessing.
- at most **one** design-context document, read once, for lens calibration.

And withholds: the source, the internals, the issue tracker, sibling specs.

---

## 4. Keep them honest — the anti-noise disciplines

A persona that *always* finds friction is noise. The signal is a persona with a
strong, specific perspective that lands on a surface and reports it's *fine for
that audience.* Three disciplines protect the signal:

- **The honesty clause.** Every brief's output format ends with "what worked for
  me — don't manufacture friction." Enforce it: a persona that never credits
  anything is miscalibrated.
- **The refuter pairing.** Don't average concurring opinions — pair a *refuter*
  with the finding. After a persona reports friction, a second pass (a skeptic,
  or the persona re-run with "try to prove this friction isn't real for your
  archetype") tests whether the friction survives. Friction that survives a
  refutation is real; friction that evaporates was invented. This matters more
  than a second concurring persona.
- **Calibration against ground truth.** Persona output is unfalsifiable inside a
  single review. Before you *trust* a persona as a standing review seat,
  calibrate it: run it against a surface where you already have real user
  feedback, and score it on two axes —
  - **Hit rate:** did it surface ≥1 friction real users actually reported?
  - **Noise rate:** did it invent friction no real user reported and the team
    hadn't already flagged?

  A persona that fails calibration twice gets its brief rewritten or retired.
  The roster is not sacred.

---

## 5. Model tier and cost

Personas are **stable, narrow lenses, not deep reasoners.** Holding a fixed
archetype and reacting to a surface is a mid-tier task — it does not need your
most capable (most expensive) model. Assign personas a **mid or economy model
tier** by default and reserve top-tier models for the synthesis and the craft
reviewers who do cross-cutting reasoning. Upgrade a specific persona's tier only
if its output is visibly weak.

Because they're cheap and read-only, personas are also safe to **fan out** past
the size cap you'd put on agents that write — a read-only review wave produces
findings, not edits, so there's no shared-state hazard in running a large panel.
Cost is then the only gate; size the panel to the artifact's risk.

---

## 6. Boundary with craft reviewers

When a persona and a craft reviewer review the same artifact:

- **The craft reviewer owns quality questions** — "is this layout right? is the
  voice consistent? is the curve correct?"
- **The persona owns engagement questions** — "would I read this? would I push
  this lever? would I quit here?"
- **Disagreement is signal, not conflict.** A persona finding friction the craft
  reviewer judged acceptable is exactly the gap the persona exists to catch. The
  synthesizer reconciles the two; the persona never gets overruled *as a user* —
  it reported a real reaction.

Personas have no write scope. If a persona's reaction implies a change, the
synthesizer routes that change to whoever owns the surface. The persona's job
ends at the honest reaction.

---

## Checklist — instantiating a persona from scratch

1. Identify an archetype whose failure your craft reviewers can't see.
2. Confirm it doesn't overlap an existing persona's lane (check "what you
   ignore" across the roster).
3. Fill every `«slot»` in `stakeholder-persona-template.md` — specific axiology,
   vivid backstory, concrete quit triggers.
4. Decide single-pass vs. two-pass (does this archetype experience the artifact
   at more than one moment of its journey?).
5. Set the model tier (mid/economy by default) and read-only tooling per your
   harness.
6. Validate by re-deriving a persona you already trust (see the template
   appendix) — if the template can't reproduce it, a slot is missing.
7. Calibrate against a surface with known real-user feedback before granting it
   a standing review seat.
