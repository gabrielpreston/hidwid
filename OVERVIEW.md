# Start here: what this corpus is and how it fits together

> **In one line.** A working playbook for running a real software project where one
> person directs a team of AI coding agents — the mechanisms that keep that fast and
> the reasons each one earns its place.

> **Built for one setup — check yours first.** This came out of a *single developer*
> running the Claude Code CLI on a *local machine*, with agents coordinating through
> *one shared git checkout* and *git worktrees* for parallelism. That shape is
> load-bearing: the claiming, the merge-lock, and the liveness probe are welded to it.
> If you work on a team, run cloud agents with no shared disk, use a non-git VCS, or
> require pull requests to merge, parts of this won't transfer as written — read
> [When this fits, and when it doesn't](#when-this-fits-and-when-it-doesnt) before you
> adopt anything.

## The pitch

Coding agents can now do most of the typing on a software project. That sounds like a
productivity story, and it is, but it quietly creates a different problem: a single
person can now set a dozen agents working at once, across a dozen copies of the same
repository, and those agents will cheerfully overwrite each other's work, re-solve the
same ticket twice, merge half-finished code, and agree with each other while being
wrong.

This corpus is the discipline that keeps that from happening. It was pulled out of one
project where a single maintainer runs many concurrent agent sessions against one
codebase, and it collects the parts that turned out to be load-bearing — how a session
starts and ends, how work is claimed so two agents never grab the same thing, how
parallel copies of the repo merge back without corrupting each other, when to spawn a
team of agents versus do it yourself, and how to keep the codebase correct when the
"people" editing it are stateless and turn over every session.

It is **prose to read and adapt**, not a tool to install. Each mechanism is written
with its reasoning attached, so you can judge whether it fits your situation before you
adopt it — and skip the ones that don't. Everything specific to the source project is
pulled out into a fill-in-the-blank **slot** (written `«like-this»`), so the doctrine
around it lifts cleanly into a project of your own.

If you direct AI agents on a codebase and have felt the coordination cost start to bite
— merge collisions, lost context between sessions, work you can't account for — the
chapters below are the defenses, each with the incident that drove it.

## When this fits, and when it doesn't

This corpus is one person's answer to "how do I run a fleet of coding agents without it
turning into chaos," written down so you can see the moving parts and take what works for
you. It is not a standard to conform to. Before you adopt a mechanism, check it against
the setup the whole thing assumes — because some of the defenses are welded to that setup
and quietly stop working outside it. A mechanism that silently half-works is worse than
one you knowingly skipped.

The setup it was built for: a **single developer**, running the **Claude Code CLI on a
local machine**, with agents coordinating through **one shared git checkout** and using
**git worktrees** for parallelism. Each line below is a way your situation might differ —
what still transfers, what doesn't, and why.

- **Concurrency — worktrees on a shared filesystem.** Fits when every session is a git
  worktree under one `.git` on one machine. The isolation, the merge-lock, and the
  stale-claim cleanup are single-machine, single-filesystem mechanisms (an OS file lock,
  a local ref). Agents spread across machines, or a cloud fleet with no shared disk, need
  different primitives for all three. The *ideas* — isolate work, serialize the merge,
  probe liveness — carry; the *implementations* don't.
- **Version control — git, and its atomic ref-creation.** Claiming a work item leans on
  git's exclusive-create (`O_CREAT|O_EXCL`) ref semantics on a local filesystem, so
  exactly one session wins a race for the same ticket. A different VCS, or git reached
  over a network backend, needs its own atomic primitive — verify it before you trust the
  claim model not to hand the same work to two agents.
- **Harness — a permission-gated CLI with a heartbeat.** Pause/resume and liveness ride a
  session heartbeat the Claude Code harness exports. Porting to a different agent (or a
  raw API loop) is the harder half: swapping the filesystem is easy, supplying a
  compatible liveness signal is not.
- **One human in the loop.** Every design call, type-system lock, and integration
  decision routes to a single, continuously-available lead who is the last word. On a
  team, "ask the lead" turns into a bottleneck and "who owns the trunk" becomes a real
  question this doctrine doesn't answer — the coordination model assumes exactly one
  arbiter.
- **Trust — broad local permissions, one reviewer.** Agents here run with wide local tool
  access, and the human is the only gate before a merge. Least-privilege agent
  permissions, mandatory multi-reviewer sign-off, or compliance that gates every
  AI-written change all break that assumption. The mechanics still run; the trust posture
  is a solo-developer tradeoff, not a team-safe one.
- **Cost — one wallet, one ledger.** The spend discipline assumes one person can see the
  bill and judge whether a session paid for itself. A shared key across a team, or spend
  routed through procurement with no per-session visibility, removes the signal the whole
  cost-hygiene chapter feeds on.
- **Merge — direct-to-trunk, not pull-request-gated.** The worktree + merge-lock flow *is*
  the review ceremony; it merges branches straight to the trunk. If your shop *requires*
  pull requests — enforced branch protection, required reviewers — the merge mechanics
  don't apply as written; that's a structural mismatch, not a tuning knob. If PRs are your
  habit but not enforced, it's an adaptation: keep the human-as-lead as the reviewer and
  add a PR step to the close gate.

These limits also show up further down, at the mechanism level — the merge-lock chapter
notes its single-machine assumption, the claim-liveness section notes its single-
filesystem one. This is the aggregated version: the place to decide go/no-go before you
descend.

One framing note, because the ground is moving: this is built for **local-machine-rooted**
agents, and cloud-run agents are fast becoming the norm. Read the cloud cases above not as
"fringe, ignore them" but as "the local-rooted mechanisms here will need cloud-native
equivalents, and those aren't in this corpus yet." Worktree isolation, the merge-lock, and
the liveness probe are the three that will need them first.

## The big ideas

Seven ideas carry the whole corpus. Everything in the individual chapters is one of
these made concrete. Read these seven and you have the shape; descend into a chapter
when you need the mechanism.

### 1. Judgement stays human; typing goes to agents

The division of labor everything else serves. Agents generate, search, refactor, and
draft at volume; the human holds the parts that don't delegate well — taste, the call
on whether something is right, and the decision to ship. The whole system is arranged
so the human spends attention on judgement and almost none on mechanics, and so an
agent is never the last word on whether work is correct. Most of the other six ideas
exist to protect this one: to keep the human out of the plumbing, and to put a check
between an agent's confidence and the trunk.
→ realized across the [multi-agent coordination](multi-agent-coordination-doctrine.md)
chapter (the always-loaded agent brief, model-tier routing), the
[debugging discipline](debugging-discipline-doctrine.md) (an agent's guessed fix is
never the last word on the cause), and the review idea below.

### 2. Concurrency without collision

Many agent sessions run at the same time against one repository, each in its own
working copy. The hazard is shared state: two sessions claiming the same ticket, or
editing the same files and colliding when they merge. The defense is to make claiming a
ticket something exactly one session can win, give each session its own isolated copy
of the repo, and funnel the one step that genuinely must happen one-at-a-time — the
merge back to the shared branch — through a single lock. No server, no database, no
queue; just git refs, worktrees, and a lock file.
→ realized in [the backlog system](backlog-system-doctrine.md) (claims as
exclusive-create refs) and [the worktree workflow](worktree-git-workflow-doctrine.md)
(isolation, the merge-lock, liveness).

### 3. One front door in, one gate out

Every unit of agent work — a **session** — enters through a single router that
classifies what kind of work it is, and exits through a single close ceremony that runs
the validation gate before anything merges. Bounding work this way means the heavy
ceremony attaches only to the work that needs it, and nothing reaches the shared trunk
without passing the same checks. The entry router and the exit gate are the two places
the whole lifecycle is enforced.
→ realized in [the session lifecycle](session-lifecycle-doctrine.md).

### 4. Durable, re-derivable handoff

An agent session has no memory of its own past, and its context window is volatile — it
can vanish mid-task. So state that needs to survive a session lives in the
**repository**: committed work, a handoff note on the work item, the backlog's own
records — never only in the conversation. A fresh session, started cold, can read the
durable artifacts and pick the work up where it was left. This is what makes
multi-session work and pause/resume safe rather than a gamble on a window staying alive.
→ realized in [the session lifecycle](session-lifecycle-doctrine.md) (pause/resume, the
handoff block) and [the backlog system](backlog-system-doctrine.md).

### 5. Mechanical enforcement over habit

Any discipline that lives only as good intention gets skipped — under pressure, by a
newcomer, or by an agent that never absorbed the culture — and the skip is silent until
it costs something. So each discipline is converted into a structure the build itself
checks: a pre-commit gate, a layer-boundary lint, a coverage floor, a never-wipe
migration contract, a self-healing hook. Violating the rule becomes a visible failure
instead of an invisible omission.
→ realized in [engineering discipline](engineering-discipline-doctrine.md),
[working with the harness](claude-code-harness-doctrine.md) (the hooks that keep the gate
alive), the [debugging discipline](debugging-discipline-doctrine.md) (the four phases and
the three-fix stop as procedure over reflex), and the
[rule-authoring doctrine](rule-authoring-doctrine.md) (evidence a rule's failure, match its
form, and harden it only where it's skipped).

### 6. Measured, attributed cost

A fleet of agent sessions spends real money, and that spend evaporates from view the
moment a session ends. Doctrine you can't measure is unfalsifiable — you can't tell a
discipline that pays for itself from ceremony that doesn't. So every session leaves a
cost trace attributed to its ticket and branch through one shared capture point, and the
standing context every session re-reads is kept on a budget so it can't grow unnoticed.
→ realized in [cost hygiene](cost-hygiene-doctrine.md).

### 7. Defend the experience in review

A correctness review asks whether the work is well-built. It can't ask the orthogonal
question — whether the person on the receiving end would actually care, or quit. So the
review stage adds read-only **persona reviewers**, vivid and narrow stand-ins for real
user archetypes, beside the craft reviewers, along with explicit defenses against
AI-generated work converging on a generic, average-of-everything look. Convergence is
countered by an adversarial seat — a reviewer whose job is to disprove the work, not
to concur with it.
→ realized in [the persona-reviewer guide](stakeholder-persona-guide.md) and
[template](stakeholder-persona-template.md), and in
[frontend anti-slop](frontend-anti-slop-doctrine.md) and
[prose anti-slop](prose-anti-slop-doctrine.md) — whose recipe-shaped form the
[rule-authoring doctrine](rule-authoring-doctrine.md) explains (Part 2).

## The map

Each chapter realizes one or more of the big ideas above:

| Chapter | Realizes | One-line |
|---|---|---|
| [session-lifecycle](session-lifecycle-doctrine.md) | [3](#3-one-front-door-in-one-gate-out) · [4](#4-durable-re-derivable-handoff) | The session spine: front door in, gate out, pause/resume |
| [backlog-system](backlog-system-doctrine.md) | [2](#2-concurrency-without-collision) · [4](#4-durable-re-derivable-handoff) | File-per-item backlog; claims as git refs; the stage machine |
| [worktree-git-workflow](worktree-git-workflow-doctrine.md) | [2](#2-concurrency-without-collision) | Parallel worktrees on one repo without collision |
| [multi-agent-coordination](multi-agent-coordination-doctrine.md) | [1](#1-judgement-stays-human-typing-goes-to-agents) · [7](#7-defend-the-experience-in-review) | When and how to spawn a team; integrate without corruption |
| [engineering-discipline](engineering-discipline-doctrine.md) | [5](#5-mechanical-enforcement-over-habit) | Turn each habit into a gate, floor, boundary, or census |
| [debugging-discipline](debugging-discipline-doctrine.md) | [1](#1-judgement-stays-human-typing-goes-to-agents) · [5](#5-mechanical-enforcement-over-habit) | Find the cause before you change anything; three failed fixes means stop |
| [rule-authoring](rule-authoring-doctrine.md) | [5](#5-mechanical-enforcement-over-habit) · [7](#7-defend-the-experience-in-review) | Author rules that hold: evidence the failure, match the form, harden against the excuse |
| [cost-hygiene](cost-hygiene-doctrine.md) | [6](#6-measured-attributed-cost) | Measure and attribute spend; budget the standing context |
| [claude-code-harness](claude-code-harness-doctrine.md) | [1](#1-judgement-stays-human-typing-goes-to-agents) · [5](#5-mechanical-enforcement-over-habit) | Make a permission-aware harness pull its weight |
| [frontend-anti-slop](frontend-anti-slop-doctrine.md) | [7](#7-defend-the-experience-in-review) | Break the model's default look; look before you change |
| [prose-anti-slop](prose-anti-slop-doctrine.md) | [7](#7-defend-the-experience-in-review) | Name the failure modes of generated text; read before you write |
| [stakeholder-persona-guide](stakeholder-persona-guide.md) | [7](#7-defend-the-experience-in-review) | Build and deploy a roster of persona reviewers |
| [stakeholder-persona-template](stakeholder-persona-template.md) | [7](#7-defend-the-experience-in-review) | The authoring scaffold for one persona |

**The one reading-path that matters.** If you read three chapters in order, read
[session-lifecycle](session-lifecycle-doctrine.md) (the spine), then
[backlog-system](backlog-system-doctrine.md) (what the spine operates on), then
[worktree-git-workflow](worktree-git-workflow-doctrine.md) (how the concurrency
underneath stays safe). Those three are the operating core; the rest are independent and
read in any order, when their problem becomes yours.

**Prefer to see it run before you read it named?** This page defines each part on its own.
Its companion on-ramp, [**A day in the life**](A-DAY-IN-THE-LIFE.md), follows one work
item from first filing to merge, retold at three depths — the same seven ideas shown
catching real falls, in the order a working afternoon meets them. Read either first; they're built to be
read together.

## Glossary

The shared vocabulary you'll meet across the chapters. Each term is defined fully where
it's load-bearing; this is the quick reference.

- **Session** — one bounded unit of agent work, from entry-router to close-gate. The
  atom the lifecycle operates on.
- **Slot** (`«slot»`) — a fill-in-the-blank marker for anything project-specific. Lift
  the doctrine around it verbatim; fill the slot with your own value.
- **The gate** — the validate step run before every commit and at session close: lint +
  typecheck + test. Referenced everywhere; defined in the engineering chapter.
- **Claim** — a record that exactly one session owns a work item, implemented as an
  exclusive-create git ref so two sessions can't both win it.
- **Worktree** — an isolated working copy of the repo sharing one `.git` database; one
  per concurrent session.
- **Trunk / base branch** (`«BASE_BRANCH»`) — the shared branch sessions sync from and
  merge back into (e.g. `main`).
- **Merge-lock** — the single-holder lock that serializes the merge-back-to-trunk step,
  the one moment concurrency must be funnelled to one writer.
- **Liveness** — the probe that tells a dead session's abandoned claim from a live
  session's held one, so stale work can be reclaimed safely.
- **Ready-check** — the ordered pre-work check a session passes before touching code.
- **Release-on-pause** — ending a session mid-work by dropping the claim but keeping the
  branch, so a fresh session can adopt and continue from durable state.
- **Persona reviewer** — a read-only agent that reacts to an artifact as a specific user
  archetype would, beside the craft reviewers.

---

New to the corpus? You're in the right place — descend into a chapter from the map above
when its problem is yours. Adapting it to your project? Every slot is catalogued in
[`PARAMETERIZATION.md`](PARAMETERIZATION.md); maintenance and re-publish steps are in
[`MAINTAIN.md`](MAINTAIN.md).
