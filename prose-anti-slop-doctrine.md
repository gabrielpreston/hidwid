# Prose anti-slop: name the failure modes, read before you write

> **In one line.** Keep agent-produced prose from converging on the model's
> competent, profound-sounding default — by naming the structural ways generated
> text fails and forcing every authored surface to be a read-aloud one.

**TL;DR — the mechanisms**
- A rubric of **structural failure modes** — durable across model generations.
- A **lexical appendix** of banned words and tics — quarantined, re-audited each tier.
- A **read-aloud pass**: name the empty clause before you edit, not after.
- **Per-voice calibration** so the rubric doesn't flatten registers that differ on purpose.

**Read this when** an agent is producing user-facing prose you don't want to read
as generated.
**Big ideas:** [defend the experience in review](OVERVIEW.md#7-defend-the-experience-in-review)

---

This is the **prose companion** to the
[frontend anti-slop doctrine](./frontend-anti-slop-doctrine.md). That doc keeps
agent-produced *UI* off the model's house style; this one keeps agent-produced
*text* off it. The premise is the same: a capable model, left to its own judgement,
writes prose that is competent, plausible, and **on-distribution** — which is
exactly the problem. It reads as intentional while being the average of everything
the model has seen, and its most recognizable tell is the **clause that sounds
profound and says nothing concrete.**

The build gate cannot catch this. Lint, types, and tests verify behavior; none of
them can tell you a sentence performs insight without carrying any. Prose quality
has no green checkmark — it has a disciplined reading-loop instead, the same way
visual quality has a looking-loop.

Wherever a concrete noun appears — a voice name, a surface, a banned word — it is a
**`«slot»`** you fill for your own project; the
[parameterization table](#parameterization-seams) collects every slot, and the
[what-to-strip](#what-to-strip) section lists the source project's expression so
you don't inherit it by accident.

---

## Why this shape

Three observations drive the parts below.

1. **The tell is structural, not lexical — but everyone reaches for the wordlist.**
   The instinct is to ban "delve," ban the em-dash, ban "tapestry." Those lists are
   real, but they staledate fast (a model generation later the words have moved)
   and they create false confidence: text with none of the banned words can still
   be pure slop. The durable contract is the **shape** of the failure — an empty
   aphorism, a manufactured triad — which survives across model versions because it
   is how the model *reasons* about "good writing," not which tokens it happened to
   pick.

2. **A failure mode on one surface is the correct register on another.** Flat,
   uniform cadence is a slop tell in literary prose and exactly right in a terse
   status line. A rubric applied uniformly will "fix" the surfaces that were already
   correct. The rubric must be **calibrated per voice**, naming where each failure
   mode is in force and where it is the intended register.

3. **The cure can become its own tell.** Faking humanness — forced typos, false
   uncertainty, performative casualness — reads as generated just as fast as the
   profundity it replaced. Over-correction is slop in another form. The fix is
   concreteness and honest cadence, never manufactured imperfection.

---

## Part 1 — the structural failure modes are the durable contract

Name the failure modes once; apply them everywhere prose is authored. Each gets a
one-line definition, a BAD exemplar, a FIXED exemplar, and the curing lever. Score
each **binary — pass or fail** per mode; severity tiers are over-engineering for a
rubric whose first mode is near-always a cut.

The `«failure-modes»` set that has proven durable:

- **Empty aphorism / false profundity** — a clause that performs insight with no
  concrete referent. *The signature tell.* Lever: swap the abstraction for the
  event; **if the referent can't be named, cut the clause.** The test is "what does
  that even mean?" — if you can't answer concretely, neither can the reader.
- **Strained impact diction** — reaching verbs (*surfaces, settles, curates,
  weaves*) where a plain one (*appears, lands, picks, lists*) is honest. Lever:
  would I use this verb about my own morning? If not, downgrade it.
- **False antithesis / manufactured triads** — "not X but Y" where there was never
  an X; a rule-of-three built for rhythm, not content. Lever: collapse it; state
  the fact once. A genuine requirement list is not a triad.
- **Decorative metaphor** — imagery the reader must decode back to a literal that
  was shorter. Lever: if translating the image to plain words takes *more* words,
  cut it.
- **Machine-flat cadence** — uniform sentence length, every clause the same weight,
  no short punches. Lever: after a long sentence, throw a short one. Read it aloud;
  if you never pause for breath, vary the lengths.
- **Abstract category for a concrete scene** — a category noun ("resilience in the
  face of adversity") standing in for the event that earned it ("three contracts
  failed; the fourth paid double"). Lever: replace the category noun with the
  specific event.

This last mode pulls the **same way** as an invented-naming discipline, not against
it: where you need a concrete place or thing, *invent* it rather than retreating to
a generic abstract. The concrete-over-abstract lever and a no-borrowed-nouns rule
are allies — both push toward a specific, owned noun.

---

## Part 2 — quarantine the lexical tells in a re-auditable appendix

A wordlist still earns a place — as an **appendix, explicitly marked
re-audit-each-tier**, never as the contract. Keep here the current model's actual
lexical tells: overused words (`«banned-words»`), punctuation habits
(`«banned-tics»` — e.g. a particular model's em-dash or triple-colon reflex), and
opener/closer formulas. Date the list. When the model tier changes, re-audit it
the same way you re-audit a visual anti-slop block: a stale list is worse than no
list, because text that passes it *feels* checked.

**Structural core = permanent; lexical appendix = perishable.** Carry both, but
never let the perishable half stand in for the durable one.

---

## Part 3 — the read-aloud pass (look before you write)

Prose has no screenshot, so the "look" is a read. Before authoring or committing a
prose surface, **read it through and name any empty clause before you edit.** For
each sentence: does it name a real thing that happened (vs. an empty aphorism or an
abstract category)? Is every verb one you'd use unselfconsciously? Do the lengths
vary? This is the prose equivalent of capturing the before-state of a UI — a
required look that *precedes* the edit, not a review bolted on after. It costs a
minute and catches most failure modes on sight.

Embed this pass where the authoring happens; do not spin it into a separate
ceremony document. The pass is ~one paragraph of obligation, not a sibling file —
a dedicated file would be ritual with no mechanism behind it.

---

## Part 4 — calibrate per voice, and review the substantive batches

One rubric, but applied through a per-voice lens. For each `«voice»` you ship
(a terse status register, an expansive narrative register, an instructional
register), state which failure modes are in force and which are the *correct*
register — so a reviewer never "fixes" a flat status line into florid prose, and
never lets a florid paragraph hide behind "that's just the voice."

Gate weight scales with stakes. A trivial single-line tweak leans on the
auto-loaded rubric alone. A **substantive batch** — a new prose pool, a template
addition, a corpus page — earns a dedicated review pass with two distinct stances:
a **reader** ("did it land? does the voice hold?") and a **critic** ("find the
empty clauses, against the rubric"). The two are different jobs; one seat cannot do
both, because the reader's immersion is exactly what the critic must suspend.

---

## Parameterization seams

| Slot | Source binding | Agnostic form |
|---|---|---|
| `«failure-modes»` | the six structural modes (empty aphorism, strained diction, false antithesis/triad, decorative metaphor, flat cadence, abstract-for-concrete) | your durable structural rubric — the shapes generated text fails in, model-version-independent |
| `«banned-words»` / `«banned-tics»` | the current model's overused words + punctuation reflexes | your perishable lexical appendix, dated and re-audited each model tier |
| `«voice»` | the project's distinct registers (terse status / narrative / instructional) | the voices you ship, each with its own in-force / relaxed failure modes |
| `«reader-seat»` / `«critic-seat»` | a persona reader + an authoring-agent critic in the review wave | your two review stances — immersion check vs. adversarial slop-hunt |
| `«authoring-agent»` | the prose-owning agent that cites the rubric by path | whichever agent owns player-facing prose and inherits the rubric as ownership |
| `«gen-tier»` / `«critic-tier»` | cheap model for generation, one step up for the rubric-check pass | your model-tier split: routine generation vs. analytical slop-hunt |

---

## What to strip

This doctrine is the *method*. The source project's *expression* is not portable:

- **The specific exemplars.** The BAD→FIXED pairs in the source rule are written in
  that project's nouns and voices. Keep the *failure mode*; rewrite the examples in
  your own surfaces, because an exemplar only teaches when it is in a register your
  authors actually write.
- **The voice roster.** A project's particular set of registers (and which failure
  mode each relaxes) is its own taxonomy. Adopt the *per-voice calibration move*;
  build your own table.
- **The lexical appendix contents.** Today's banned words are model-current and
  project-flavored. The *discipline* (quarantine + date + re-audit) transfers; the
  wordlist does not.
- **Genre / IP naming clauses.** If your project bans borrowed proper nouns, that
  reserved-name table is its own domain — the concrete-over-abstract lever points at
  it but does not contain it.

---

## When to adopt this

- **Part 1 (structural rubric)** the moment a model writes *any* user-facing prose
  on your behalf — it is the cheapest defense and the one most likely skipped,
  because the default *reads* fine.
- **Part 2 (lexical appendix)** alongside Part 1, but never instead of it — and
  re-audited every time the model tier moves.
- **Part 3 (read-aloud)** as soon as prose ships to readers who would notice it
  feeling generated, which is essentially always for a text-forward product.
- **Part 4 (per-voice + review wave)** when you author prose in more than one
  register, or in batches large enough that a silent slop regression would embarrass
  you.

The non-negotiable core, if you adopt nothing else: **name the structural failure
modes and read every authored surface aloud before it ships.** The model will
always pull toward the profound-sounding average, and the build gate will always be
blind to whether a sentence means anything. The two habits counter exactly that:
name the failure modes, and read before you write.
