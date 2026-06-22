# Solo-with-agents engineering doctrine

A portable corpus of doctrine for running a software project where **one person
directs a team of AI coding agents** across many concurrent worktrees, under a
permission-aware CLI harness. It is the distilled, project-agnostic form of a
working playbook — the *mechanisms* and the *why*, with every project-specific
noun pulled out into a fill-in **`«slot»`**.

It is **prose to read and adapt**, not a package to install. There is no runnable
tool here. Where the doctrine describes a CLI (`bl`, the backlog tool) or a script,
it documents the shape and contract so you can build or port your own — it does not
ship the implementation. (See *What's not here* below.)

> **New here?** Start with **[`OVERVIEW.md`](OVERVIEW.md)** — a plain-language pitch, the
> seven big ideas the whole corpus rests on, and a map of which chapter realizes which
> idea. This README is the navigation desk; OVERVIEW is the on-ramp above it.

## Who this is for

A team lead — human or agent — standing up the operating discipline for a
repo where:

- AI agents do most of the typing, and the human holds judgement, taste, and ship.
- Multiple sessions/worktrees run **concurrently** against one repo and must not
  collide on a shared index or double-claim work.
- Work spans multiple sessions and needs durable, re-derivable handoff.
- A permission-gated harness (Claude Code or similar) mediates every action.

If you run a solo, single-stream repo with no agents, most of this is more
machinery than you need — but the engineering-discipline, cost-hygiene, and
frontend-anti-slop files still pay rent.

**Where this stops fitting.** The concurrency, claim, and merge machinery assumes a
single developer on a local machine, one shared git checkout, git worktrees, and
direct-to-trunk merges. Teams, cloud agents without a shared filesystem, a non-git VCS,
or a PR-required merge model each break one or more of those assumptions — sometimes as a
clean adaptation, sometimes as a structural mismatch. **[`OVERVIEW.md`](OVERVIEW.md#when-this-fits-and-when-it-doesnt)**
walks each axis with what transfers and what doesn't; read it before you adopt the
concurrency or merge mechanics.

## How to navigate

If you're new, read **[`OVERVIEW.md`](OVERVIEW.md)** first — it carries the pitch, the
seven big ideas, and a map. From there, the spine is **session-lifecycle** (one front
door in, one gate out) and **backlog-system** (what the lifecycle operates on); the rest
are independent and read in any order.

| File | What it covers | Read it when |
|---|---|---|
| [`OVERVIEW.md`](OVERVIEW.md) | The on-ramp: plain-language pitch, the seven big ideas, the map of chapter → idea, the one reading-path, a glossary | You're new to the corpus, or want the shape before the mechanisms |
| [`session-lifecycle-doctrine.md`](session-lifecycle-doctrine.md) | The session spine: a multi-flow entry router, the ordered ready-check, the close-out gate, multi-session pause/resume | You're designing how a session starts, hands off, and ends |
| [`backlog-system-doctrine.md`](backlog-system-doctrine.md) | File-per-work-item backlog: state in frontmatter, claims as exclusive-create git refs, the stage machine + decomposition gate, typed parent items | You need a concurrent-safe backlog without a server or DB |
| [`worktree-git-workflow-doctrine.md`](worktree-git-workflow-doctrine.md) | Parallel worktrees on one repo: claim liveness, merge discipline, the merge-lock, branch identity, trunk-leak detection | Multiple worktrees share one `.git` and you must keep them from colliding |
| [`multi-agent-coordination-doctrine.md`](multi-agent-coordination-doctrine.md) | When to spawn agents, model-tier routing, the team vs. parallel-spawn choice, three-phase integration, worktree isolation for committing agents | You're dispatching a team of subagents and need them to coordinate, not corrupt each other's branches |
| [`engineering-discipline-doctrine.md`](engineering-discipline-doctrine.md) | The per-commit gate, layer-boundary enforcement, coverage floors, the never-wipe save-migration contract, the no-singleton (swap) contract, the warn→error lint ratchet | You're setting the invariants the codebase enforces mechanically rather than by habit |
| [`cost-hygiene-doctrine.md`](cost-hygiene-doctrine.md) | Measuring per-session/per-ticket cost, the single shared capture point, prompt-cache preservation, effort tiers, context-management commands | Spend is unexplained and you want it measured and attributed, not guessed |
| [`claude-code-harness-doctrine.md`](claude-code-harness-doctrine.md) | Working *with* a permission-aware harness: allowlist-friendly commands, self-healing hooks, the run-until-verifiable loop, tool-to-job matching (structured editor / batch substitute / symbol nav) | You're tuning how agents act under a gated CLI and its hooks |
| [`frontend-anti-slop-doctrine.md`](frontend-anti-slop-doctrine.md) | Look-before-you-change UI screenshots, the model's default house style and how to break it, the two-surface font policy | An agent is producing player-/user-facing UI and you want it to not look generated |
| [`stakeholder-persona-guide.md`](stakeholder-persona-guide.md) | How to build and deploy a roster of read-only persona reviewers beside your craft reviewers | You want engagement/UX defended in review, not just correctness |
| [`stakeholder-persona-template.md`](stakeholder-persona-template.md) | The authoring template for a single persona reviewer | You're writing one persona and need the slot-by-slot scaffold |
| [`PARAMETERIZATION.md`](PARAMETERIZATION.md) | The consolidated catalogue of every `«slot»` across the corpus, grouped by concern | You're adapting the doctrine and need every fill-in-the-blank in one place |

## The `«slot»` convention

Everything project-specific in this corpus is written as a **`«slot»`** —
`«BASE_BRANCH»`, `«linter»`, `«PREFIX»`, and so on. The doctrine around a slot is
meant to be lifted **verbatim**; you fill the slot with your project's concrete
value. Each file ends with its own *parameterization* / *adapt this* section listing
the slots it introduces and where each binds in a reference implementation.

A few slots recur as plain prose markers rather than config: `«slot»` itself (the
generic "a project-specific noun goes here" marker) and `«gate»` (your validate
step — lint + typecheck + test; defined in the engineering doctrine and referenced
everywhere). Fill `«gate»` once, mentally, and reuse it across all files.

The full catalogue of every slot — grouped by concern, with what to fill each with and
which chapter defines it — lives in **[`PARAMETERIZATION.md`](PARAMETERIZATION.md)**. Reach
for it when you're porting a mechanism into your own project; each chapter also carries its
own *adapt-this* section for the slots it introduces.

## What's not here

This is the **doctrine**, not the implementation. Deliberately excluded:

- **The runnable backlog tool (`bl`) and its scripts.** They are tightly coupled to
  the source project (its id prefixes, paths, and harness). The `backlog-system` and
  `worktree-git-workflow` files document the tool's *contract* and structure so you
  can build or port one; the code itself is not the export target. If you want to
  adapt a concrete implementation, see the source repo (below).
- **Project-specific content** — game/domain nouns, design specifics, and any one
  product's surfaces. Where a worked example was unavoidable it's generic.

Model-specific (not project-specific) guidance — e.g. a current model's default
house style in the frontend file — is intentionally **kept**, because it transfers
to anyone using that model.

## Maintenance & provenance

This corpus is a **snapshot**, refreshed on demand before each publish — not a
continuously-synced mirror of its source rules. See [`MAINTAIN.md`](MAINTAIN.md)
for the refresh-before-publish checklist.

The source of truth is the `doctrine/` directory in the project this was extracted
from. Updates flow from there; this published copy is downstream. If you find a gap
or a slot that doesn't generalize, that's expected drift — fold it back at the
source and re-snapshot.
