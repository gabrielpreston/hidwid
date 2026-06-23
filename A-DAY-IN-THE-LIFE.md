# A day in the life: one slice, three depths

> **What this is.** [`OVERVIEW.md`](OVERVIEW.md) names the parts one at a time. This page
> shows them working at once — several concurrent sessions on one disk, across a single
> slice of one day. Read OVERVIEW first for the names; read this to watch them run.

> **How to read it.** One work item, start to finish, told three times. The first telling
> looks effortless. The second replays the same slice with the other worktrees visible,
> and shows why it only looked that way. The third runs the loop on a worse day, when the
> easy saves aren't enough, and shows what failing safely costs. The first two change what
> you can see. The third changes what goes wrong.

This slice only happens inside the setup the corpus assumes. If you haven't checked yours
against [When this fits, and when it doesn't](OVERVIEW.md#when-this-fits-and-when-it-doesnt),
do that first. The mechanisms below are welded to that shape.

## The cast

One afternoon. Four sessions, each in its own worktree under one `.git`, each nudged along
by the human in passing — a popup here, a decision there. One work item threads through the
whole slice: **`«PREFIX»-7`**, a small feature nobody has filed yet when the afternoon
starts.

- **Session A** is mid-way through `«PREFIX»-3`.
- **Session B** and **Session C** are each wrapping up, about to reach for the next thing
  on the board.
- **Session D** doesn't exist yet. It starts cold the next morning.

That's the cast. Watch `«PREFIX»-7`.

## First telling — the happy path

Session A is deep in `«PREFIX»-3`. It notices something next to the work — its own job,
clearly — files a ticket for it, `«PREFIX»-7`, and keeps going. A few minutes later Session
B needs its next thing and takes `«PREFIX»-7`. It builds it in its own worktree. The checks
pass at every commit. When it's done, the branch lands on the trunk: green, logged. The
whole thing took one afternoon. Nobody — no agent, no human — ever had to think about the
other sessions working alongside.

That's the afternoon from the outside. Nothing snags. Four plain moments in that paragraph
are each hiding something — A *files it and keeps going*, B *takes it*, *nobody thinks
about the other sessions*, the branch *lands clean*. The next telling reopens each one.

## Second telling — the near-disaster underneath

The same afternoon, with the other worktrees visible. Every place the first telling said
"just worked," something was holding it up.

**A filed it and kept going.** When A filed
`«PREFIX»-7`, it landed on the trunk right away, as its own small commit — not bundled into
A's feature work, not waiting for `«PREFIX»-3` to finish. That's why B could see it within
seconds. Had the ticket ridden A's branch to the end, B would have been blind to it all
afternoon, and the work would have lived only in A's context — gone if that window died.
The discovery survived because it was written to the repository, not the conversation.

**B took it — but it won a race to do that.** B and C reached for the next item at almost
the same instant, and both eyed `«PREFIX»-7`. Only one can own a thing. Claiming it is an
exclusive-create: one session's claim ref gets written, the other's create fails. The pick
and the claim are one step, so the same operation that gave B `«PREFIX»-7` gave C the next
eligible item, `«PREFIX»-8`. C had no conflict to handle and no retry to run — the losing
create was resolved inside the verb, never surfaced. Neither agent, nor the human, saw a
collision. The filesystem decided it. Before either began, each synced from the trunk, so
both built on the ground A's ticket had just landed on.

**Nobody thought about the other sessions — because the worktrees kept them apart.** B's
edits, B's index, B's half-finished commits sat in a directory no other session could
touch, though all four shared one `.git`. C working on `«PREFIX»-8` next door couldn't
corrupt B's tree, and B couldn't corrupt C's. At every commit, B's checks ran locally — the
`«gate»` — so nothing half-broken could even become a commit, let alone reach the trunk.

**The branch landed clean — but the landing was serialized.** When B finished, it went to
merge back. So, within the same minute, did C. Two sessions arriving at the one step that
can't run in parallel. A single lock ordered them: B took it, merged, released; C waited a
few seconds, re-synced onto B's just-landed work, re-ran its checks, and merged on top.
Their near-simultaneous writes to the shared records didn't collide either, for two reasons
worth keeping straight: the backlog is one file per item, so two new tickets are two new
files; the shared ledgers are append-only, so concurrent appends merge instead of fight.
The human saw two features land cleanly, and never learned how close they came.

The effortlessness was real. It was also earned. A claim only one session can win, an
isolation boundary, a gate at every commit, a lock on the one step that can't be concurrent
— take any one away and the happy path is where the corruption starts.

## Third telling — the day a save isn't enough

Those mechanisms catch the ordinary falls: the race, the collision, the half-broken commit.
The third telling is two harder cases — one the machinery still catches, but not for free,
and one it can't catch at all.

**The window dies — and the save isn't free.** Partway through `«PREFIX»-7`, Session B's
machine sleeps and the CLI is archived. The session is just gone, mid-task, no clean exit.
The durable floor reaches only as far as B's last commit — and as far as the last time B
*committed* its handoff note. The note has to be committed, not just typed into the editor,
or it dies with the window like everything else. So there is a real loss: whatever B did
since its last commit is gone, and a fresh session pays to rebuild B's half-formed intent
from a static note instead of inheriting a live train of thought. What survives is the
branch — every committed step — and the committed note: what's done, what's next, what's in
flight. The next morning Session D starts cold, inherits nothing from B's dead context, reads
those artifacts, pays the rebuild cost once, and adopts the work. It doesn't resume B's
machine state; there's nothing to resume.

Adoption has one thing to get right: telling a dead claim from a held one. A clean stop
drops its claim on the way out, so an absent claim means the work is free — no probe needed. B
got no such chance; it crashed with its claim still held. Treat that orphaned claim as live
and `«PREFIX»-7` is frozen forever, owned by a session that no longer exists. So for a crash,
adoption runs a liveness probe: it reclaims a claim with no live session behind it, and
refuses one that's genuinely held. Either way a signal decides it, not a guess. The branch
carried the work across the gap — but only as far as the last commit reached.

**The plan turns out to be wrong.** As D works, it hits something the plan didn't foresee —
a load-bearing assumption that can't hold. This is the failure with no tidy catch. No lock
or probe repairs a wrong plan, and pushing through would bury the problem in shipped code.
The discipline is the opposite of momentum: stop. Not push through, not quietly invent a
workaround — surface it to the human, say what's wrong, let the one judgement-holder decide.
That costs something worth saying plainly: the session may end with nothing merged, the
ticket sent back to be rethought, the spend on the attempt written off. The same posture
holds when a test D didn't write fails — own it, don't step over it — and when a partial
migration can't be made safe, where the branch is paused and left unshipped rather than
forced onto the trunk to keep the afternoon looking clean. Failing safely buys a green trunk
and a true account for the next session, and pays for them with the work that didn't land. A
price paid in the open. That's the cheap version of this failure, not the free one.

The first two tellings skip this layer. The corpus can't. The claim is not "it never fails."
The claim is that when it fails, the failure is visible, the work is recoverable, and the trunk
stays trustworthy — a zero on the cost ledger or a surfaced failing test is signal, not
something to round away.

## What each beat was holding up

The six beats, in order — with the idea and chapter each realizes. Read top to bottom, the
table is a story-shaped index into the corpus: the same material the chapters define on
their own, in the order a real afternoon meets it.

| Beat | What runs | Idea it realizes | Chapter |
|---|---|---|---|
| **Discovery** | A new item is filed and lands on the trunk at once, so live sessions see it and it survives the session that found it | [durable handoff](OVERVIEW.md#4-durable-re-derivable-handoff) · [concurrency](OVERVIEW.md#2-concurrency-without-collision) | [backlog-system](backlog-system-doctrine.md) · [session-lifecycle](session-lifecycle-doctrine.md) |
| **Entry** | The router classifies the session; sync-from-trunk first; an exclusive-create claim settles a two-session race | [front door in](OVERVIEW.md#3-one-front-door-in-one-gate-out) · [concurrency](OVERVIEW.md#2-concurrency-without-collision) | [session-lifecycle](session-lifecycle-doctrine.md) · [backlog-system](backlog-system-doctrine.md) |
| **Work** | An isolated worktree on one shared `.git`; the gate fires at every commit | [concurrency](OVERVIEW.md#2-concurrency-without-collision) · [mechanical enforcement](OVERVIEW.md#5-mechanical-enforcement-over-habit) | [worktree-git-workflow](worktree-git-workflow-doctrine.md) · [engineering-discipline](engineering-discipline-doctrine.md) |
| **Window death** | A session vanishes mid-task; its branch + committed handoff note survive; a fresh session adopts — on claim-absence, or a liveness probe when a crash never released the claim | [durable handoff](OVERVIEW.md#4-durable-re-derivable-handoff) | [session-lifecycle](session-lifecycle-doctrine.md) · [backlog-system](backlog-system-doctrine.md) · [worktree-git-workflow](worktree-git-workflow-doctrine.md) |
| **Close** | Two sessions hit the merge step together; one lock serializes them; the loser re-syncs and retries | [concurrency](OVERVIEW.md#2-concurrency-without-collision) | [worktree-git-workflow](worktree-git-workflow-doctrine.md) |
| **Ship** | The gate runs once at the boundary; the trunk stays green; the cost is logged and attributed | [gate out](OVERVIEW.md#3-one-front-door-in-one-gate-out) · [measured cost](OVERVIEW.md#6-measured-attributed-cost) | [session-lifecycle](session-lifecycle-doctrine.md) · [cost-hygiene](cost-hygiene-doctrine.md) |

Two ideas never appear as a beat — [judgement stays
human](OVERVIEW.md#1-judgement-stays-human-typing-goes-to-agents) and [defend the
experience in review](OVERVIEW.md#7-defend-the-experience-in-review). Those the human
carries, not the loop. They surface in the third telling, the moment D stops and flags the
wrong plan: where the machinery hands the decision back.

---

That's one slice at three depths. The first telling is the day-to-day. The second and third
are what has to hold underneath for the first to stay effortless. Descend into a chapter
from the [map](OVERVIEW.md#the-map) and you're reading the static definition of something
you've just watched run.
