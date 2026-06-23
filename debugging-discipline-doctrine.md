# Debugging discipline: find the cause before you change anything

> **In one line.** Replace the reflex to "try a change and see" with a four-phase
> procedure that finds a bug's root cause before touching code — and a hard stop that
> reads three failed fixes as a wrong structure, not a run of bad luck.

**TL;DR — the mechanisms**
- An iron law: no fix before a root-cause investigation — a fix proposed before the cause is named is a guess.
- Four phases, each finished before the next: root cause → pattern → hypothesis → fix.
- Instrument the boundaries of a multi-layer failure to find *where* it breaks before asking *why*.
- A three-fix rule: three failed fixes means the architecture is wrong, not the hypotheses — stop and surface it.

**Read this when** an agent (or you) is about to change code to chase a bug whose
cause isn't yet named with certainty.
**Big ideas:** [judgement stays human](OVERVIEW.md#1-judgement-stays-human-typing-goes-to-agents) · [mechanical enforcement over habit](OVERVIEW.md#5-mechanical-enforcement-over-habit)
**Depends on:** [engineering discipline](engineering-discipline-doctrine.md) (failure ownership, the regression-test-on-fix rule) · [working with the harness](claude-code-harness-doctrine.md) (the run-until-verifiable loop is the mechanical cousin of phase 4)

---

How to debug under an agent without the debugging itself becoming a source of damage —
by converting the *habit* of "change something and re-run" into a *procedure* that
locates the cause before it changes anything, and by naming the point at which
continuing to patch is itself the bug. The premise is that a capable model, handed a
failing test or a stack trace, will reach for a plausible edit immediately — and a
plausible edit aimed at a symptom masks the real defect, often while breeding a second
one. The cost of a wrong fix is not just the wasted turn; it is a codebase that now has
two problems wearing one set of clothes.

This is project-agnostic doctrine extracted from a working discipline (itself adapted
from a published debugging skill). It introduces almost no project nouns: its one slot
is `«gate»` (your validate step — lint + typecheck + test, defined in the
[engineering doctrine](./engineering-discipline-doctrine.md)). Where your project has a
*runtime-specific* debugging checklist — a browser/e2e harness with its own state-
interrogation steps — treat this as the **general** discipline behind it: reach for the
runtime checklist first for that class of failure, and fall back to the four phases here
for everything it doesn't cover (logic bugs, build failures, data-shape mismatches).

---

## Why this shape

The reflex this counters is specific and near-universal under an agent: *read the error,
guess a cause, edit, re-run.* It feels efficient because each cycle is fast. It is not,
for three compounding reasons:

1. **A symptom patch masks the defect.** The edit makes the visible failure go away
   without addressing why the bad value existed, so the defect resurfaces later, further
   from its origin and harder to trace. The fast cycle bought a slower one.
2. **Stacked guesses entangle.** When the first guess doesn't work and a second is laid
   on top without reverting the first, the system now carries two speculative changes
   whose interaction nobody reasoned about. Three of these deep and the failure mode is
   no longer the original bug.
3. **The agent declares victory on green.** A model under pressure to pass will accept
   the first green it sees — including a green it reached by weakening the check rather
   than fixing the code (see the engineering doctrine's silent-pass blind spots).

The defense is to make the *investigation* mandatory and *ordered*, so that a fix is
the **last** step, taken once, against a cause that has been named and confirmed — and
to install a circuit-breaker that fires when the procedure itself stops converging.

> **The iron law.** No fix without a root-cause investigation first. A fix proposed
> before the cause is found is a guess; a guess that lands looks exactly like a fix and
> is trusted like one. Find *why* it breaks before you change *what* breaks.

---

## Part 1 — the four phases, each finished before the next

The procedure is four phases run in order. The ordering is the discipline: each phase
produces the input the next one needs, and skipping ahead is how the reflex
re-establishes itself.

**Phase 1 — root cause.** Read the error and the stack trace *completely* — line
numbers and error codes frequently name the fix outright, and the reflex skips them.
Reproduce the failure reliably; if you cannot reproduce it, that is a data-gathering
problem, not a licence to guess at a fix. Check what changed — the recent diff, new
dependencies, the last commits to the failing path. For a failure that crosses layers
(UI → application → domain, or CI → build → deploy), **instrument each boundary** — log
what enters and leaves each layer — and run once to see *where* the value goes bad
before investigating *why*. Then trace the bad value backward to its origin and fix it
**at the source, not where it surfaced.**

**Phase 2 — pattern.** Find working code in the same codebase that does the
similar-but-correct thing, and list *every* difference between the working path and the
broken one, however small. "That can't matter" is precisely where the cause hides — if
it truly didn't matter, the two paths wouldn't differ on it. If you are following a
reference (a doc, an example, another module's approach), read it *completely* rather
than skimming the part that looks relevant; the relevant part is often the part that
looked like boilerplate.

**Phase 3 — hypothesis.** State one cause, in one sentence, with its evidence: "I think
X because Y." Test it with the **smallest possible change, one variable at a time.**
Confirmed — the symptom moves exactly as the hypothesis predicts — advance to phase 4.
Not confirmed — discard it and form a new hypothesis; **do not** stack a second change
on top of an unconfirmed first. When you don't understand something, say so plainly
rather than papering the gap with a confident-sounding edit.

**Phase 4 — fix.** Write a failing test that reproduces the bug *first* — every bug fix
earns a regression test that proves the fix holds and fails without it (the engineering
doctrine's "when a test is owed"). Make **one** change addressing the root cause: no
"while I'm here" refactor, no bundled cleanup, nothing the cause didn't demand. Then
verify with **fresh evidence** before claiming it fixed — run the covering checks, read
the output, confirm the symptom is gone *and* nothing adjacent broke. A remembered
green from two edits ago is not evidence; confidence is not evidence. (This is the
debugging-time face of the corpus-wide rule that a completion claim needs proof, not
assertion — the engineering doctrine carries the general posture, the harness doctrine
its mechanical enforcement.)

---

## Part 2 — the three-fix rule

The circuit-breaker. **If three fixes have failed, stop — do not attempt a fourth.**

Three failures of a specific kind are the signal: each fix resolved its target but
surfaced a new problem somewhere else, or each one demanded "just a bit of refactoring"
that grew. That pattern is not a run of unlucky hypotheses — it is the shape a **wrong
architecture** makes when you push correct local reasoning through it. Each fix is right
about its local cause and wrong about the structure that keeps generating local causes.

So the question changes. It is no longer "what is the fix"; it is "is this structure
sound, or are we continuing through inertia?" — and that is a judgement call to hand
back to the human, not one for the agent to settle by attempting fix four. This is the
same escalation posture as a failing test you can't explain (the engineering doctrine's
*surface-it* response): stop, state what the three attempts were and what each one
broke, and let the one judgement-holder decide whether to keep patching or rethink the
structure. The three-fix rule is what keeps a debugging session from quietly turning
into an unplanned, unreviewed rewrite.

---

## Red flags — stop and return to phase 1

Each of these is the reflex re-asserting itself mid-procedure. Catching the *thought* is
the intervention:

- "Quick fix now, investigate later." — the investigation is the fix; later never comes.
- "Just try changing X and see if it works." — name the cause first; an edit you can't predict the result of is a guess, not a fix.
- Changing several things at once, then re-running. — you've given up the one-variable
  test that tells you *which* change mattered.
- "It's probably X" — a cause asserted before the data flow was traced.
- "One more attempt" after two have already failed. — the on-ramp to the three-fix wall;
  count the attempts honestly.
- Each fix surfaces a new problem in a different place. — the architecture signal; invoke
  the three-fix rule.

---

## Parameterization seams

This chapter is almost slot-free by design — the discipline is cognitive, not
configured. The one binding:

| Slot | Source binding | Agnostic form |
|---|---|---|
| `«gate»` | lint → typecheck → test | the validate step phase 4 runs to confirm the fix with fresh evidence (defined in the engineering doctrine) |

Two **companions** rather than slots: a *runtime-specific* debugging checklist (a
browser/e2e harness's state-interrogation steps), used first for its class of failure
with these four phases behind it; and the *failure-ownership* posture from the
engineering doctrine, which decides what to do with a failing test that isn't the one
you came to fix (fix / capture / surface — the three-fix rule routes to *surface*).

---

## When to adopt this

Adopt it the first time an agent "fixes" a bug by changing what the error pointed at,
the symptom moves, and the real defect resurfaces a week later wearing a different
stack trace — which is to say, almost immediately once an agent is allowed to debug
unsupervised. The four phases cost a few minutes of investigation the reflex wants to
skip; they buy back the hours a stacked-guess session costs and the trust a
masked-defect codebase loses.

The non-negotiable core, if you adopt nothing else: **no fix before the cause is named,
and three failed fixes means stop and question the structure.** The first kills the
guess-and-see reflex at the front of the procedure; the second catches the case where
the procedure runs correctly and the architecture, not the hypothesis, is what's wrong.
Everything between them — the boundary instrumentation, the difference list, the
one-variable test, the regression test — is how you get from a symptom to a cause
without changing anything you'll later have to explain.
