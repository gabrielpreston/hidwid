# Rule-authoring doctrine: evidence it, form it, enforce it

> **In one line.** A rule earns its place three ways — it prevents a *named real
> failure*, it takes the *form* its failure responds to, and when it is correct but
> still skipped it is *hardened* against the excuse the moment generates.

**TL;DR — the mechanisms**
- Evidence gate: a new rule names the observed failure it prevents, falsifiable by "did that actually happen?" — never a rule by assertion.
- Form match: classify the failure as discipline (→ prohibit) or shaping (→ positive recipe); a prohibition backfires on a shaping failure.
- Enforcement hardening: a correct rule that's still drifted from earns a rationalization-vs-reality table — reserved for the rules actually skipped, not every rule.
- The thread: a rule costs a re-read every turn, so each gate makes that standing cost buy something real.

**Read this when** you're writing a rule, or a rule you already wrote isn't changing
the behavior you wanted.
**Big ideas:** [mechanical enforcement over habit](OVERVIEW.md#5-mechanical-enforcement-over-habit) · [defend the experience in review](OVERVIEW.md#7-defend-the-experience-in-review)
**Depends on:** [cost hygiene](cost-hygiene-doctrine.md) (the per-turn tax a rule must earn back) · [prose anti-slop](prose-anti-slop-doctrine.md) and [frontend anti-slop](frontend-anti-slop-doctrine.md) (the recipe family Part 2 explains)

---

This is the **meta-doctrine**: rules about authoring rules. Most of the corpus tells you
*what* discipline to enforce; this tells you how to write the enforcement so it actually
holds, because a rule is not free. Every always-loaded rule is re-read every turn — the
per-turn context tax the [cost-hygiene doctrine](./cost-hygiene-doctrine.md) measures — so
a rule that doesn't pull its weight isn't neutral, it is standing cost. The three parts
below are the three ways a rule earns that cost back: it must prevent a *real* failure
(Part 1), it must be *shaped* for the kind of failure it targets (Part 2), and if it is
right yet still skipped it must be *hardened* against the rationalization that skips it
(Part 3). Part 2 also explains the corpus's own split — the engineering gates are
prohibitions, the anti-slop docs are recipes — as no accident.

Where it points at a concrete rubric it borrows the [anti-slop](./prose-anti-slop-doctrine.md)
slot (`«failure-modes»`); otherwise it is slot-free, because the doctrine is judgement,
not configuration.

---

## Why this shape

A rule constrains what an agent (or a person) produces, and a *bad* rule fails in one of
three ways — each the inverse of a part below:

1. **Written by assertion.** The author saw something they didn't like, or imagined one,
   and wrote "don't do that" with no real failure behind it. It reads as sound and costs
   a turn-tax forever while preventing nothing. *Defense: the evidence gate (Part 1).*
2. **Right intent, wrong form.** The failure is real but the rule is shaped wrong — a
   prohibition aimed at a failure that prohibitions don't fix. It reads as reasonable and
   the behavior simply doesn't change. *Defense: the form match (Part 2).*
3. **Right rule, ignored.** The rule is evidenced and well-formed and still gets skipped
   under pressure, because the moment generates an excuse its calm prose never answers.
   It is present and inert. *Defense: enforcement hardening (Part 3).*

The order is the lifecycle of a rule: you decide whether to write it, you choose its
form, and — only if it later proves skippable — you harden it.

---

## Part 1 — evidence: name the failure, don't assert the rule

**A new rule must name the observed failure it prevents, and that name must be falsifiable
by a single question: *did the cited failure actually happen?*** Cite a real incident —
a session that went wrong, a *recorded* piece of feedback (a named entry, not a vague
memory of annoyance), a tracked ticket. A rule that can only offer "this is good practice,"
with no failure behind it, fails the gate and does not ship.

The bar is *evidence of a failure*, not a quality argument, and the reason is the tax:
because every resident rule is re-read every turn, an unevidenced rule is pure standing
cost, charged against every future session. So the question at
authoring time is never "is this a good idea" (everything sounds like a good idea); it is
"did its absence actually cost something." If you can't answer concretely, you have a
preference, not a rule.

**The minimal-rule corollary.** Write the smallest rule that addresses *exactly* the
observed failure. No speculative coverage for adjacent failures you have not seen — those
earn their own evidence when they occur. A rule padded with "while we're at it" clauses is
mostly unevidenced rule, paying tax for failures that never happened.

**The rigorous form, for a high-stakes rule: reproduce the failure empirically.** When you
cannot point at a past incident, or you want to confirm a costly rule actually changes
behavior, run it as a RED→GREEN→REFACTOR pass:

- **RED.** Dispatch a *fresh subagent* — one that never saw the rule — on the realistic
  pressure scenario the rule is meant to govern, with the rule unloaded. The isolation is
  the point: an author who already knows the target behavior can't run the test without
  contaminating it. Give the subagent the real task, not a leading prompt — you are
  observing the default, not coaching the failure into existence — and capture verbatim
  what it does wrong. If it does *not* fail, read that through the honesty clause below:
  when you have *no* past incident to fall back on, a non-reproducing scenario means you
  have no evidence the rule is load-bearing — either it is cargo cult (don't write it) or
  the scenario lacked real pressure (sharpen it and re-run). A rule already backed by a
  real past incident stands on that evidence; one empirical run that happens not to
  reproduce does not retract it.
- **GREEN.** Write the minimal rule targeting exactly that failure.
- **REFACTOR.** Re-run the same scenario *with* the rule, confirm the behavior changed,
  and tighten the wording until it holds.

**The honesty clause — load-bearing.** Agent behavior is nondeterministic, so a single RED
run is *not* a deterministic unit test: a rule can pass by luck, and a real failure can
fail to reproduce on one attempt. The empirical pass is therefore **corroborating evidence
that a rule is load-bearing, not a pass/fail gate you can game** by re-running until you
get the result you want — nondeterminism defeats every stopping rule. That is
exactly why the *mandatory* artifact is the cheaper, falsifiable one — a named, real
observed failure — and the empirical pass stays optional, reserved for the rules worth the
effort. Use it to build confidence; never to manufacture a green you can point at.

---

## Part 2 — form: match the rule's form to its failure

A rule constrains in one of two ways: forbid the wrong output (*"don't do X"*) or specify
the right one (*"produce Y; here is what Y is"*). The instinct is the prohibition every
time — it is shorter and it names what you just saw go wrong — and for one class of failure
it is correct. For another it **backfires**, quietly, while reading as reasonable.

### Classify the failure first

The fork is the *kind* of failure the rule targets:

- A **discipline failure** is a known-correct action that was omitted: the gate wasn't
  run, a layer boundary was crossed, a commit skipped its tests. The actor wasn't confused
  about the right move — they skipped it. The space of correct behavior has a *single known
  fill*: do the step.
- A **shaping failure** is the wrong *kind* of output where no single right output exists:
  generated prose that reads as competent and says nothing, a UI that converges on the
  model's house style, a patch note that softens a change into evasion. Nothing discrete
  was omitted; the output landed in the wrong region of a taste continuum.

| | Discipline failure | Shaping failure |
|---|---|---|
| **Looks like** | a known step omitted | the wrong *kind* of output on a taste continuum |
| **Right form** | **prohibition / gate** — ideally mechanically enforced | **positive recipe** — *"produce Y; here is what Y is"* |
| **Why** | the correct fill is known, so removing the wrong option suffices | no single correct fill, so a negative relocates the failure and can grow a new tell |
| **Lives in** | [engineering discipline](engineering-discipline-doctrine.md), the harness gates | the [anti-slop](prose-anti-slop-doctrine.md) docs, the persona-reviewer recipes |

A reliable test for the miscast: **if a lint check could enforce the rule, it is a
discipline failure and a prohibition is right.** If no mechanical check could ever decide
whether the output passed — because the judgement is one of degree — it is a shaping
failure, and a prohibition will quietly fail to move the behavior.

### Why a prohibition backfires on a shaping failure

A prohibition removes one option from the space the actor must still fill. That is exactly
enough for a discipline failure — the space has a known-correct fill, so forbidding the
wrong move leaves the right one. It is *not* enough for a shaping failure, for two reasons
that compound:

1. **The negative relocates the failure.** Forbid one on-distribution default ("don't use
   the obvious palette") and the model moves to the *next* default, which you didn't name.
   Subtracting one wrong fill from a space with no single right fill just selects a
   different wrong one; you can play whack-a-mole down the distribution forever.
2. **The cure becomes its own tell.** A prohibition against a *manner* ("don't sound
   robotic," "don't fake humanness") invites the model to *perform the opposite* — forced
   casualness, hedges, performative imperfection — which reads as generated just as fast.

### Lead with the recipe; demote the negative

The conversion is not "delete the prohibition" — a negative still cheaply flags the
generic case the positive spec didn't pin. The move is to **invert the emphasis**: the
recipe leads, the negative trails as a secondary drift-catch. Four shapes the recipe
takes, each drawn from a real conversion of a backfiring prohibition:

- **A substitution map, not a test.** Replace *"don't use reaching verbs"* with a
  named-replacement table (*surfaces → appears, settles → sits*), keeping the original
  test only as the keep-condition on the replacement.
- **A target-state description, not a manner ban.** Replace *"don't fake humanness"* with
  a read-for-it check: read each sentence and ask whether it is *trying to seem* a certain
  way — the passing sentence states its concrete thing and stops.
- **Specify the spec, then propose-then-pick.** Replace *"avoid the generic look"* with
  the exact spec (cite the tokens, the colors, the typefaces) and, where none is locked,
  have the agent offer three or four distinct directions and pick one.
- **A declarative template, not a register ban.** Replace *"don't write evasive notes"*
  with the template the good note follows; the banned phrasings move into a parenthetical
  *example* of what the template excludes.

### This is why the corpus has two rule families

The split is visible across the whole corpus, and it is the strongest evidence the
classification is real rather than stylistic. The **prohibition family** — the
engineering gates, the layer-boundary lint, the harness permission rules — targets
discipline failures and is (or could be) mechanically enforced. The **recipe family** —
the prose and frontend anti-slop docs, the persona-reviewer template — targets shaping
failures, and none is a wordlist-of-bans at its core. When you write a new rule you are
choosing which family it joins, and the classification above is how you choose correctly
instead of defaulting to the prohibition because it was the faster sentence to write.

---

## Part 3 — enforcement: harden the rule the excuse skips

The third failure is the subtlest: a rule that is evidenced (Part 1) and correctly formed
(Part 2) and *still gets skipped* — not because anyone disagrees with it, but because the
moment generates an excuse its calm declarative prose never answers. **The tell that you
are here is repetition: the same rule keeps being re-asserted** — a second incident, a
new note saying the thing the rule already says — which means the prose isn't holding
against the rationalization the situation produces. Externalize that signal — a note, a
record, a named incident — so a re-assertion survives across sessions; a repetition no one
catches never triggers the hardening.

This is a *drift* problem, not a *never-existed* one, and it takes a different fix than
Part 1. More evidence won't help — the rule is already evidenced. The fix is to **harden
the prose against the specific excuse**, with three elements:

- **A rationalization-vs-reality table.** Two columns: the *rationalization* (the excuse in
  the words the moment actually produces — *"the suite came back green, so the coverage
  held"*; *"the subagent reported success, so it's done"*) and the *reality* (the counter —
  *"green is the cheapest path, not proof; refute the diff, not the pass count"*; *"a
  subagent's report is a claim, not evidence — re-verify it"*). These are representative,
  not a closed set: source each rationalization *verbatim from where the violation was
  recorded*, never from imagination, because the excuse that actually gets used is the only
  one worth answering.
- **A red-flags STOP list.** The thoughts that precede the skip, written as catch-yourself
  triggers ("reaching for a second concurring reviewer instead of one refuter").
- **An omission clause.** The blunt statement that a skipped gate is a skipped gate: *"if
  you skipped the refuter, the step did not happen — even when the work looks done."*

Why a table and a STOP list rather than firmer declarative prose: the calm register is
*exactly* what the in-the-moment excuse slips past. Naming the excuse in its own words,
beside its refutation, is what catches a reader rationalizing in real time — which is
itself the Part 2 move, a recipe for the reader's behavior rather than a louder
re-prohibition.

**Reserve it for the rules actually skipped.** Hardening every rule this way is weight
without cause, and it dilutes the signal: the table is supposed to mean "this one really
does get skipped." A rule earns its rationalization table by the same discipline that
earned its existence in Part 1 — evidence — but the evidence here is necessarily
*retrospective*: a second real violation, which, unlike Part 1's failure, you cannot
simulate ahead of time. Quality, not weight.

---

## Parameterization seams

Near slot-free by design — the doctrine is a set of judgements, not configuration. Its one
borrowed binding:

| Slot | Source binding | Agnostic form |
|---|---|---|
| `«failure-modes»` | the anti-slop structural rubric | your shaping-failure recipe (defined in the [prose anti-slop doctrine](prose-anti-slop-doctrine.md)); the recipe family Part 2 sends shaping failures to |

---

## When to adopt this

- **Part 1 (evidence)** the moment your rule set is big enough to feel like overhead — the
  unevidenced rules are why it does, and the gate is what stops the next one from landing.
- **Part 2 (form)** the first time a "don't do X" rule reads as sound and changes nothing —
  the diagnostic that X is a shaping failure that needed a recipe.
- **Part 3 (enforcement)** the first time you find yourself writing the same rule a *second*
  time because the first was ignored — that repetition is the signal to harden, not to
  restate.

The non-negotiable core: **a rule prevents a named real failure, takes the form its failure
responds to, and earns a rationalization table only once it has actually been skipped.**
The unifying thread under all three is the tax — a rule is a standing per-turn cost, so
each gate is a way of making sure that cost buys something: a real failure prevented, in a
form that works, enforced only where enforcement is owed.
