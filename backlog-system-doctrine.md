# A file-per-work-item backlog system

> **In one line.** One markdown file per work item, lifecycle state in its frontmatter
> and ownership in a git ref — diffable, server-free, and safe for many concurrent
> worktrees.

**TL;DR — the mechanisms**
- One file per work item; lifecycle stage lives in frontmatter, not in folders.
- "Who owns it" is an exclusive-create git ref — a claim exactly one session can win.
- A stage machine with a decomposition gate stops an over-large item from starting.
- Typed parent items (tracker / umbrella / record) model the work-about-work.
- Tooling parses committed frontmatter across branches; a raw grep is not the read path.

**Read this when** you need a concurrent-safe backlog without a server or a database.
**Big ideas:** [concurrency without collision](OVERVIEW.md#2-concurrency-without-collision) · [durable handoff](OVERVIEW.md#4-durable-re-derivable-handoff)

---

How to run a backlog where each work item is **one markdown file**, its lifecycle
state lives in that file's **frontmatter**, and "who is working on it" lives in a
**git ref** rather than the file. The result is a backlog that is diffable,
grep-resistant-but-tool-readable, and **safe for many concurrent worktrees on one
repo** without a server, a database, or a lock daemon.

This is project-agnostic doctrine extracted from a working implementation. Wherever
a concrete noun appears (an ID prefix, a directory name, a section heading), it is
called out as a **`«slot»`** you fill for your own project. The parameterization
table near the end collects every slot in one place.

A companion runnable tool ships the verbs described here as a single CLI (referred
to throughout as **`bl`**, the "backlog" tool — rename to taste). See *Packaging
the tool* at the end for how the tool is structured and split.

---

## Why this shape

Three problems sink most lightweight backlogs as soon as more than one person — or
more than one agent in more than one worktree — touches them at once:

1. **Merge conflicts on a shared index.** A single `BACKLOG.md` or a tickets table
   in one file conflicts on every concurrent edit. *Fix: one file per item.* Two
   sessions creating two items touch two different files; nothing conflicts.
2. **Lost claims / double-claims.** "Who's working on this?" stored *in* the item
   file races: two sessions both read "unclaimed," both write "claimed," both
   commit on their own branches, and the conflict only surfaces at merge — after
   both did the work. *Fix: claims are git refs created with exclusive semantics*,
   atomic across worktrees that share one object store.
3. **State drift between branches.** A long-lived branch flips an item to
   in-progress; a sibling worktree querying "what's next?" from its own checkout
   never sees that flip and hands the same item out again. *Fix: cross-branch reads
   straight from the git object store*, not the current working tree.

The system below is the smallest set of mechanisms that solves all three while
keeping the backlog as plain files a human can read, edit, and diff.

---

## The four moving parts

| Part | Where it lives | What it answers |
|---|---|---|
| **The item file** | `«BACKLOG_DIR»/«ID».md` | What is the work? (prose body) |
| **Frontmatter** | YAML block at the file top | What stage / priority / dependencies? |
| **The claim** | `refs/claims/«ID»` git ref | Who is working on it right now? |
| **The terminal archive** | `«SHIPPED_DIR»/«ID».md` | What shipped, and on which branch? |

Stage, priority, and dependencies move by **editing frontmatter in place** — never
by moving the file between folders. The file moves exactly once, at the end of its
life, from `«BACKLOG_DIR»` to `«SHIPPED_DIR»`.

---

## The stage machine

Every item has a `stage`, advancing in one direction through a fixed ladder:

```
triage → designing → ready → in-progress → (terminal)
```

- **triage** — captured, not yet specified. The inbox.
- **designing** — being specified; the spec is being written and decomposed.
- **ready** — specified, decomposed, and claimable. This is the *only* stage the
  "what's next?" picker draws from.
- **in-progress** — claimed and being worked. Set by the claim verb.
- **terminal** — shipped, deferred, or superseded. The file has moved to
  `«SHIPPED_DIR»` and dropped its `stage` field (see *Terminal items*).

The ladder is deliberately short. The load-bearing transition is
**designing → ready**, because that is where the *decomposition gate* fires (below).
Everything upstream of `ready` is invisible to the work picker, so an
under-specified item cannot be handed out as work.

Stage transitions are made by the tool's `promote` verb, which runs the gate for
the *current* stage before advancing. A human can also hand-edit the field; the
tool's validation pass will still hold the invariants at check time.

---

## Frontmatter: the item's state

The frontmatter is the entire machine-readable surface of an item. Keep it small
and typed. A representative schema:

```yaml
---
id: «PREFIX»-0            # canonical id; also the filename stem
title: <one line>
schemaVersion: 1          # frontmatter schema version (see Migrations)
stage: <triage|designing|ready|in-progress>
priority: <0–«CAP»>       # higher = sooner; sparse integers
requires: [«PREFIX»-0]    # ids that must be terminal before this is eligible
conflict-with: []         # ids that must not be *claimed* concurrently
supersedes: []            # ids this replaces
informs: []               # arc/relationship links (and parent↔child links)
single-session-justified: # optional: bypass the decomposition gate, with a reason
parentKind: arc-tracker   # optional: typed parent-item kind (see Parent items)
---
```

Rules that keep the schema honest:

- **The full filename stem is the canonical id, verbatim.** `«PREFIX»-42.md` *is*
  item `«PREFIX»-42`. A suffixed archive filename keeps its **whole** stem as the id
  — `«PREFIX»-42-some-slug.md` is the id `«PREFIX»-42-some-slug`, and `requires:`
  references it by that full stem, *not* a truncated `«PREFIX»-42`. Treat the stem
  as opaque; an anchored `«PREFIX»-<digits>` pattern is a *sanity filter* for
  skipping non-item files, never a truncation extractor. Above all, never derive an
  id with `split('-')[0]` (that yields the prefix alone and silently breaks every
  dependency check).
- **`id` carries a typed prefix.** The `«PREFIX»` registry (your domains: features,
  infra, content, …) is a configurable set. New ids are minted by scanning all
  known ids — across the backlog dir, the shipped archive, **and** live claim refs
  — and taking the next free number, never by counting files.
- **Arrays are arrays.** `requires`, `conflict-with`, `supersedes`, `informs` are
  lists of ids. Empty is `[]`, not absent.

### Terminal items

On ship, an item file moves to `«SHIPPED_DIR»` and its frontmatter is trimmed to a
fixed **whitelist** — the fields that still mean something after shipping
(`id`, `title`, `schemaVersion`, the dependency arrays, plus `shipped-on: <branch>`
and an optional `terminal: deferred|superseded`; absence of `terminal` means
shipped). Working-state fields (`stage`, `priority`, the gate-override flag) are
dropped — they have no post-ship meaning. The body is collapsed too: a **designated
set of ephemeral section headings** is stripped so the archive records *what
shipped*, not the scaffolding around it. The reference set drops an open-questions
section always, drops the spec section for a shipped/superseded item (and for a
deferred item only when it's a thin checklist), and rewrites a history section to a
one-line "shipped on `<branch>`" stamp. Any working section you want gone post-ship
must be an explicit member of that strip set.

Terminal files are **never deleted.** The archive is the project's durable memory.

---

## The git-ref claim model (the crux)

A claim is **not** a frontmatter field. It is a git ref at `refs/claims/«ID»`,
created atomically:

```
git update-ref --create-reflog -m "session:«SESSION_UUID»" \
    refs/claims/«ID» <HEAD-sha> 0000…0000
```

The trailing all-zeros old-value is the load-bearing trick: `update-ref` with an
old-value of "0000…" means **"create only if it does not already exist"** — the
kernel-level compare-and-swap fails if the ref is already present. So two sessions
racing to claim the same id: exactly one `update-ref` succeeds, the other gets
`reference already exists` and is told the item is taken. No lockfile, no daemon,
atomic across every worktree sharing the object store.

Why a custom ref namespace (`refs/claims/…`, not `refs/heads/…`):
- It stays out of `git branch` output and ordinary tooling.
- It is not pushed by a normal `git push`, so claims never leak to a remote.
- `for-each-ref refs/claims/` enumerates exactly the active claims.

Claims are created by `claim`, deleted by `ship` and by `release`. The reflog
message stamps the **session identity** (`session:<uuid>`), which is what makes
liveness detection possible.

### Claim liveness — surviving a dead session

A claim that outlives its session (hard crash, killed terminal, remote archive)
would otherwise block the item forever. Detecting "is the claimant still alive?"
cannot always rely on a process PID — in some deployments (e.g. a supervised
remote-fleet harness) worktree locks share an immortal supervisor PID, so
`kill -0` always says "alive" and cannot tell a dead session from a live one. The
reference implementation instead uses two per-session signals, both reliable on a
single local (non-network) filesystem:

1. **A session id** stamped into the claim's reflog at claim time (the value of an
   env var the harness exports per session; absent → the stamp is the bare word
   `claim` and liveness reads `unknown`).
2. **A per-session heartbeat artifact's mtime.** *This signal is harness-specific.*
   The reference implementation reads the AI coding harness's per-session
   transcript file (`<config>/projects/<slug>/<session-id>.jsonl`), which the
   harness rewrites on every message/tool result — so its mtime is a free passive
   heartbeat. **A consumer on a different harness substitutes whatever per-session
   liveness artifact its environment maintains** (a heartbeat file, a touched
   lockfile, a process registry), or — if it has none — drops to *claim-absence
   only* plus a manual stale-clear.

Given those two signals, liveness is a three-way verdict (`live` / `dead` /
`unknown`):
- `unknown` — the claim has no session stamp (an old-format or env-less claim).
  Refuse to guess; a human resolves it.
- `dead` — session id present but the heartbeat artifact is gone, or its mtime is
  older than a generous threshold (90 minutes is a sound default — long enough to
  tolerate a multi-agent dispatch or a wait on human input).
- `live` — session id present and heartbeat mtime within the threshold.

The *adopt* handoff (below) uses this verdict to auto-clear a dead claim while
still refusing a live one. **Claim-absence — not PID-liveness — is the primary
abandonment signal;** the liveness probe is only the recovery path for a claim
that was never released because the session died before it could.

---

## Forward-only frontmatter migrations

The frontmatter schema *will* change (a new field, a renamed key). Old item files
on disk and on long-lived branches must keep parsing. The pattern mirrors
save-game migration:

- An ordered **migration array**: index `N` is a pure function `fm → fm` that
  upgrades a schema-`N` frontmatter object to schema-`N+1`.
- `schemaVersion` on each file records the last migration applied. On load, run
  migrations from the file's version up to `array.length`; a fresh file starts at
  `array.length`. An **absent** `schemaVersion` defaults to `0` (a pre-schema file),
  not an error.
- **Append-only.** Never edit a shipped migration — old files migrate forward
  through the whole chain on every load. A version *above* the current max is a hard
  error (a file from the future). A configurable **minimum-version floor** (default
  0, i.e. no floor) lets you hard-fail or wipe files too old to migrate, mirroring
  save-game migration — raise it only as a deliberate break-glass.

This runs at *parse* time, in memory, every load — so every consumer (the picker,
the validator, the listing) sees a uniform current-schema object regardless of what
shape is on disk. The files themselves are rewritten to the new shape only when a
verb next edits them.

---

## The dependency graph

Three relationship fields, three distinct meanings:

- **`requires: [ids]`** — a hard gate. An item is not *eligible* (not handed out by
  the picker) until every id in `requires` is **terminal** (shipped). This is the
  ordering primitive.
- **`conflict-with: [ids]`** — a mutual-exclusion lock. An item is blocked while any
  id in `conflict-with` is **claimed** (not shipped — *claimed*). Use it for two
  items that touch the same volatile surface and shouldn't be in flight together.
- **`informs: [ids]`** — a soft arc link, no gating effect on its own — *except*
  when the item is a typed **parent** (below), where the `informs:` list *is* the
  parent↔child wiring the parent's gate keys on.

The validator enforces the graph's integrity:
- **No dangling requires** — every required id must exist (in backlog or archive).
- **No cycles** — `requires` edges are checked for cycles via a depth-first
  three-color walk; a cycle is a hard error.
- **No duplicate ids** — two files claiming the same id, or a backlog id colliding
  with a shipped id, is a hard error (this is the failure a parallel-id-minting race
  produces, so guard it explicitly).

### Eligibility, in one place

The picker ("what should I work on?") draws from exactly the items that are:
`stage === ready` **and** not claimed **and** all `requires` terminal **and** no
`conflict-with` peer claimed **and** (if an *arc-tracker* parent) all children
shipped **and** not already in-progress on another branch — sorted by `priority`
descending. Keep this predicate as **one pure function** the picker and the listing
both call, so "eligible" means exactly one thing project-wide.

### Parent items: one typed `parentKind`, three shapes

An item that exists to *organize other items* rather than carry its own unit of work
is a **parent**. There are three genuinely distinct parent shapes, and the
load-bearing lesson is that **they differ in which stage they live at, which gate
fires, and how they terminate** — so the kind must be a *typed field*
(`parentKind`), not a memorized convention. Encoding it as one boolean
(`coordinator: true`, the shape this doctrine originally carried) collapses three
behaviors into one and silently mis-stages two of them; a typed enum lets the
validator enforce the right stage per kind instead of leaving it to prose.

| Kind | `informs:` points | Lives at | Gate / terminal |
|---|---|---|---|
| **arc-tracker** | *down* at its children | `ready` (or `in-progress` once its gate opens) | Hidden from the picker and refused by `claim`/`ship` until **every child is terminal**; the validator then emits "children all shipped — close it." Its work *is* the arc; it ships once the arc completes, so the stage validator accepts `ready` *or* `in-progress` (the latter once the gate opens at end-of-life). |
| **umbrella** | *up* at its own parent | `designing` | Its work is writing the rollout playbook and spawning children. **Must not sit at `ready`** — the picker draws only from `ready`, so an umbrella parked there is handed out as top-priority claimable work with nothing to ship (the failure this kind exists to prevent). `designing` honestly means "not yet decomposed into a claimable unit." |
| **record** | *down* at its children | terminal (archived) | An incident/record parent — a *record*, not work; never claimed or worked. It ships into the terminal archive **on creation** while its children stay live in the backlog. A record parent sitting in the active backlog is a validation error. |

Why the stage matters and is enforced, not just documented: the picker is the one
consumer that must never be surprised, and two of the three kinds are *defined* by
"don't hand me out the way you'd hand out the third." So the validator checks
**stage-per-kind** (arc-tracker → `ready`/`in-progress`, umbrella → `designing`,
record → terminal) as a hard invariant. A boolean cannot express that; the typed kind is what
makes "wrong stage for this parent" a caught error rather than a convention someone
forgets.

> **Migrating off a boolean.** If you started with a `coordinator: true` boolean (the
> arc-tracker case), introduce `parentKind` and add a forward frontmatter migration
> (above) that rewrites `coordinator: true → parentKind: arc-tracker`. Old items
> migrate forward on load; the boolean is retired without touching shipped files.

---

## Decomposition: the designing → ready gate

The single most valuable rule in the system. Promotion from **designing → ready**
fails unless the spec is genuinely one session of work. Concretely, the gate
rejects a spec that is too big to be one claimable unit:

- The `«SPEC_SECTION»` (e.g. `## Spec`) must exist.
- It must have **≤ 4 top-level bullets**.
- It must contain **no sequenced-pass keywords** (`Pass N`, `Step N`, `Phase N`,
  "first/second pass" …). These signal multi-session work masquerading as one item.
- **Override:** a `single-session-justified: <reason>` frontmatter field bypasses
  the gate, for the rare genuinely-atomic-but-wordy spec.

> Implementation note: in the reference tool these are **hardcoded constants** —
> the section names are string literals in the gate function (`## Intent`,
> `## Spec`) and the keyword blocklist is a single regex. They are genuine
> parameterization seams (the table below lists them), but adapting them is a
> *source edit at extraction time*, not a runtime config toggle. Pull them into a
> config object only if you expect to vary them after extraction.

When the gate fails, the fix is **decomposition** — split the item into children
(below), each its own claimable unit. Decomposition is the *default exit shape* of
the designing stage, not an exception.

The triage → designing gate is lighter: it just requires an `«INTENT_SECTION»`
(e.g. `## Intent`) so an item can't reach designing as a bare title.

### Children

A `new-child` verb mints letter-suffixed children of a parent
(`«PREFIX»-88` → `«PREFIX»-88a`, `«PREFIX»-88b`, …), one per supplied title. Each
child links back to the parent via `informs:`, and the parent's `informs:` gains
each child — so an *arc-tracker* parent closes when they all ship. An optional
**chain** mode additionally sets each child's `requires:` to the previous child,
forcing sequential execution. Mint children in one atomic commit.

The same verb family sets the parent's kind at mint time rather than by hand-editing
frontmatter: a `--kind` flag on `new-child` marks the parent an *arc-tracker*, and a
separate `new-record` verb mints a *record* parent straight into the terminal archive
(it ships on creation) while leaving its children live in the backlog. An *umbrella*
is created like any ordinary item and carries `parentKind: umbrella` — it points *up*
at its own parent, so it isn't minted through the child-spawning path.

### Coverage-add items: enumerate the debt before promoting

One class of item resists the decomposition gate because its true size is *hidden at
groom time*. A **coverage-add item** is any item whose deliverable is "apply
\<instrument\> to \<area\> for the first time" — a lint rule newly pointed at a
directory, a `strict` flag on a loose module, a test suite over a service that had
none, an accessibility audit on a surface never audited. Unlike a point fix (whose
cost you can read off the spec), a coverage-add item's real cost equals the area's
**latent debt**, which is unknown until the instrument first runs over the whole
area.

Two obligations make that cost visible *before* the item is handed out:

- **Enumerate at groom time.** Before promoting a coverage-add item designing →
  ready, run the instrument over the area **once** and paste its violations into the
  spec as a checklist. The instrument already exists as a runnable tool in the cases
  that matter, so the run is cheap; it converts a session-time surprise into a
  budgeted list. **Run the *whole* instrument over the *whole* area and cross-check
  its headline total** — a hand-picked file subset whose per-rule counts merely
  *look* complete will mis-size the work, and a per-rule *global* gate cannot flip to
  enforcing until that rule is globally zero, so a partial enumeration sizes the
  wrong program. Record the instrument's own total next to the checklist so a reader
  can confirm the list sums to it. (Worked failure: an enumeration that measured the
  logic files but silently dropped the UI layer reported a fraction of the real
  violation count — internally consistent, globally wrong.)
- **Split the point fix from the sweep.** "Fix this one instance" (cost knowable
  from the spec) and "bring the whole area instrument-clean and add the gate" (cost =
  latent debt) are *different units of work*. A title written for the one-liner that
  silently bundles the sweep hides the sweep's cost from estimation and scheduling.
  When both live in one designing item, split before promoting: one item for the
  named instance, one for the sweep with its debt enumerated as above.

---

## Release-on-pause and the adopt handoff

Multi-session work needs a way to put an item down without finishing it, such that
a *fresh session in a different worktree* can pick it up — without returning to the
original local state.

- **pause** (release-on-pause): drop the claim ref but **keep the item at
  `in-progress`**. The branch — not the worktree — carries the work. Because the
  claim is gone, the item reads as "nobody is holding this," which is the correct
  abandonment signal (claim-absence, per the liveness discussion). Commit a handoff
  note first (below).
- **adopt**: from a fresh worktree, re-acquire a paused (or dead-claimed) item:
  free the branch from its old (possibly locked/zombie) worktree, check it out
  here, and re-create the claim. The liveness probe gates this — a *live* claim is
  refused, a *dead* one is auto-cleared.

### The handoff block

A multi-session item carries a small, re-derivable **handoff block** in its body
(call it `«RESUME_SECTION»`, e.g. `## Resume`) with three labelled parts:

- **Done.** Committed work so far (with SHAs where it helps).
- **Next.** The single next concrete action.
- **In flight.** Open decisions, partial state, constraints (e.g. "these commits
  intentionally not on the trunk until phase N").

It is working state, not a contract — intended to be ephemeral, so add its heading
to the ship-time strip set (above) if you want it guaranteed gone on ship. Keep it
terse: it is a pointer back into durable state (commits, the dependency graph), not
a transcript. The validator surfaces its *absence* on an in-progress item as a
**non-blocking warning**, nudging the handoff without ever gating on it.

---

## New doc-only items reach the trunk immediately

When a session discovers new work and files a work-item for it, that item is a
**pure doc change** — one new markdown file, no code. The default close ceremony
would carry it to the trunk only when the current branch closes, which may be
sessions away. That delay has real cost: parallel sessions cannot see or claim the
item in the interim, and if the current branch is abandoned the item disappears
with it.

The fix: a doc-only item is **cherry-picked to the trunk under the same merge-lock
used for every trunk write**, as a standalone operation independent of the current
branch's close. The steps are:

1. **Commit the item in isolation** — not bundled with code — so it is
   cleanly cherry-pickable with no unintended diff.
2. **Acquire the merge-lock**, cherry-pick that commit to the trunk, push, release
   the lock. Because it is doc-only, no test gate is needed (or rather, any gate
   that auto-skips doc-only changes will auto-skip it — the discipline is the same
   machinery, not a special bypass).
3. **Leave the item on the current branch too.** At close, the branch-to-trunk merge
   sees identical content and git deduplicates it cleanly — no conflict, no
   double-add.

**Why use the merge-lock for something this cheap?** Uniformity. The lock is the
single point where trunk-ref writes serialize. Honoring it even for a doc-only
cherry-pick keeps the discipline uniform and avoids a class of "it's trivial so I
skipped the lock" races. The wait is short; correctness beats shaving it.

The rule applies only when all three hold: (1) the item is **new** work discovered
this session, not a refile of existing work; (2) it is **independent** of the
current branch's code — a pure new file, no code entanglement; (3) the merge-lock
is acquirable within the session. If any condition fails, let the item ride the
branch to close as usual.

---

## Reading state correctly (two traps)

1. **Never grep the raw item files for state.** A `grep 'stage: ready' *.md`
   matches any field-shaped line — example blocks in a README, draft fragments,
   this very doctrine — yielding *phantom items*. The tool is the read surface: it
   parses frontmatter and skips non-item files. (Defensive belt-and-braces: write
   the schema *example* in your README with a reserved non-matching id and the
   `stage:` enum spelled as a union, so a stray grep can't mistake the example for
   a live item.)
2. **Cross-branch reads come from the object store, not the working tree.** Verbs
   that *mutate the current branch* (claim, promote, ship, release) read and write
   the working tree — correct. Verbs that answer *"what is known across the whole
   project?"* (the picker, id-minting, validation) must read item presence and
   stage from **every local branch's committed state** (`ls-tree` + `cat-file`
   over `refs/heads/`), because a sibling worktree's branch may carry committed
   work this checkout has never seen. Cache the cross-branch index per process.

### The trunk-leak caveat

The trunk branch (`«BASE_BRANCH»`, e.g. `main`) is the one branch where an
`in-progress` stage is always spurious: the in-progress flip is meant to live only
on a feature branch, so if it reached the trunk, a merge carried it there by
accident. Cross-branch consumers should treat an `in-progress` item on the trunk as
phantom (exclude it from "blocked, in flight elsewhere" reasoning), and the
validator should flag it for cleanup.

---

## The invariant suite

A single `check` verb is the project's truth-keeper. In the reference implementation
it has two behaviorally distinct modes, plus a third reserved tier:

- **fast** — skips the expensive **dependency-graph validation** (dangling requires,
  cycles, duplicate ids) and the cross-branch stranded-worktree scan; everything
  cheap still runs (schema/parse, orphaned claim refs, priority-uniqueness, **parent
  stage-per-kind** — an arc-tracker not at `ready`/`in-progress`, an umbrella not at
  `designing`, a record not terminal is an error — and the **parent-ready-to-close**
  signal). Cheap enough for a pre-commit hook.
- **default** (full) — adds the dependency-graph validation and the stranded-worktree
  scan on top of the fast set. This is the real CI gate.
- **strict** — a forward-compat hook reserved for "treat warnings as errors too." In
  the reference impl it is currently *identical to default* because no warning-class
  checks exist yet; it exists so a warnings-as-errors tier can be added without a new
  verb. (The escalating fast → default → strict shape is the pattern to copy; just
  don't assume strict runs a distinct check set until you give it one.)

`check` never throws on the first problem — it accumulates and reports all of them,
so one run surfaces the whole mess rather than one error at a time.

---

## IDs are provenance, not identifiers

A work-item id (`«PREFIX»-42`) is a *project-management* handle. Keep it out of
**codebase identifiers** — the names of source files, test files, and the
`describe`/`test` labels inside them. A reader should learn what a module or test
covers from its name, not by translating an id back into "what that work was." This
keeps a clean seam between the tracker and the code.

- **Names describe intent, not tickets.** `failureCooldownInjury.test.ts`, not
  `«PREFIX»24Balance.test.ts`; `describe('whyCantScout', …)`, not
  `describe('«PREFIX»-31 — …', …)`.
- **One durable exception: a real sequence number that indexes a structure.** A
  migration's index (`migration65BackfillTimestamps.ts`) is a legitimate
  identifier — the index anchors a real array position. The redundant part to drop
  is the *ticket* id, not the structural number.
- **Ids stay welcome as provenance** — in comments, commit messages, and the
  terminal archive (`«SHIPPED_DIR»/«ID».md`, never deleted), so an id in a comment
  is a durable, greppable link to the *why*.
- **The boundary in one line:** codebase identifiers say *what*; the ids that link
  to *why* live in comments, commits, and the archive.

Why it pays off: the tracker churns — ids get superseded, renamed, merged — and a
codebase that embedded them ages into a decoder ring. Intent-named identifiers don't.

---

## Parameterization seams (binding → agnostic)

Everything project-specific is one of these slots. Lift the doctrine verbatim;
fill these for your project.

| Slot | What it is | Where it binds today |
|---|---|---|
| `«PREFIX»` registry | The set of id domains (features / infra / content / …) | id-minting + validation |
| `«BACKLOG_DIR»` | Active items directory | file-top constant in the tool |
| `«SHIPPED_DIR»` | Terminal archive directory | file-top constant in the tool |
| `«BASE_BRANCH»` | The trunk to merge from / detect leaks on | merge + trunk-leak checks |
| `«SPEC_SECTION»` / `«INTENT_SECTION»` | The section-name pair the promotion gates fire against | `canPromote` |
| Decomposition keywords | The sequenced-pass keyword blocklist | the designing→ready gate |
| `«RESUME_SECTION»` | The multi-session handoff block heading | the resume-block warning (opt-in) |
| `«CAP»` | Priority ceiling (e.g. 99) | priority validation |
| `«SESSION_UUID»` source | Env var the harness exports per session (reference: a Claude-Code-style session-id var) | claim reflog stamp + liveness |
| Liveness heartbeat source | The per-session artifact whose mtime proves a claim is live (reference: the harness transcript `.jsonl`) — **harness-coupled, not a constant swap** | the liveness probe |
| Claim namespace | `refs/claims/<id>` | already generic — no change needed |

The claim namespace is already project-neutral. In the reference implementation
most of these slots are **hardcoded constants** (string literals, a regex, file-top
path constants, function-parameter defaults) rather than entries in a single config
object — so swapping them is a source edit at extraction time. Centralize them into
a config object only if you expect to vary them per-deployment after the lift.

One slot is *not* a simple constant swap: the **liveness heartbeat source** (the
per-session artifact whose mtime proves a claim is live) is coupled to the host
harness. See *Claim liveness* — a consumer on a different harness must supply its
own heartbeat artifact or fall back to claim-absence only.

---

## Packaging the tool

The runnable side is one CLI (`bl`) plus a pure-function library it imports (the
library is where the unit tests bind). The pure library contains: the frontmatter
parser + migration chain, the eligibility/picker predicate, the dependency-graph
validator, the lifecycle state machine, **and the git-ref claim model**. The CLI is
the thin I/O shell around it (git calls, file reads/writes, argument parsing).

**One structural decision a consumer faces: do the *claim/branch-index/priority*
primitives and the *lifecycle/frontmatter* machinery ship together or split?** They
**share the parsed-frontmatter representation** — the cross-branch index returns
item objects whose shape the frontmatter parser defines — so they are coupled
through that one type. Two options:

- **Option A — split at extraction:** a `core` module (frontmatter + claim +
  branch-index + priority) and a `lifecycle` module (stage verbs + gates). Cleanest
  seam for a consumer who wants *only* the git claim/worktree primitives without the
  backlog lifecycle — but the split is a deliberate session with no output of its
  own.
- **Option B — ship the whole tool as one unit (recommended).** One fewer
  structural decision; a consumer wanting only the git primitives performs the split
  later, which is mechanical and need not precede any value delivery. The trade-off
  is that the claim-model primitives are not independently published — they live
  inside the one tool, documented as a section of it.

**This doctrine adopts Option B:** the file-per-work-item backlog tool is the unit
of extraction, and the parallel-worktree git primitives (claim model, branch index,
merge helpers) are documented as **a part of this tool**, not as a separately
packaged library. A downstream consumer who needs only those primitives takes the
A-split at that point — it is a known, mechanical follow-on, not a prerequisite.

> Consequence for a multi-part extraction effort: the git-primitives doctrine
> depends on this decision, so **this backlog-system lift is finalized before** any
> separately-scoped "just the worktree git primitives" lift — the latter points at
> the claim-model section here rather than re-deriving it.

---

## When to adopt this

This system pays for itself when **two or more agents or people work the same repo
concurrently** and a shared index would conflict, or when work routinely spans
multiple sessions and needs durable, re-derivable handoff. For a solo, single-stream
backlog it is more machinery than a flat list needs — though the decomposition gate
and the terminal archive are worth borrowing even then.

The non-negotiable core, if you adopt nothing else: **one file per item**, **state
in frontmatter**, **claims as exclusive-create git refs**, and **one pure
eligibility predicate**. Those four are what make it concurrent-safe; the rest is
ergonomics on top.
