# The session lifecycle: one front door in, one gate out

> **In one line.** Bound each unit of agent work so it's classified at entry and
> validated at exit — heavy ceremony attaches only to the work that needs it.

**TL;DR — the mechanisms**
- A multi-flow entry router classifies intent before any ceremony runs.
- An ordered ready-check must pass before any code is touched.
- A single close gate runs validation, then merges as an explicit commitment.
- Pause/resume carries multi-session work across a dead window without losing the thread.

**Read this when** you're designing how a session starts, hands off, and ends.
**Big ideas:** [front door in / gate out](OVERVIEW.md#3-one-front-door-in-one-gate-out) · [durable handoff](OVERVIEW.md#4-durable-re-derivable-handoff)
**Depends on:** [backlog system](backlog-system-doctrine.md) · [worktree workflow](worktree-git-workflow-doctrine.md)

---

How to bound a unit of agent work — a **session** — so that it is **classified at
entry** through a single front door and **finalized at exit** through a single
close ceremony, with the validation gate run at the close. The result is that the
heavy open/close ceremony attaches to exactly the work that needs it, every
session leaves the shared trunk in a known-good state, and nothing a session
touched merges without passing the same gate.

This is project-agnostic doctrine extracted from a working implementation. Wherever
a concrete noun appears (a CLI verb, a branch-naming rule, a gate command), it is
called out as a **`«slot»`** you fill for your own project. The parameterization
table near the end collects every slot in one place.

It assumes a host where many sessions run concurrently against one repo (each in
its own worktree), and it leans on three companions for the machinery it does not
re-derive: the [file-per-work-item backlog system](./backlog-system-doctrine.md)
owns **who is working on what** (the claim model, the eligibility picker); the
[parallel-worktree git workflow](./worktree-git-workflow-doctrine.md) owns **how
branches converge** (sync, the merge-lock, the reaper); and
[coordinating a team of subagents](./multi-agent-coordination-doctrine.md) owns
**how a session that fans out delegates work**. This doctrine owns only the
*temporal envelope*: how a session opens, what it must establish before touching
code, and how it closes.

---

## Why this shape

A session is one person or agent sitting down to do one piece of work. Left
unstructured it has two failure modes that compound across a team:

1. **Ceremony taxes the wrong work.** The open ceremony that a *new feature* needs
   — pick an item, claim it, branch, sync, baseline-check — is pure overhead for a
   *one-off question* or a *backlog grooming* pass. If every session pays it, the
   cheap intents get taxed and people route around the ceremony entirely, which is
   worse. *Fix: a front-door router that classifies intent first and routes the
   heavy ceremony to the one intent that needs it.*
2. **Sessions leave the trunk in an unknown state.** Without a close gate, a
   session merges whatever it happened to produce — half-validated, with a failing
   test it "didn't cause," with no record of what shipped. The next session
   inherits the mess and can't tell good state from bad. *Fix: a single close
   ceremony whose first act is the full validation gate, and whose merge is an
   explicit commitment, not a side effect.*

The two halves share one invariant: **the gate is the same at both ends.** At open
it runs as a *baseline check* — proving you inherited a clean tree, so any failure
you see later is yours. At close it runs as the *merge gate* — proving you are
handing back a clean tree. Same commands, two roles. Everything else in this doc is
the scaffolding that makes those two gate-runs trustworthy.

---

## Part 1 — the front door: classify, then route

The entry point is **not** the open ceremony. It is a *router* that reads the
session's first intent and dispatches to one of a few flows. Routing happens
**before** any ceremony, so a question is never blocked behind claim-and-branch.

A representative flow set (yours may differ in number, not in kind):

| First intent looks like… | Flow | Route to |
|---|---|---|
| silent / "next" / "continue", **and** held work exists | **resume** | re-attach to the in-flight item (below) |
| silent / "pick something up", no held work | **start** | the open ceremony (Part 2) |
| "groom / triage / research the backlog" | **meta-work** | the backlog's own ceremony — *not* the open ceremony |
| a question with no work intent | **one-off** | answer directly — no ceremony at all |
| names a concrete change ("fix Z", "add W") | **ad-hoc** | branch now, file the work-item once it proves real |
| spans flows or is vague | — | **ask** the operator which, then route |

Three rules make the router pay off:

- **Infer when clear; ask only when genuinely ambiguous.** The router is a
  classifier, not an interrogation. A vague-or-spanning first message is the only
  case that earns a clarifying question.
- **The open ceremony is one leaf, not the entrypoint.** This is the load-bearing
  inversion: the full claim → branch → baseline ceremony serves exactly the
  *start-new-work* intent. Resume, meta-work, one-off, and ad-hoc each have their
  own (lighter or different) entry. Making the heavy ceremony universal is the
  anti-pattern this part exists to retire.
- **Misroute escalates in place; it does not restart.** If you routed a message to
  *one-off* (answered it) and it turns out to be the front of real work, escalate
  to the ad-hoc or start ceremony *at that moment*. The open ceremony's
  no-code-before-ready law (Part 2) is the backstop: a late escalation cannot have
  leaked unbranched code, because no code was written.

### Resume vs. ad-hoc — the two non-start entries that still touch code

- **Resume** re-attaches to multi-session work that was *paused*, not finished (see
  *Ship or pause* in Part 3). The resume read is **read-only**: it reports the
  item, its branch, the branch's worktree, commits-ahead-of-base, and the durable
  handoff note — read **from the branch**, which is authoritative; the trunk's copy
  is stale-by-design. From a fresh context you **adopt** the branch (free it from
  its old worktree, check it out here, re-acquire the claim) rather than returning
  to the original local state. The claim model and adopt mechanics belong to the
  [backlog](./backlog-system-doctrine.md) and
  [worktree](./worktree-git-workflow-doctrine.md) companions.
- **Ad-hoc** is unplanned work named on the spot. It does **not** pull from the
  groomed queue, but it is **not** an excuse to skip the work-item: branch first,
  and **before the first commit**, once the change clearly warrants shipping, file
  its work-item and run the open ceremony's tail (claim → rename branch → sync from
  base → baseline). This keeps every shipped change traceable to an item without forcing
  premature filing of work that might evaporate.

---

## Part 2 — the open ceremony (the *start* leaf)

When the router lands on *start new work*, run an **ordered, gated** ceremony. Its
defining property is a law:

> **No code before the ready-check is stated.** The steps run in order; each
> completes before the next begins. If you are writing implementation code and the
> ready-check has not been printed, stop.

The law matters because the ready-check is where the session proves it is working
on the *right* item, on a *clean* baseline, without *colliding* with a parallel
session. Code written before that proof is code written blind.

The steps, abstracted:

1. **Get into an isolated workspace.** If the session is in the shared base repo,
   mint a worktree and continue there. (Worktree mechanics:
   [companion](./worktree-git-workflow-doctrine.md).)
2. **Sync from the base branch** (`«sync»`) *before* picking work — so the picker
   reads current state and does not hand you an already-claimed item.
3. **Orient, then pick-and-claim atomically.** Read what parallel sessions are
   doing (context for the conflict-risk assessment in step 7), then **atomically**
   pick the highest-priority eligible item and claim it in one step (`«claim-next»`).
   A two-step "pick, then claim" leaves a race window; fuse them. State the choice;
   do not ask for confirmation — priority order is almost never overridden.
4. **Name the branch deterministically** from the claimed item (`«branch-name»`) —
   never hand-spelled, never left as the base-branch name. A deterministic rule
   means a branch's item is always recoverable from its name.
5. **Baseline-check** — run the gate (`«gate»`) plus any cheap static checks.
   Whatever fails here is **pre-existing**, and *seeing it transfers ownership*
   (see *Failure ownership* below): for each failure pick fix-inline / capture as a
   tracked item / escalate. "Carry forward unaddressed" is not a legitimate
   response.
6. **Classify and declare the team.** Identify the layers the work touches. If it
   clears the fan-out bar, name the subagents and their write scopes
   ([coordination companion](./multi-agent-coordination-doctrine.md)); otherwise
   declare "inline, single scope." The declaration *is* part of the ready-check.
7. **Confirm ready.** State, as a single block: branch + item title; the primary
   deliverable; baseline status (clean, or each pre-existing failure with its
   response); active parallel sessions; **conflict risk** (files this session will
   touch, flagged against parallel sessions); the team declaration; and any
   unresolved design decision. If a decision is unresolved, stop and ask. If all
   are resolved, announce and proceed — do not frame readiness as a question.

The ceremony is deliberately front-loaded: every expensive mistake (wrong item,
dirty baseline, colliding scope, unresolved design) is cheaper to catch here than
after code exists.

---

## Part 3 — the close ceremony (the *exit* gate)

A session closes through one ceremony, and its first act is the gate. The ordering
is doctrine: **validate before you do anything else**, because every later step
(patch notes, merge, cleanup) assumes a green tree.

### The gate runs first

Run `«gate»` (e.g. lint → typecheck → full test suite → backlog-integrity check),
fastest-feedback-first. All must pass. The
[engineering-discipline companion](./engineering-discipline-doctrine.md) fills this
slot — what the gate checks, the standing invariants it defends, and the transitive
ownership posture below. **Failure ownership is transitive** (see below): any
failure here — even one the session did not cause — gets fix / capture / escalate
before the merge, never "ships with a known red."

### Ship or pause — two terminal shapes

A close is one of two things; decide before doing the rest:

- **Ship** — the item's work is done. Run the full tail (record what shipped →
  serialize → merge to base → publish). This is the default.
- **Pause** — multi-session work that ended a *sub-session* but is not done (more
  phases coming, or commits that must not hit the base branch yet). The pause path
  is **release-on-pause**: refresh the durable handoff note, then **drop the claim
  but keep the branch at its in-progress stage**. The branch carries the work
  across sessions; the base branch is never touched by a pause. *Stop there* — a
  pause runs none of the ship tail.

The pause shape is what makes long work safe to interrupt: claim-absence (not a
lock's liveness) is the signal that "nobody is holding this," so a fresh session
can adopt it cleanly (Part 1, resume). The durable note is the contract — anything
uncommitted is lost on adopt, so commit before pausing.

### Auto-chaining — pause, then fire the successor automatically

For a review-attested multi-phase migration — a committed phase plan whose
adversarial review actually happened — you may want the next phase to start in a
**fresh cold-started session** rather than requiring a human to resume it by hand.
The pattern is the automation of the pause → adopt seam: the session at a phase
boundary pauses itself, then fires a trigger that spawns the successor. The
successor cold-starts against durable repo state (the committed work and the
handoff note) and never sees the predecessor's context window.

Four guardrails make the chain trustworthy:

1. **Gate before fire.** The validation suite (`«gate»`) must pass before the
   trigger fires. This is a cold-start-safety requirement, not a deploy concern:
   the successor builds on a baseline it cannot inspect, so a red baseline propagates
   silently across phases. A failed gate stops the chain and notifies a human.
2. **Depth counter.** The handoff note carries a `depth-remaining` field,
   decremented by each successor on adopt. When it reaches zero the chain stops
   rather than running forever. This is the in-band runaway guard — the only one
   that works without external state.
3. **Envelope stop.** If the phase-boundary diff touches the reviewed envelope
   (the shared type contract, the top-level wiring file — `«protected-files»`), the
   chain stops for human re-review before firing. The plan earned its "trust the
   plan" status from its reviewed scope; a change that crosses the envelope exits
   that scope.
4. **Adopt-on-claim-absence.** The pause drops the claim; the successor adopts on
   claim-absence, not on a lock-liveness probe. This is the same signal the manual
   resume flow uses — the chain fires the same adopt path, making it the automatic
   version of what a human operator would do.

The phase identity (which phase, how many remain, the plan document to read) lives
in the handoff note — read by the successor on adopt, not embedded in the trigger
prompt. This keeps the trigger lean and makes the handoff auditable in the repo
rather than embedded in a transient dispatch payload.

**What qualifies.** This pattern applies only to work with a committed phase plan
that was adversarially reviewed before execution began — not to ordinary multi-commit
work filing itself into the auto-chain path. When in doubt whether work qualifies,
it does not. The gate and the depth counter exist precisely because a mis-qualified
chain runs unsupervised.

### The ship tail (abstracted)

Run only on the *ship* branch:

1. **Optional quality-review pass**, risk-tiered by the diff's blast radius
   (skip on docs/data-only; a cheap inline pass on a small diff; a delegated
   factoring review on a multi-layer or shared-primitive diff). It is the
   *factoring* backstop the behavioral gate cannot see — **advisory, never
   blocking**; if it changes the tree, **re-run the gate**.
2. **Record what shipped** — player-facing notes if anything is user-visible
   (let the rubric decide, don't pre-judge); a one-line changelog entry regardless.
3. **Shed-paired memory / doctrine capture** — if the session produced a durable
   fact, a reusable procedure, or a "do it this way" rule, write it now; pair any
   memory *add* with a *shed* of a now-stale one so the curated set does not grow
   unbounded. (The instrumentation and context-discipline side of close — cost
   capture, compaction checkpoints — is a separable concern owned by the
   [cost-hygiene companion](./cost-hygiene-doctrine.md); this doctrine only notes
   that those steps live in the tail.)
4. **Serialize the merge under a lock**, then ship-the-item → sync base → merge →
   push → release the lock. The lock and merge mechanics are the
   [worktree companion](./worktree-git-workflow-doctrine.md); the item-transition
   (move to terminal, collapse the body, delete the claim) is the
   [backlog companion](./backlog-system-doctrine.md). **Acquire the lock before the
   item-transition**, and hold it across the whole window — a lock dropped mid-merge
   reintroduces the race it exists to prevent.

The merge to the base branch is **the commitment**. Before it, a failure is "the
previous session's problem"; after it, it is the trunk's problem and every future
session inherits it. That asymmetry is why the gate guards the merge specifically.

---

## The gate, and failure ownership

Two principles bind the gate-runs at both ends together:

- **One gate definition, two roles.** Define the validation suite once
  (`«gate»` — fastest-feedback-first so a cheap failure fails the run before an
  expensive one runs). It is the *baseline check* at open and the *merge gate* at
  close. A standalone "just run the gate" verb is useful mid-session, but the
  authority is the same suite each time.
- **Ownership is transitive.** A failing check encountered *anywhere* — baseline,
  mid-session, or close — is **this session's** to resolve, regardless of who
  caused it. Three legitimate responses, pick one per failure: **fix** it,
  **capture** it as a tracked work-item (with the failing assertion, actual-vs-
  expected, and the commit where first observed), or **escalate** it to a human.
  The forbidden fourth option is noting it in passing and moving on — that is how a
  red baseline silently propagates into the trunk. (The full discipline — what
  belongs in a unit test vs. an end-to-end check, regression-test obligations — is a
  companion engineering-discipline concern; this doctrine only fixes *that a seen
  failure is owned*.)

---

## Parameterization seams

| Slot | Source binding | Agnostic form |
|---|---|---|
| `«router flows»` | the four-flow session front door | your set of entry intents + their routes (start / resume / meta-work / one-off / ad-hoc, ≥ the start+close pair) |
| `«gate»` | lint → typecheck → test → backlog-check | your validation suite, fastest-feedback-first; same definition at open and close |
| `«sync»` | the merge-from-base script | how a worktree pulls the base branch before picking work and before the final merge |
| `«claim-next»` | the atomic next-and-claim verb | your pick-highest-eligible-and-claim-in-one-step operation (companion backlog) |
| `«branch-name»` | the item-derived branch rename | your deterministic item → branch-name rule |
| `«ready-check»` | the step-7 readiness block | the printed proof (item / deliverable / baseline / conflict-risk / team / open-decisions) that gates first code |
| `«pause»` | release-on-pause (drop claim, keep branch) | your hold-unfinished-multi-session-work shape |
| `«handoff note»` | the in-branch Resume block | your durable, in-repo done/next/in-flight note for a paused item |
| `«merge-lock»` | the serialized main-merge window | your mechanism for one-merge-at-a-time into the base (companion worktree) |
| `«chain-depth»` | `depth-remaining` in the handoff note | the runaway-loop guard for auto-chained phases; decremented by each successor on adopt |
| `«chain-trigger»` | a harness trigger that spawns a fresh session | your mechanism for firing the successor session from inside the predecessor (auto-chain) |
| quality-review module | the risk-tiered factoring pass | **opt-in**: a post-gate, advisory, blast-radius-tiered factoring review |
| instrumentation module | cost capture + compaction checkpoints | **opt-in**: the close-tail steps owned by the [cost-hygiene companion](./cost-hygiene-doctrine.md) |

The two `opt-in` modules are separable: the lifecycle works without them, and a
project with no cost instrumentation or no factoring pass drops them cleanly.

---

## When to adopt this

This pays off the moment **more than one session runs against one repo** — solo or
team — and you need every session to start on a known baseline and end on one. For
a single throwaway script in a scratch repo it is overhead; the value is entirely
in the *repeatability* across many sessions and many worktrees.

The non-negotiable core, if you adopt nothing else: **a front door that classifies
intent before charging ceremony**, **a no-code-before-ready law on the open
ceremony**, **one gate definition run at both baseline and close**, and
**transitive failure ownership** so no red check is ever "not my problem." Those
four are what keep a fleet of concurrent sessions converging on a clean trunk
instead of slowly degrading it. The ship/pause split, the quality-review pass, and
the instrumentation tail are leverage on top.
