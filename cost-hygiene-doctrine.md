# Cost hygiene: measure every session's spend, budget the standing context

> **In one line.** Make every session leave a cost trace attributed to its work, and keep
> the standing context every session re-reads on a budget — so spend is explained and the
> context tax can't grow unnoticed.

**TL;DR — the mechanisms**
- One shared capture point logs every session's cost, tagged by ticket and branch.
- A context budget caps the always-loaded surface so it can't creep silently.
- Path-scoped rules auto-load only when a matching file is read — a third mode between resident and read-on-demand.
- Mid-session levers — compact, clear, rewind — bound the context tax as you go.
- Effort tiers match model spend to how hard the task actually is.

**Read this when** spend is unexplained and you want it measured, not guessed.
**Big ideas:** [measured, attributed cost](OVERVIEW.md#6-measured-attributed-cost)

---

How to make a fleet of agent sessions **leave a measured cost trace** and **pay a
bounded, accounted-for context tax** — so that "what did that work cost?" and
"which work is expensive?" are answerable, and the standing context every session
re-reads cannot grow without someone noticing.

Two disciplines, one premise. The premise is that an agent harness spends real
money per session and re-reads its standing context every turn, and that both are
*invisible by default*: a session ends and the spend evaporates from view; a rule
file grows by ten lines and no one feels it until every future session is slower.
The fix for each is the same shape — **instrument the thing, attribute it to the
work that caused it, and shed structurally** rather than relying on good
intentions.

This is project-agnostic doctrine extracted from a working implementation. Wherever
a concrete noun appears (a CLI verb, a cost tool, a hook name), it is called out as
a **`«slot»`** you fill for your own project; the parameterization table near the
end collects every slot in one place.

It is the **context/cost-discipline companion** to the
[session lifecycle doctrine](./session-lifecycle-doctrine.md): that doc owns the
temporal envelope (open ceremony, close ceremony, the gate) and names two
*opt-in modules* it deliberately does not specify — an **instrumentation module**
(cost capture at session exit) and a **context-discipline** concern (budgets, the
compaction checkpoint). This doc is those modules. A project can adopt the
lifecycle without them; this is what it adopts when it wants them.

---

## Why this shape

### Cost: the measurement makes the doctrine falsifiable

A team accumulates workflow doctrine — "use cheaper models for mechanical work,"
"fan out only when the task is parallel," "this kind of ticket is heavy." Every one
of those claims is a *cost hypothesis*, and without per-session measurement none of
them can be tested. You cannot tell whether a doctrine paid off, whether a category
of work is structurally expensive, or whether a model-mix change helped — you can
only assert. **The cost trace is what turns assertion into evidence.** The design
follows from one requirement: *every* session must leave a record, attributed to
*the work it was doing*, captured *at write time* (because the upstream cost data
ages out and a later re-join would lose the attribution).

### Context: the auto-loaded surface is a per-turn tax

An agent harness reads a set of standing surfaces into context at the start of
every session — instructions, rules, a memory index, agent briefs. That set is paid
for on **every turn of every session**, forever. Unbounded growth there is the most
expensive kind of creep because it is silent and compounding: each added line is
cheap once and costly a million times. The discipline is to **budget each
auto-loaded surface, warn when it exceeds budget, and pair every addition with a
shed** so the standing set stays curated rather than accreting.

The two halves share a method: measure spend per session so each cost claim can be
tested, and shed context through a structural gate because nobody sheds it by
intention.

---

## Part 1 — the measured cost trace

### One capture point, many callers, deduped by session

The load-bearing structural choice: cost capture lives in **exactly one place**
(`«capture-script»`), and every session-exit path *routes through it* rather than
re-implementing capture. This is what guarantees a new exit verb cannot silently
skip instrumentation — there is one place to call, and forgetting to call it is
visible in code review of the new verb.

A representative caller set (yours may differ in number, not in kind):

| Caller | When it fires | Reliability |
|---|---|---|
| A **graceful-exit hook** (`«session-end-hook»`) | the harness emits a clean-shutdown event | fires only on *graceful* exit — a hard-killed process never reaches it |
| The **close ceremony** | a deliberate, run-to-completion user action | the **reliable** path — runs before the session can be torn down |
| The **pause verb** | a multi-session sub-session ends without shipping | closes the hole where long-running work captured *zero* cost |

Each caller **names itself** (`«captured-via»` — stamped into the record so you can
see which path wrote it) and **attributes the work** (`«session-ticket»`, below).

**Dedup is by session id.** Before appending, the script checks whether this
session is already logged (a fixed-string match on the log). Whichever caller fires
first wins; any later one — another path, or a re-run — is a silent no-op. This is
what makes it safe to wire *many* callers: they cannot double-count. The
`«captured-via»` value set is therefore **open** — a new exit verb adds its own
caller and its own label without changing the record shape.

> **Why the reliable path is the close ceremony, not the graceful-exit hook.** A
> graceful-exit hook is the obvious capture point and the wrong *primary* one: any
> environment where the session can be killed hard (a remote session archived from
> a separate client, a crashed terminal, an OOM) never emits the clean-shutdown
> event, so the hook never fires. A deliberate close ceremony, by contrast, runs to
> completion *before* the session is torn down. Wire both — the hook catches the
> sessions that close gracefully without a ceremony — but treat the ceremony as the
> path you rely on.
>
> **The pause caveat.** Because pause records cost-so-far and dedup then no-ops the
> later graceful-exit capture, spend *after* a pause in the same session is not
> recaptured. Pause is by convention a session's final act, so this is acceptable;
> the alternative (no record for paused work at all) is strictly worse.

### Attribution is layered, because the obvious source disappears too early

The record must say *which work item* the cost belongs to — that is the whole point
(cost-per-item is the lever). The trap: the obvious source of the item id (an
active claim, a "current ticket" marker) is often **deleted by the ship step before
cost capture runs**, so a single-source lookup writes an empty attribution for
every shipped session — exactly the sessions you most want to measure.

The fix is a **precedence chain**, most-authoritative first:

1. **An explicit hand-off variable** (`«session-ticket»`) the close ceremony sets
   from the live claim *while it still exists*, before ship deletes it. This is the
   deterministic fix for the reliable path — independent of any ref that ship may
   have removed.
2. **The live claim/marker** pointing at the current commit — covers a pre-ship
   (still-live) session, and a graceful-exit hook firing before any ship.
3. **A terminal-record fallback** — match the shipped work-item by branch in the
   terminal store. Backstop for a graceful exit *after* a ship with neither the
   variable nor the claim in scope. (Read the match deterministically — e.g. an
   exact branch-line match, **no modification-time sort**, because a sync step
   touches sibling files and would poison "newest first.")

A session that ships nothing correctly leaves the attribution empty. The principle
generalizes: **capture attribution from the most authoritative source still alive
at capture time, and fall back explicitly** — never assume the obvious source
outlives the work.

### Capture the detail at write time — the upstream data rotates

The cost record carries not just a headline number but the per-model breakdown
(cost and token split per model used). **Capture this at write time**, even though
it could in principle be re-derived later, because the upstream cost tool *rotates
old transcripts* — a later re-join by session id would silently lose the breakdown
for any session whose transcript has aged out. Version the record shape
(`recordSchema`) so the append-only log stays self-describing as fields are added,
and **rename upstream keys into your own schema** so your record format is decoupled
from the cost tool's format. (Note that the breakdown array may contain rows for
*non-native* models — a different CLI run in the same session window — so a consumer
should not assume every row is the same provider.)

### Fail open, always

Cost capture is instrumentation, not a gate. **Any error in the capture path exits
successfully and silently**, so a failed capture never blocks a session from
ending. A missing cost trace is a tolerable loss; a session that cannot exit because
its accountant crashed is not.

### The lifecycle instrumentation matrix

The discipline that keeps capture honest is an **audit table**: enumerate *every*
way a session can start and end, and mark for each whether cost is captured, how it
is attributed, and by what mechanism. Entry paths carry no record (they acquire the
work the session begins under); capture happens only at exit. The matrix is the
canonical surface — **a new lifecycle verb must land a row in it and route through
the shared capture point**, which is how the audit stays complete rather than
drifting.

The audit's value is that it makes the *gaps* explicit and forces each to be
classified as either a bug to fix or sound-by-design:

| Exit kind | Captured? | Why |
|---|---|---|
| Close ceremony | ✅ | reliable path, explicit attribution |
| Graceful-exit hook | ✅ (graceful only) | fallback attribution |
| Pause | ✅ | the fix for "long work captured nothing" |
| Abort/release verb | ❌ by design | aborted work is not attributable to its item |
| Sub-agent / fan-out spend | ✅ in the session total | rolled into the parent session by the cost tool; the eventual exit captures it |
| Hard-kill / crash | ❌ irrecoverable | no exit path fires; documented limitation, not a bug |

Running the audit is what surfaces the *actionable* hole (in the source
implementation, exactly one: the pause path captured nothing until it was wired in).
The remaining ❌ rows are sound-by-design or covered by aggregation. **Any future
hole the matrix exposes is filed as its own work-item rather than absorbed
silently** — the table is a standing invariant, not a one-time pass.

### The value ledger — did the discipline pay off?

Cost records tell you what a session *spent*; they do not tell you what that spend
*produced*. A parallel optional field on each record (`«value-signal»`) captures
whether the session's review/critic/refuter pass caught anything — a simple
categorical: *caught a high-severity issue* / *caught noise or minor findings* /
*nothing*.

Two principles govern the value ledger:

- **Zero is the signal, not a failure to hide.** A session whose critic pass catches
  nothing is correctly recorded as zero-catch. The ledger's purpose is to surface a
  *sustained* zero-catch rate on a specific ceremony type, which is the prune trigger
  — not to justify any individual session's spend after the fact.
- **Aggregate, don't grade.** A single zero-catch does not indict a ceremony. A
  ceremony whose last N sessions all returned zero-catch, across a variety of
  work-item types, is a candidate for removal or scope reduction. That judgment
  belongs to a human reviewing the ledger, not to an automated threshold that fires
  per-session.

The value field is optional; sessions that don't run a critic pass leave it absent
rather than fabricating a value. A consumer querying the ledger can filter to
records that carry the field and group by ceremony type — that is the analysis that
answers "which disciplines pay for themselves?"

### Reading the data

Keep the log as **append-only JSONL at a location that survives ephemeral
workspaces** (e.g. the base repo root, not a per-worktree dir) so it aggregates
across parallel sessions. Because each record is flat and self-describing, the
common questions are one query each: **total per work-item** (group the records by
their work-item field), the **per-item per-model mix** — the model-mix lever — (the
same group-by, but first fan each record out across its nested per-model breakdown
array, since the per-model cost lives *inside* the breakdown, not at the top level),
or a live burn view from the cost tool directly. Keep the headline scalars (total
cost, token split) flat at the top of the record so a human can read a line by eye
or pull one figure without unpacking the nested detail.

### When to escalate beyond a flat log

A flat JSONL log is the right floor. **Escalate to live monitoring / richer
telemetry only on a measured trigger** — e.g. a single session *regularly* exceeds
a cost threshold, or routinely consumes most of a rate-limit window. The discipline
here is the same as everywhere in this doc: let the *data already being captured*
tell you when the cheap instrument is no longer enough, rather than building
dashboards speculatively. If cost is *explained* (a known model floor plus a known
mega-project tail) it is not an anomaly needing more instrumentation — only an
unexplained detachment from a known driver is.

### The merge-timing ledger — watching the merge window grow

Session cost answers "what did that work spend?" A sibling question worth a
separate log is "how long does the merge take?" The merge window is dominated by
the pre-merge end-to-end gate (see the [engineering-discipline companion](./engineering-discipline-doctrine.md)),
and that gate grows with the test suite — so the curve is worth watching as a trend
rather than treating as folklore.

Keep a **second append-only JSONL log** (`«merge-timings-log»`) at the same
location as the cost log (a location that survives worktree cleanup and aggregates
across parallel sessions). One record per *merge event*, not per session — a pause
or graceful-exit path never merges, which is why this is a dedicated ledger rather
than a field on the cost log. Each record carries two distinct metrics:

| Metric | What | Emitted by |
|---|---|---|
| `e2e_gate` | Seconds the pre-merge end-to-end gate ran, plus its outcome (passed/failed) | the gate script |
| `lock_held` | Seconds the merge-lock was held | the lock-release step |

Both are timestamp differences computed at write time — no hot-path overhead — and
both fail open (a timing-write error never blocks the merge). Separating the two
metrics makes the trend readable: `e2e_gate` tracks suite growth over time;
`lock_held` tracks whether the contention window is widening independently.

The ledger's job is to make the trend visible before the merge window becomes a
point of friction. Query it periodically — not after every merge — with a rolling
average. When the gate time starts to grow, the question becomes "what tests were
added?" rather than "why is this taking so long?" A measurement that exists answers
that question; folklore does not.

---

## Part 2 — the context budget

### Budget every auto-loaded surface

Enumerate the surfaces read into context **at session start, every session** — the
standing instructions, each rule file, the memory index, the shared agent brief —
and give each a **budget**, measured in the unit the *writer feels at edit time*
(lines, not bytes; a writer feels line count as they edit). A representative set:

| Surface | Budget unit | Enforcement |
|---|---|---|
| Standing instruction file(s) | lines | session-start warning when exceeded |
| Each rule / policy file | lines | session-start warning when exceeded |
| Memory / knowledge **index** | entries | cap-triggered shed at close |
| Shared agent brief | lines | warning; cost is *multiplied* by team size |

Pick the numbers off a **calibration datum** — the largest file that genuinely
earns every line — and set the budget a little above it. Too tight and the warning
fires on healthy files and trains everyone to ignore it; too loose and it never
fires. The budget's job is to stay *meaningful*: a file over budget has probably
absorbed content that belongs in a load-on-demand location, not in the
always-loaded set.

### Aggregate corpus ceiling — per-file budgets times an unbounded file count is still unbounded

Per-file budgets control each file's size but leave the total auto-loaded surface
uncapped: a team that adds a new rule file every few sessions still grows the
session tax indefinitely, even if every individual file stays in budget. The fix is
an **aggregate ceiling** (`«corpus-ceiling»`) on the total size of the always-loaded
rule corpus — measured in the same unit (lines) for the same reason. Exceeding it is
a **prune trigger**, not an automatic block:

- Rehome verbose design rationale to a load-on-demand location (e.g. a design-doc
  directory that a session reads only when the work needs it).
- Collapse a rule file to a pointer-stub that names the rationale's new home.
- Only then is adding a new file deficit-neutral.

Set the ceiling a little above the measured baseline (the same philosophy as the
per-file budget: enough headroom to absorb a run of new rules before the signal
fires). Adding a new rule file without pruning equivalent volume elsewhere is what
trips the gate. The session-start warning checks the aggregate alongside each
per-file check — one script, one run, two signals.

### Path-scope the situational rules so they cost nothing until they apply

The budget and the ceiling fight the always-loaded surface's *size*; a third lever
attacks its *membership*. Not every rule needs to be resident every session. A rule
that only applies when you touch a particular surface — a deploy rule when editing the
deploy config, a migration rule when editing the persistence layer — can be
**path-scoped** (`«path-scope»`): tagged with the file globs it governs and loaded
*automatically but conditionally*, only when the session (lead or subagent) reads a
matching file, instead of always-on. It is a third mode between the two the rest of
Part 2 assumes:

- **Always-loaded (resident).** In the context every turn; pays the full per-turn tax;
  the surface the budget and the ceiling bound.
- **Load-on-demand (rehomed).** Moved to a location a human reads deliberately; zero
  standing cost, but it only arrives if someone remembers to fetch it.
- **Path-scoped (conditional auto-load).** Keeps the *automatic delivery* of the
  resident mode — it fires without anyone remembering — while shedding the *standing
  cost*, because it isn't in the context when the work doesn't touch its surface.

Path-scoping is what lets the resident set shrink to the genuinely cross-cutting rules
without exiling the situational ones to a directory nobody reads. Two properties to
hold straight: **measure the budget and the ceiling against the resident set only** —
path-scoped rules aren't always present, so counting them would re-inflate the number
you moved them out of; and the resident set is **snapshotted at session start**, so
re-scoping a rule takes effect next session, not mid-one. The discriminator for which
mode a rule belongs in is one question: *must this hold regardless of which file is
open?* If yes, it is resident; if it only governs a surface, path-scope it.

### Warn at the moment of orientation; gate at the moment of writing

Two enforcement points, deliberately different in force:

- **A session-start warning** (`«budget-check-hook»`) measures every surface and
  prints a visible, **non-blocking** notice when any is over budget. Non-blocking is
  the point: a blocking budget check invites workarounds, whereas a visible signal
  *at the moment the operator is orienting* creates the right incentive without
  becoming a gate to route around.
- **A close-ceremony shed** is the structural counterpart — enforcement at
  **write time, paired with the add**. Two rules: (1) before writing a new
  standing-knowledge entry, check whether an existing one is now stale or redundant
  and remove it *in the same change* — the add does not ship without the shed; (2)
  if the index exceeds its entry cap, archive the lowest-value entries (to a dated
  archive file, never a rolling one) before the new entry is admitted.

The warning is the *observable signal*; the paired shed is the *structural gate*.
Neither alone holds the line — the warning is ignorable and the shed only fires when
someone is adding — but together the surface stays bounded.

### Catalog completeness needs its own check

If you keep a load-on-demand **full catalog** alongside the curated index, note that
it has its own silent-drift failure mode: nothing forces it to stay complete, so it
diverges from the actual file set over time. A second non-blocking session-start
check (`«catalog-check-hook»`) that diffs the catalog against the real files —
warning on both *missing* rows (a file with no entry) and *stale* rows (an entry
whose file is gone) — catches this. Same posture as every check here: **warn, never
gate**, and reconcile in the same session the warning fires.

### The write protocol that prevents creep at the source

The cheapest enforcement is at authoring time. Require every new rule/policy file to
open with two short stanzas: a **Purpose** (one sentence: what this governs) and an
**Out of scope** (what looks like it belongs here but does not). The stanzas force a
write-time decision — *is this a rule, or is it rationale that belongs in a design
doc?* — which is the decision whose absence causes creep. Self-policing at the point
of writing beats any after-the-fact sweep.

---

## Part 3 — context preservation and effort, mid-session

Two operational disciplines that sit alongside the budgets — they govern the *live*
context of a running session rather than the standing surfaces.

### Preserve the prompt cache

If the harness caches the conversation prefix with a short time-to-live, certain
mid-session actions **invalidate the cached prefix and force a full re-read** of the
context — paying for the whole conversation again. The high-cost invalidators are
typically: adding or removing tool providers (e.g. MCP servers) mid-session,
switching the model mid-session, and adding/removing plugins. The discipline is to **lock these at
session start** rather than toggling them mid-flight. (Per-agent overrides in
*separate* contexts — e.g. a sub-agent on a cheaper model — do **not** invalidate
the parent's cache, because they run in their own context; the rule is specifically
about mutating the *lead* session's prefix.)

There is also a time dimension worth a single sentence of doctrine: if the cache has
a fixed TTL, a deliberate *wait* that crosses it pays a full re-read on the next
turn — so when a session must idle, idle either comfortably *inside* the window or
commit to a long wait that amortizes the one re-read, never land just past the
boundary repeatedly.

### Compact at named gates, not just on a usage threshold

Run the context-compaction verb (`«compact»`) **proactively at structural
boundaries** — after a design is locked and before implementation, after integration
and before verification, after the tests pass and before the close phase — not only
when usage crosses a high-water mark. Auto-compaction is a safety net, not a
substitute: compacting at a *named gate* discards the now-irrelevant exploration
context exactly when it has gone cold, which is cheaper and cleaner than letting it
ride until a threshold forces a worse-timed compaction. Pair it with the cheaper
context resets where they fit: a *clear* when switching to unrelated work, a *rewind*
to drop a bad turn rather than correcting on top of it.

### Match effort to the work

If the harness exposes a tunable effort/reasoning level, treat it as a **cost lever
with a floor, not a free dial**. The doctrine has two parts:

- **There is a floor for intelligence-sensitive work.** Multi-file, cross-layer,
  design, and planning work has a minimum effort rung below which the model
  *under-thinks* — and the symptom (shallow reasoning) is fixed by *raising effort*,
  not by prompting around it. Reserve the low rungs for genuinely single-concern,
  well-specified, latency-sensitive tasks.
- **Effort is strict at the low end on some models.** A model may honor a low effort
  setting *literally* — scoping work to exactly what was asked and no further — which
  makes a low/medium default a **hard scope-limiter**, not a neutral middle. Know
  your model's calibration: if low effort means "do the minimum," then a moderately
  complex task at that setting is mis-rung, and the fix is to match the rung to the
  work rather than to write more prompt.

### Push bulk and research spend out of the lead context

The lead session's context is the most expensive place to do work, because it is the
longest-lived and most-cached. Two standing moves:

- **Delegate by cost tier.** Mechanical bulk work (search, transforms, data
  generation) goes to the cheapest model; scoped research and implementation to a
  mid tier; reserve the expensive tier for planning, design, and cross-layer
  integration. Each delegate runs its **own** context, so a large research result
  never lands in the lead window.
- **Keep bulk output out of context.** Prefer a single bulk command over a long
  chain of individual tool calls when the operation is mechanical and verifiable;
  tee verbose tool output to a scratch file and grep it for the part that matters
  rather than reading the whole thing into context.

---

## Parameterization seams

| Slot | Source binding | Agnostic form |
|---|---|---|
| `«capture-script»` | the single ccusage-backed capture script | your one shared cost-capture entry point that every exit verb routes through |
| `«session-end-hook»` | the graceful `SessionEnd` hook | your harness's clean-shutdown event, if it has one (fallback caller, not primary) |
| `«captured-via»` | the `capturedVia` record field | a self-naming label each caller stamps so a record names the path that wrote it |
| `«session-ticket»` | the `IF_SESSION_TICKET` hand-off var | the explicit work-item id the close ceremony sets from the live claim before ship deletes it |
| `«cost-tool»` | a CLI that aggregates per-session cost from the local session transcripts | whatever reports per-session cost offline; match sessions by a stable id |
| `«budget-check-hook»` | the session-start doc-budget check | your non-blocking start-of-session budget warning over auto-loaded surfaces |
| `«catalog-check-hook»` | the memory-catalog completeness check | your non-blocking start-of-session check that a load-on-demand catalog matches the real file set |
| `«value-signal»` | the optional per-session critic-catch field | the categorical (caught/noise/nothing) that powers the ceremony-value ledger |
| `«merge-timings-log»` | the append-only JSONL at the base repo root | your per-merge-event timing log (e2e gate seconds + lock-held seconds), separate from the session cost log |
| `«corpus-ceiling»` | the total-lines cap on the always-loaded rule corpus | your aggregate ceiling over all auto-loaded rule files; prune trigger when exceeded |
| `«path-scope»` | the `paths:` rule-file frontmatter that conditionally auto-loads | your mechanism for loading a situational rule only when a matching file is read (lead + subagent), measured against the resident set, not always-on |
| `«compact»` / `«clear»` / `«rewind»` | the context-management verbs | your harness's compact / reset / drop-last-turn operations |
| effort lever | the `/effort` rungs | your harness's reasoning/effort dial, if it has one — a cost lever with an intelligence floor |
| escalation trigger | the ~$N/session live-monitor threshold | your measured trigger for graduating from a flat log to live monitoring |

---

## When to adopt this

The cost half pays off the moment **you want to test a workflow-cost claim with
evidence instead of asserting it** — which is as soon as you have more than a couple
of sessions and any doctrine about how to run them. The context half pays off the
moment **a standing surface is read into every session** and more than one person
(or one model, over time) edits it.

The non-negotiable core, if you adopt nothing else: **one shared capture point that
every exit routes through**, **attribution captured from the most authoritative
source still alive at capture time**, **a lifecycle matrix that makes every
capture gap explicit**, and on the context side **a budget per auto-loaded surface
with a shed paired to every add**. Those four are what make cost *falsifiable* and
context *bounded*. The instrumentation matrix audit, the prompt-cache rules, the
effort calibration, and the delegation tiers are leverage on top — adopt them as the
measured data shows you need them, which is the whole point of measuring.
