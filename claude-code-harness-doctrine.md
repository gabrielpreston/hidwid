# Working with the harness: permission-aware actions, self-healing hooks, and a run-until-verifiable loop

> **In one line.** Shape the agent's actions to the harness's permission model and make
> its hooks heal themselves — so the host's control surface becomes leverage instead of
> low-grade friction.

**TL;DR — the mechanisms**
- One command per permission match: keep actions on the allowlist, not in prompts.
- Pick the lowest-friction tool for the job (structured editor / batch substitute / symbol nav).
- Self-repairing hook installation: enforcement that heals instead of silently dying.
- One validator, two callers (local gate + CI) with an auditable bypass seam.
- A run-until-verifiable loop the agent can't declare done without proving.
- A periodic hygiene sweep that mines the harness's own telemetry for friction.

**Read this when** you're tuning how agents act under a gated CLI and its hooks.
**Big ideas:** [judgement stays human](OVERVIEW.md#1-judgement-stays-human-typing-goes-to-agents) · [mechanical enforcement over habit](OVERVIEW.md#5-mechanical-enforcement-over-habit)
**Depends on:** [engineering discipline](engineering-discipline-doctrine.md) (the gate its hooks keep alive)

---

How to make an agentic coding harness *pull its weight* instead of fighting it —
by shaping the agent's actions to the harness's permission model, making the local
enforcement hooks heal themselves instead of silently dying, sharing one validator
between the local gate and CI, closing a verifiable-completion loop the agent
cannot game, and periodically mining the harness's own telemetry for the friction
worth automating away. The premise is that a coding agent runs inside a host with
an opinionated control surface — a permission matcher, lifecycle hooks, a stop
gate, a session-telemetry log — and that surface is either a source of constant
low-grade friction or a set of leverage points, depending on whether your
conventions are built *for* it.

This is project-agnostic doctrine extracted from a working implementation, but it
is the most **platform-coupled** doc in this set: its audience is teams running an
agentic coding harness with a string-matched permission allowlist, installable
lifecycle/stop hooks, and a per-session usage log — Claude Code is the reference
host, and several patterns assume primitives in that shape. Where a concrete noun
appears — a gate command, a bypass sentinel, a directory layout — it is called out
as a **`«slot»`** you fill for your own project; the
[parameterization table](#parameterization-seams) near the end collects every slot
in one place. Patterns that bind tightly to a specific harness primitive are
flagged inline so you can tell the portable principle from the host-specific wiring.

Two companions carry machinery this doc leans on rather than re-derives. The
[engineering-discipline doctrine](./engineering-discipline-doctrine.md) owns the
**pre-commit gate** (the lint/type/test triad) and the **transitive-ownership**
posture; the goal loop of §5 and the validator of §4 are enforcement points
*around* that gate, not replacements for it. The
[session-lifecycle doctrine](./session-lifecycle-doctrine.md) owns the **temporal
envelope** (open → work → close); the run-until-verifiable loop is a session-time
tool that runs *inside* that envelope, and the hygiene sweep of §6 is a periodic
out-of-band session of its own.

---

## Why this shape

An agent acting inside a harness is not a person at a terminal. It cannot feel a
permission prompt as friction and route around it; it stalls, or worse, silently
fails. It does not remember last month's editor-config incident that broke the
hooks. It will happily declare a task "done" the moment the obvious checks pass,
having quietly weakened a test to get there. And it generates a dense, machine-
readable exhaust — every tool call, every rejection — that no one reads.

Each of those is a structural property of the host, and each has a structural
answer:

- **The permission matcher reads whole command strings, not intent.** So the way
  an agent composes a shell command determines whether a pre-approved action
  prompts or runs. Conventions that mirror the matcher's grain (§1) and that pick
  the lowest-friction tool for a job (§2) remove prompts the agent would otherwise
  stall on.
- **Local hooks are fragile and their breakage is silent.** A hook that an editor
  has unhooked enforces nothing while *looking* installed. So hook installation
  must be self-repairing and drift must be *observable* (§3), and the one
  enforcement that can never be bypassed must live server-side, shared with the
  local copy from a single source (§4).
- **An agent's "done" is untrustworthy without a gate it cannot move.** A loop
  that grades itself against a fixed, integrity-checked rubric — and physically
  cannot end its turn until the rubric is green — converts "I think this is done"
  into "the gate says this is done" (§5).
- **The harness's telemetry is the cheapest source of truth about friction.** A
  periodic sweep that reads the rejection log and usage signal turns accumulated
  annoyance into a small number of mechanical fixes, and prunes the auto-loaded
  context that has gone stale (§6).

The unifying move is the same one the engineering-discipline doctrine makes for
correctness, applied to the *harness relationship*: find the place where you are
relying on the agent (or yourself) to remember a convention, and make the host
remember it instead — in how commands are shaped, in how hooks reinstall, in what
the stop gate refuses to let pass, in what the telemetry sweep surfaces.

---

## Part 1 — one command per permission match

### The posture

**Each shell action the agent takes is one command, matched as one string against
the allowlist.** A permission allowlist matches *patterns against the entire
command string*. An allow rule like `«tool»(git branch*)` matches `git branch -m
feat-x`, but it does **not** match `git branch -m feat-x && git merge main` — the
`&&` makes the whole string a different, unmatched thing, so a chain of two
individually-approved commands prompts anyway. For an agent that cannot dismiss a
prompt fluidly, that stall is pure waste.

So the convention is: **never join commands with `&&`, `||`, or `;` in a single
call.** Each command is issued on its own. Independent commands are issued as
parallel calls in one turn; sequential ones as separate calls across turns. The
result is that every action stays inside the grain the matcher actually sees, and
a pre-approved action runs without a prompt.

### The two exceptions that prove the rule

Two cases are *not* workflow chains and stay as one call:

- **A pipe into a formatter** (`… | head`, `| jq`, `| grep`) is one logical
  operation — a single command whose output is shaped — not two steps. It stays.
- **The atomic stage-and-commit** (`«vcs» add <paths> && «vcs» commit -m "…"`) is
  one indivisible operation: splitting it risks a half-staged tree if the second
  step is skipped. The `&&` here joins one commit operation, not a workflow, so it
  stays as a single call.

The distinction is *logical atomicity*, not the literal presence of `&&`. A chain
that exists only to save a turn is the banned case; a chain that is one operation
is fine.

### Adapt this

*Slots to fill:* `«tool»` (your permission-rule syntax for shell commands), the set
of formatters your pipe-exception covers. The portable invariant: **shape each
agent action to match the permission matcher's unit — one logically-atomic command
per call — so pre-approved work never stalls on a spurious prompt.** This pays off
the instant you maintain an allowlist; without it, the allowlist's value leaks away
one chained command at a time.

---

## Part 2 — pick the lowest-friction tool for the job

### The posture

**The harness's default tool steering is tuned for the common case; for the
bulk-mechanical case it is wrong, and the convention should say so explicitly.** A
good harness steers the agent toward its safe structured editor (a string-replace
tool that demands a prior read and enforces match-uniqueness) and *away* from raw
`sed`/`awk`. For a one-off targeted edit that steering is correct — the structured
editor's uniqueness check is real safety. But for a **bulk mechanical edit** — a
verifiably-unique string substitution across many files — the same steering forces
the agent to read every file just to satisfy the editor's prior-read requirement,
then send each match with enough surrounding context to be unique. That is 10–200×
the tokens of a single `sed -i`, for no safety gain when the target is provably
unique.

So the project carries an explicit *override*: a rule that names the conditions
under which batch `sed -i` is the right tool, and puts the substitution command on
the allowlist so it does not prompt. The three conditions, all of which must hold:
the change is purely mechanical (string-for-string, not structural); the target is
verifiably unique (confirmed by a search first — no substring collisions); and it
spans several files (or one file so large that reading it to edit dwarfs the
change). The mandatory final step is to **re-run the gate** — the batch tool has no
uniqueness check, so the type/lint pass is what catches a bad substitution the
structured editor would have refused.

### Navigate by the symbol graph before you grep

A bulk rename *starts* with a navigation question — "who uses `X`?" — and a
text search is the wrong tool for it. A search over source text returns substring
collisions (`user` also matches `userId`, `superuser`) and forces a read of
every hit to disambiguate. If the harness exposes a **language-server / AST
navigation** capability, that is the right tool for the "who uses this symbol?"
question: it returns the *true* reference set with zero substring noise and zero
file reads — a large token win on the navigation hot path, and the precondition for
a safe rename. Two host-specific cautions the reference implementation learned the
hard way (flagged as harness-coupled — your tool may differ):

- **Cold-start under-counts silently.** A server that indexes the project lazily
  returns a *plausible, well-formed under-count* — not an error — before the graph
  finishes loading. A suspiciously small result (e.g. same-file-only) means "not
  yet indexed," not "no references." Issue one warm-up query, wait for the index to
  settle, and re-query before trusting a count.
- **Anchor a property reference at a use site, not its declaration.** Some servers
  under-return when a struct/object *property* reference is anchored at the
  declaration; anchor at an actual usage (`obj.prop`) instead.

For an AST-anchored *rename* whose uniqueness is dubious (short or common
identifiers — exactly the case the structured editor handles better than `sed`),
prefer the language server's rename-symbol operation: it is AST-aware and touches
only true references. `sed` stays correct for non-symbol substitutions (import
paths, version strings, magic numbers) where there is no symbol to anchor.

### Adapt this

*Slots to fill:* `«structured-editor»` (your safe edit tool and its prior-read /
uniqueness contract), `«batch-substitute»` (the raw tool you allowlist for bulk
edits), `«symbol-nav»` (your language-server reference/rename capability, if any),
`«gate»` (the validate step — see the engineering doctrine). The portable
invariant: **match the tool to the job's shape — structured editor for targeted
edits, batch substitution for verifiably-unique mechanical sweeps, symbol-graph
navigation for "who uses this" — and let an explicit project rule override a
harness default that is tuned for a different case.** The non-negotiable: a batch
substitution without a uniqueness check is only safe because the gate re-runs after.

---

## Part 3 — self-repairing hook installation

### The posture

**Local enforcement hooks must reinstall themselves and make their own breakage
observable, because a silently-disabled hook enforces nothing while looking
installed.** A version-controlled hook (a pre-commit/pre-push/commit-msg script
checked into the repo, activated by pointing the VCS's hooks-path at the tracked
directory) has two independent failure modes, and a common editor's VCS integration
trips both:

1. **The hooks-path gets overwritten.** The editor rewrites the hooks-path config
   to an absolute default directory, which wins over the relative tracked path —
   VCS config is last-writer-wins, and the editor writes last. The hooks are now
   unreachable.
2. **The hook files lose their executable bit.** With file-mode tracking disabled
   (another editor/filesystem artifact), the VCS ignores the on-disk exec bit, and
   a non-executable hook is *silently skipped* with at most a buried warning.

Either one disables the hooks while the repo still *looks* hooked. The defense is a
small **idempotent installer** that repairs both — re-points the hooks-path,
re-applies the exec bit — and is wired to run on the natural reinstall vector
(the dependency-install lifecycle script, e.g. a `prepare`-style post-install
hook), so a fresh checkout or a routine reinstall heals it. Committing the hook
files with their executable bit already set makes fresh checkouts executable
regardless of the file-mode setting.

### Repair on a deliberate vector; only *warn* on the shared one

The subtle half is *where* the repair is allowed to fire. The hooks-path lives in
**shared** VCS config (one setting for the base repo and every worktree against it).
A repair triggered from session start in one worktree would flip that shared setting
for every parallel worktree at once — and could activate hooks for a sibling session
that does not yet carry the automation bypass (§4), breaking it. So the installer
runs in two modes:

- **Repair mode** fires on the *deliberate, single-actor* vector (dependency
  install) — a fresh checkout, an explicit reinstall. It is safe there because it is
  not racing parallel sessions.
- **Check-only mode** fires at *session start* and only **warns** about drift; it
  never repairs. The warning names the broken state and points at the repair vector.

The principle: **a self-healing mechanism that writes shared state must heal only on
a vector where it owns the write, and merely *observe* on vectors where a write
would race.** Observability without a racing write is still a large improvement over
silent breakage.

### Adapt this

*Slots to fill:* `«hooks-dir»` (the tracked hook directory), `«hookspath-config»`
(the VCS setting that points at it), `«install-vector»` (the lifecycle script that
runs the repair), the editor/host that causes the drift in your environment. The
portable invariant: **make hook installation idempotent and self-repairing on a
non-racing vector, warn-only on a shared-state vector, and never let a disabled hook
masquerade as an installed one.** The corollary: a local hook is necessary but never
sufficient — the non-bypassable layer is §4's CI backstop.

---

## Part 4 — one validator, two callers, and an auditable bypass

### The posture

**A check that must hold at both the local commit and the server-side merge is
written once and called from both places.** The reference case is a commit-message
prefix validator — a single zero-dependency script (a regex over the allowed
`«type-prefix»` grammar) that is the *single source of truth* for two callers: the
local commit-message hook (fast feedback, bypassable) and the CI job on pull
requests (the non-bypassable backstop). Writing it once means the grammar cannot
drift between the local gate and CI; a zero-dependency script means no toolchain to
keep in sync.

The validator's grammar must admit not just the *documented* human prefixes but
also any subjects your *automation* emits (a backlog tool's claim/ship commits, for
instance) — rejecting those would break the lifecycle that depends on them. And it
skips the machine-generated subjects the VCS itself produces (merge/revert/fixup
commits) rather than failing them.

### The bypass sentinel: local-only, never CI

Re-enabling local hooks must not block the automated lifecycle flows (claim, ship,
the close-session merge) that predate the hooks and rely on them staying inert.
Those flows set an explicit **bypass sentinel** — an environment variable
(`«hook-bypass»`) — on their *own* commit invocations, and the local hooks early-exit
when it is set. Three properties make this safe:

- **It is an explicit sentinel, not TTY detection.** Commits issued through tooling
  are non-interactive too, so "is there a terminal?" cannot distinguish automation
  from a human; an explicit env flag can.
- **A human commit never sets it,** so real work always runs the full local gate.
- **CI never honors it.** The bypass is for the *local* layer only; the server-side
  backstop re-runs the same checks unconditionally and is the floor that holds when
  every softer layer is skipped.

This is the §2-of-the-engineering-doctrine "two enforcement layers, only the local
one bypassable" rule, realized: the shared validator is the *what*, the sentinel is
the *who may skip the local copy*, and CI is the *what can never be skipped*.

### Adapt this

*Slots to fill:* `«type-prefix»` (your commit-message grammar), `«hook-bypass»` (the
env sentinel automated commits set for the local layer only), `«ci»` (the
non-bypassable re-run), the automation subjects your grammar must admit. The
portable invariant: **one validator, called from the local hook and from CI;
automation may bypass the local hook via an explicit auditable sentinel; CI honors
no bypass.**

---

## Part 5 — the run-until-verifiable loop

### The posture

**An agent's claim of "done" is replaced by a loop that grades itself against a
fixed rubric, cannot end its turn until the rubric is green, and is structurally
prevented from gaming the rubric.** Left to itself, an agent declares completion the
moment the visible checks pass — and under pressure to pass, it may weaken a test,
skip one, or quietly drop coverage to get there. The loop closes both gaps: it makes
*green* the only exit, and it makes *cheating toward green* a blocking violation.

The mechanism splits cleanly into three pieces, and the split is the point:

1. **A pure, runner-injected core.** All the decision logic — resolve the goal, parse
   the gate result, check rubric integrity, decide continue/done/escalate — lives in
   pure functions with no process spawning and no model calls. The IO (running the
   gate commands, listing changed files) is *injected*, so the core is deterministic
   and unit-testable in isolation. This mirrors the backlog tool's pure-core /
   IO-shell split and is what makes the loop's own logic trustworthy.
2. **A stop-hook verifier.** The harness's *stop hook* (the hook that fires when the
   agent tries to end its turn) reads a sentinel file; if the loop is armed, it runs
   the gate, checks integrity, and either **allows** the turn to end (goal met, or
   the iteration budget is exhausted) or **blocks** it — emitting a continuation
   directive the agent must act on, and advancing the iteration counter. The agent
   literally cannot stop while the gate is red.
3. **A deterministic rubric-integrity guard.** Before accepting green, a pure check
   compares a baseline snapshot against the current state and refuses these
   anti-patterns: the test count dropped, coverage dropped, a test file was edited
   when the goal was not itself a test task, or a protected path was touched. "Reach
   green by weakening the rubric" becomes a detected, named violation rather than a
   silent success.

### The guarantees that keep it safe

A self-perpetuating loop is dangerous if any of its safety properties is missing.
This one carries four:

- **Termination is guaranteed by an iteration budget, not by good behavior.** The
  loop continues only while `iteration < maxIterations`; on the budget it
  *escalates* (allows the turn to end with a "not met after N turns" report) rather
  than looping forever. The budget is clamped to a sane positive ceiling so a
  malformed input cannot create a runaway.
- **It fails open.** Any error anywhere in the verifier — a corrupt sentinel, a
  crashed gate — exits in the *allow* state. A verifier bug can never trap a session.
- **It is zero-cost when disarmed.** No sentinel file → the hook exits immediately.
  Normal sessions pay nothing.
- **The integrity policy is layered.** The interactive entry point may treat
  integrity violations as *warnings* (a human is watching); the unattended/automated
  entry point treats them as *hard blocks* (no human to catch a laundered green).
  Same pure check, different blocking policy at the boundary.

The portable shape behind all of this: a **completion gate with a runner-injected
grader, an iteration budget, and an integrity check the loop cannot satisfy by
cheating** — "run until verifiably done" instead of "run until the agent says done."

### Adapt this

*Slots to fill:* `«stop-hook»` (the harness's end-of-turn hook), `«gate»` (the
commands the grader runs — your lint/type/test triad), `«goal-source»` (where the
goal statement comes from — an explicit condition or a work-item's spec section),
`«protected-paths»` (the surfaces the loop may not edit unattended), the
warn-vs-block policy for your interactive and unattended entry points. The portable
invariant: **separate the pure decision core from injected IO; gate turn-completion
on a green, integrity-checked rubric; guarantee termination with a clamped budget;
fail open.** This pattern needs a harness with a programmable stop hook — it is the
most host-coupled mechanism in this doc, and the one with the highest leverage where
the primitive exists.

---

## Part 6 — the periodic hygiene sweep

### The posture

**The harness's own telemetry is mined on a cadence to convert accumulated friction
into a few mechanical fixes, and to prune the context that has gone stale.** An
agentic harness emits a dense exhaust — every tool call, every rejection, every
loaded rule — and on a quarterly cadence (or after a major workflow change) a
dedicated sweep reads it. The sweep distinguishes three signal classes that no
single source captures:

- **Hard friction — rejections.** Parse the session logs for tool calls the human
  *rejected*. A repeated rejection of a pre-approvable action (a path mistake, a
  command that should be allowlisted) is a candidate for a mechanical fix.
- **Soft friction — accepted-but-wasteful.** A separate signal: the same files
  re-read across many turns, redundant repeated invocations, edits fully reverted.
  The rejection log sees "no"; this signal sees "yes, but wasteful." Both are
  hygiene.
- **Staleness — auto-loaded docs that no longer fire.** Every auto-loaded rule or
  skill costs context on every turn. Scanning the transcript corpus for each
  rule/skill filename yields a last-referenced date and a recent-hit count; a
  zero-hit file is a *staleness candidate*.

### Triage by pattern, and never auto-delete

Two disciplines keep the sweep from doing harm:

- **A signal earns a mechanical fix only by pattern, not by count.** A rejection
  becomes a permission-deny rule only if it recurs, has no legitimate approved
  siblings, and expresses as a clean glob. Everything else becomes a rule-file
  edit or a human-readable note, not a settings change. A single loud count is a
  pointer to investigate, not a mandate to automate.
- **Staleness is advisory; the human makes every cut.** Silence is *weak* evidence:
  a rarely-referenced rule can be load-bearing (a data-migration rule that protects
  one rare edge case is the canonical example — it almost never appears in
  transcripts yet is never removable). A zero-hit count only *nominates* a file for
  an end-to-end human read; it never auto-removes anything. The deletion criteria
  are conjunctive — stale **and** low-value **and** no protecting edge case, all
  three confirmed by reading the file — precisely so silence alone cannot trigger a
  cut.

The output of a sweep is a small, justified set of changes: a few permission-deny
globs, a rule-file tweak or two, a handful of pruned stale entries — each traceable
to a pattern in the telemetry. The discipline is what keeps it from degenerating
into either noise-chasing or over-eager deletion.

### Adapt this

*Slots to fill:* `«telemetry-log»` (where the harness writes per-session tool-use
records), `«usage-insight»` (the soft-friction signal source, if your host has one),
`«settings»` (the permission/deny config a mechanical fix edits), the auto-loaded
surfaces whose staleness you scan. The portable invariant: **mine the harness
telemetry on a cadence; promote only pattern-backed friction to a mechanical fix;
treat staleness as an advisory nomination a human confirms by reading — never an
auto-delete.**

---

## Parameterization seams

| Slot | Source binding | Agnostic form |
|---|---|---|
| `«tool»` | the shell-command permission-rule syntax | your allowlist's pattern for matching shell commands |
| `«structured-editor»` | the safe string-replace edit tool | your editor with a prior-read + match-uniqueness contract |
| `«batch-substitute»` | `sed -i` (allowlisted for bulk edits) | the raw substitution tool you allow for verifiably-unique sweeps |
| `«symbol-nav»` | the language-server reference/rename capability | your AST navigation for "who uses this symbol" and AST-safe rename |
| `«hooks-dir»` | the tracked hook directory | where your version-controlled hooks live |
| `«hookspath-config»` | the VCS hooks-path setting | the config that points the VCS at your tracked hooks |
| `«install-vector»` | the post-install `prepare` script | the lifecycle script that runs the idempotent hook repair |
| `«type-prefix»` | the commit-message prefix grammar | your allowed commit-subject grammar, admitting automation subjects |
| `«hook-bypass»` | the `BL_HOOK_BYPASS` env sentinel | the auditable flag automation sets to skip the *local* hook only |
| `«ci»` | the CI workflow | the non-bypassable server-side re-run of the same checks |
| `«stop-hook»` | the harness end-of-turn hook | the programmable hook that fires when the agent tries to stop |
| `«gate»` | the lint/type/test triad | the commands the completion grader runs (see engineering doctrine) |
| `«goal-source»` | an explicit condition or a work-item `## Spec` | where the loop reads the goal it grades against |
| `«protected-paths»` | UI + the lead-owned shared primitives | the surfaces the loop may not edit unattended |
| `«telemetry-log»` | the per-session tool-use JSONL | where the harness records each session's tool calls and rejections |
| `«usage-insight»` | the soft-friction insight signal | the host signal for accepted-but-wasteful patterns, if any |
| `«settings»` | the user/project permission config | the config a pattern-backed mechanical fix edits |

---

## On packaging these as runnable artifacts

This doc is deliberately scoped to the **portable doctrine** — the principles above
are useful to any team on a comparable harness. The *runnable* artifacts behind them
(the goal loop's pure core + stop-hook verifier, the hook scripts, the
commit-message validator) are the cleanest standalone lifts in the corpus: the
validator is a near-verbatim lift with almost no per-project wiring, and the goal
loop's pure core is fully harness-agnostic (IO is injected). Packaging them is
deferred pending the parent extraction's distribution-shape decision — but the
evidence points one way worth recording: these surfaces map **1:1 onto the
plugin layout** of a mature agentic-harness plugin ecosystem (`commands/` +
`hooks/` directories install verbatim). The one wrinkle a packaging decision must
resolve is that hooks and commands install *verbatim* while the runnable tooling
needs per-project config (gate commands, the bypass-sentinel name, directory
layout) — so a plugin target implies a thin parameterized-config seam *inside* the
plugin for the tool. Until that decision lands, this prose doctrine stands on its
own; adopt the principles by hand-filling the slots above.

---

## When to adopt this

These patterns pay off at different points, so reach for each as its pressure
arrives:

- **One command per permission match (§1)** the moment you maintain an allowlist
  for an agent — it is the cheapest win here and prevents the allowlist's value from
  leaking away.
- **Tool-fit conventions (§2)** once the agent does enough bulk-mechanical or
  large-rename work that the structured editor's per-edit overhead is a visible
  cost.
- **Self-repairing hooks (§3)** as soon as you have local hooks *and* contributors
  using an editor whose VCS integration fights them — which is most teams.
- **The shared validator + bypass (§4)** the first time a check must hold at both
  the local commit and the merge, and automation needs to commit without tripping
  the local copy.
- **The run-until-verifiable loop (§5)** when you let an agent run multi-turn toward
  a verifiable goal and have been burned by a premature or laundered "done" — and
  only if your harness exposes a programmable stop hook.
- **The hygiene sweep (§6)** once the harness has produced enough telemetry that
  friction has accumulated and the auto-loaded context has started to drift stale —
  typically a quarter or two in.

The non-negotiable core, if you adopt nothing else: **shape the agent's actions to
the permission matcher's grain (§1), make local hooks self-healing and back them
with a non-bypassable CI re-run of a shared validator (§3–4), and never trust an
agent's unverified "done" where a stop-hook completion gate can hold it (§5).**
Everything else — the tool-fit override, the telemetry sweep — is leverage on top of
that frame. The unifying move never changes: the host has a control surface; build
your conventions *for* it, so the harness enforces them instead of fighting you.
