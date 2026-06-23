# Parallel-worktree git workflow

> **In one line.** Let many worktrees on one repo sync from and merge back to a shared
> branch without corrupting each other — using only git, a few scripts, and one lock file.

**TL;DR — the mechanisms**
- Each session gets its own worktree; sync from trunk always lands as a merge commit.
- One merge-lock serializes the close window so two merges never interleave.
- A liveness probe distinguishes a dead session's claim from a live one's.
- A reaper reclaims dead worktrees without losing their committed work.
- An auditable bypass seam lets automated merges skip the per-commit gate.

**Read this when** multiple worktrees share one `.git` and must not collide.
**Big ideas:** [concurrency without collision](OVERVIEW.md#2-concurrency-without-collision)
**Depends on:** [backlog system](backlog-system-doctrine.md) (claims it keeps live)

---

How to let **many concurrent worktrees on one repo** sync from a shared base branch
and merge back into it without corrupting each other — using nothing but git, a few
small shell scripts, and one lock file. No server, no CI gate, no merge queue.

This is project-agnostic doctrine extracted from a working implementation. Wherever a
concrete noun appears (a branch name, a directory, an env-var name), it is called out
as a **`«slot»`** you fill for your own project. The parameterization table near the
end collects every slot in one place.

It is the operational companion to [a file-per-work-item backlog
system](./backlog-system-doctrine.md). That doctrine owns **who is working on what**
(the git-ref *claim model*, the cross-branch *index*, the eligibility *picker*). This
doctrine owns **how their branches converge**: the merge ceremony, the serialization
lock, and the reaper that cleans up after them. Where this doc needs the claim model
— and it does, in two places — it **points at** the claim-model section of that
companion rather than re-deriving it.

---

## Why this shape

The setup is *N* worktrees checked out from one repo, each on its own branch, each a
separate session (a person, or an agent) doing independent work. They share one git
object store and one filesystem. They periodically pull the base branch in to stay
current, and eventually each merges its branch back. Three things go wrong when more
than one of them is active at once:

1. **Divergence compounds silently.** A branch that never pulls the base branch drifts
   until its eventual merge-back is a conflict minefield. *Fix: a one-command sync
   that any session runs on cadence*, cheap enough that "sync first" is the reflex —
   so divergence stays bounded and conflicts surface early, a few files at a time.

2. **Two sessions merge into the base at the same instant.** Both ran their tests
   green on their own branch; both fast-forward-resolve the base into themselves;
   both push. The second push either rejects (and the session re-races) or, worse,
   they collide on a shared append-only structure — a migration index, a changelog,
   a counter — that was non-conflicting until both landed. *Fix: serialize the
   merge-back window* so only one session is inside it at a time.

3. **Crashed sessions leave litter.** A worktree whose session died holds a branch, a
   lock, maybe a running dev server. Nothing reclaims it. The next session trips over
   a stale lock or hands out work the dead session was holding. *Fix: a reaper that
   distinguishes a live session from a dead one* and cleans only the dead — without
   ever discarding unmerged work.

The four scripts below are the smallest set that solves all three while keeping every
operation a plain, inspectable shell command a human can run by hand.

> **A prerequisite the scripts assume: branch identity is keyed on the immutable item
> id, not on anything that can change.** If a session derives its branch name from a
> work item's *title* (a readable slug), a mid-flight re-title forks branch identity:
> the same item now has two differently-named branches, and no tool sees them as "the
> same item's branch." With two sessions on one item that compounds into duplicated
> effort, a claim stolen out from under the first session, and a merge that clobbers
> its work. Anchor the branch name on the **id** (stable for the item's whole life) —
> the title can ride along for readability (`{prefix}-{id}-{title-slug}`), but the id
> is the part tools key on, so a re-title still resolves to the same item. And have any
> branch-rename step **refuse** when a differently-slugged branch already carries the
> same id, directing the operator to the adopt verb to land on that existing branch
> rather than minting a second one — you cannot `git branch -m` onto a ref that may be
> checked out in another worktree, so reuse is a deliberate adopt, not an automatic
> rename. The convergence machinery below is correct only if each item maps to exactly
> one branch.

---

## The four moving parts

| Script | Verb | What it answers |
|---|---|---|
| **sync** (`«sync»`) | merge the base branch *in* | "Am I current with the base branch?" |
| **finalize** (`«finalize»`) | finish an in-progress merge | "Conflicts resolved — commit the merge." |
| **merge-lock** (`«merge-lock»`) | serialize the merge-back window | "Is anyone else merging into the base right now?" |
| **reaper** (`«reaper»`) | clean dead worktrees / locks / branches | "What can be safely reclaimed?" |

`sync` and `finalize` are a pair: one starts a merge, the other resumes it after
manual conflict resolution. `merge-lock` wraps the close window so two sessions never
merge back at once. `reaper` runs occasionally (or on session start) to garbage-collect
what crashed sessions left behind.

---

## Sync: merge the base branch in, always with a merge commit

The sync script merges the base branch (`«BASE_BRANCH»`, e.g. `main`) into the
current branch. Two properties are load-bearing:

**Always `--no-ff`.** Even when a fast-forward is possible, force a real merge commit.
Each sync then becomes a **discrete, addressable commit** — which is what makes the
merge-back serialization model work: every base-branch integration is a node you can
point at or revert, never an invisible fast-forward folded into history. A project that
doesn't serialize merges can drop this; a project that does cannot.

**Two-phase, so conflicts have a clean resume point.** The merge runs
`--no-commit --no-ff`. If git resolves everything automatically, the script calls
`finalize` itself and exits. If conflicts arise, it stops *inside* the merge and prints
the three-step recovery:

```
1. Fix the conflicts in the listed files.
2. git add <resolved-files>
3. run «finalize»
```

The script refuses to start if a merge is already in progress (a stray `MERGE_HEAD`),
and exits 0 with "already up to date" when there is nothing to merge. Both are guards
against compounding a half-finished state.

> **The reflex this enables:** "sync before you read state, and sync again before you
> merge back." Because the command is one line and bounded-cost, a long-lived branch
> runs it at every natural boundary. Divergence never accumulates to the point where
> the merge-back surfaces dozens of conflicts at once.

### Sync must re-assert branch-local state the merge can clobber

A subtle interaction with the companion backlog system: the item's flip to
**in-progress lives only on the feature branch** (the trunk deliberately never
carries it — see that doctrine's *trunk-leak caveat*). So if the item was re-groomed
on the trunk while you worked — its stage edited back, or any sibling's merge carried
a stale copy onto the trunk — then **merging the trunk *in* can overwrite your
branch-local in-progress flip with the trunk's copy.** The symptom is nasty:
"what's my active item?" now reports *nothing*, even though your claim ref is still
live, and a manual stage restore is needed before you can ship.

The fix is to make `sync` **idempotently re-assert branch-local state after the
merge**: once the merge commit lands, for any item with a live claim on this branch,
re-write its stage to in-progress if the merge knocked it off. This is a distinct,
*work-branch* direction — not to be confused with a trunk-only normalizer that pushes
the opposite way (reverting a leaked in-progress on the trunk back to ready). Same
field, opposite directions, different code paths; don't try to reuse one for the
other.

## Finalize: the resume point after conflict resolution

`finalize` exists so the conflict path and the clean path end the same way. It is the
second half of `sync`, callable on its own after a human resolves conflicts. It:

- **Refuses if you are not in a merge** (no `MERGE_HEAD`) — you cannot "finalize"
  nothing, and the error points back at `sync`.
- **Refuses if unresolved conflicts remain** — staged-but-conflicted entries (`git
  ls-files -u`) block the commit, so a half-resolved tree cannot be committed by
  accident.
- **Commits with `--no-edit`** and the hook-bypass sentinel set (see *The bypass
  seam* below).

Splitting finalize out of sync is what lets the automatic and manual paths share one
commit step — the clean merge and the hand-resolved merge produce identical commits.

---

## The merge lock: serialize the close window

When a session is ready to merge back, it enters a critical window: *sync the base in
→ run the validation gate → merge the branch into the base → push*. If two sessions
are in that window simultaneously, they race. The merge lock makes the window
single-occupant across every worktree on the machine. This is a **single-machine,
single-filesystem** guarantee: the lock is a shared file plus an OS `flock`, neither of
which is reliable across NFS or separate hosts. Worktrees spread across machines need a
different primitive (a real lock service); this doctrine assumes the common case of many
worktrees on one disk.

### Keyed on the worktree, not the branch — and why

The lock file (`«LOCK_DIR»/.merge-lock`, a small JSON blob `{branch, worktree,
acquired_at}`) lives at the **base repo root** — the one filesystem location every
worktree shares, resolved by anchoring `git rev-parse --git-common-dir` at the *script's
own* directory (not the caller's `cwd`) and taking its parent, so the script finds the
right base repo no matter where it's invoked from. `«LOCK_DIR»` is that shared root, not
a dedicated lock directory; both `.merge-lock` and an `flock` sentinel (below) sit in
it. The identity that matters is **the worktree path (`pwd`), not the branch.** The
branch in the lock is a human-readable label only.

> **Dependency:** the lock scripts use `jq` to read and write the JSON blob, and every
> read is `jq … 2>/dev/null || echo ""` — so on a machine *without* `jq` the lock
> degrades silently (every field reads empty), not loudly. Treat `jq` as a hard
> dependency of this layer, not an optional nicety.

This matters because the obvious liveness token — *"is the holder's claim ref still
pointing at HEAD?"* — is **wrong**, and was a real bug. In the companion backlog
system, a claim ref points at the *pre-claim* commit (never HEAD once the session has
committed), and the ship step *deletes* the claim mid-merge. A lock keyed on
"claim ref points at HEAD" therefore deems every holder stale for its entire session,
making the lock decorative exactly when it is needed. The fix is to key liveness on a
**wall-clock staleness timeout** instead.

### Staleness by timeout, failing closed

A lock is stealable only if its holding worktree directory has **vanished**, or its
`acquired_at` is older than `«STALE_TIMEOUT»` (default 60 min — sized generously over
the realistic worst-case merge-back window; the long pole is conflict resolution while
syncing the base in during a close). The timeout is the auto-recovery path for a
genuinely crashed holder.

Two safety properties guard against stealing a *live* lock:

- **Parse failures fail closed.** A missing or unparseable `acquired_at` is treated as
  *live*, never stale. Stealing on a parse error would reintroduce the race the lock
  exists to prevent.
- **Stealing a still-present holder is loud.** If a lock times out but its worktree
  still exists (a *timeout* steal, not a vanished-worktree steal), the steal prints a
  warning naming the holder and its age — because if that session is still merging,
  the steal is a bug, and it must be visible. The clean case (worktree gone) is silent.

### The rest of the surface

- **`acquire [branch]`** retries 3× (1 s apart) before giving up, and reports the
  current holder on failure ("another session is merging — retry when done").
- **`release`** is idempotent and **refuses to release a lock owned by a different
  worktree** — so a script run from the wrong checkout can't unlock someone else's
  merge.
- **`status`** prints the holder; **`prune-stale`** drops a lock that meets the
  staleness test without acquiring.
- A second, OS-level `flock` (on its own `.merge-lock-flock` sentinel beside the lock
  file) wraps the read-modify-write of `.merge-lock` itself, so the check-then-write is
  atomic even under simultaneous `acquire` calls — bounded-retry, not a blocking wait
  (3× then exit). The merge-lock (backlog state's sibling) and this flock (file-write
  atomicity) are **two different locks at two different layers** — don't collapse them.

> **Claim ref vs. merge lock — keep them separate.** The claim ref (companion doctrine)
> is *acquire-time backlog state*: who owns this item across its whole life. The merge
> lock is *close-time operational state*: who is inside the merge window right now.
> Same machine, different lifetimes, different keys. Conflating them is what produced
> the decorative-lock bug above.

---

## The reaper: reclaim dead worktrees without losing work

Crashed and finished sessions leave worktrees, locks, and branches behind. The reaper
sweeps them. It defaults to **dry-run** (print what it *would* do); `--execute` applies.
It never touches the current worktree or the base checkout. Its passes, grouped here by
purpose (the source numbers them 1, 2, 3, 3.5, 3.7, 4, 5 — the half-steps were inserted
later, so don't expect a 1:1 mapping):

1. **Prune stale git worktree refs** (`git worktree prune`).
2. **Remove orphan directories** — dirs under `«WORKTREE_DIR»` that git no longer
   tracks.
3. **Unlock dead-PID locks** — *if* your worktree locks carry a real, mortal PID (see
   the caveat below).
4. **Kill stale dev-server processes** — an **injected process-kill hook**
   (`«process-kill»`), not core logic. Omit it if your sessions don't spawn servers.
5. **Flag stranded worktrees** — unmerged work with no live owner. **Advisory only,
   never auto-removed**: unmerged commits need human intent to discard. *Detection is
   project-injected* — the source delegates to its backlog tool's `check` verb (which
   reads claim refs); a generic reaper supplies its own stranded-detection or omits the
   pass. Like `«process-kill»`, this is a hook, not built-in logic.
6. **Remove stale-merged worktrees** — the main event, gated by the guards below.
7. **Delete merged branches** — branches already contained in the base branch, skipping
   the current branch and any branch an active worktree holds.

### The liveness signal is injectable

Whether a worktree is *alive* is the crux of pass 6, and it is the one piece that is
**not** generic — it is an **injected liveness callback** (`«liveness»`). A worktree is
kept iff:

- the liveness probe reports a **live owner** on its branch, **or**
- the worktree was created within a **`«GRACE_MINUTES»`** window (default 15 min) —
  covering a nascent session that hasn't acquired its claim yet.

In the source implementation the probe maps *branch → in-progress item → claim ref →
owning session*, then checks that session's heartbeat — in that harness, the mtime of
the session's transcript file, a harness-specific stand-in for "is this session still
alive?" Yours will differ; the contract is all that's portable: **`«liveness»(branch) →
live | dead`**, with a grace window for the not-yet-claimed. A dead owner's stale claim
is cleared as a side effect of the probe (so it stops blocking re-adoption) — though
only on `--execute`; dry-run omits the clear — and that clearing never by itself removes
the worktree, which stays behind the data-loss guards.

### The data-loss guards (never bypass these)

A worktree is removed only when **all** hold:

- **No commits ahead of the base branch** (`git rev-list «BASE_BRANCH»..branch` is
  empty) — never discard unmerged work.
- **Clean tree** (`git status --porcelain` empty) — never discard uncommitted work.
- **Does not hold the merge lock** — a long close-session lands its merge into the base
  (emptying `rev-list`) *before* it pushes and releases the lock; without this guard
  the reaper could reap a worktree mid-close. The merge lock is the authoritative
  "I am mid-merge" signal, so honor it directly.
- **Not live and past grace**, per the liveness signal above.

### The daemon-PID caveat (why pass 3 may not work for you)

Pass 3 assumes lock files name a **mortal** PID you can `kill -0` to test liveness.
Some harnesses lock every worktree to a single **immortal supervisor/daemon PID** — in
that environment a PID liveness test cannot distinguish a dead session from a live one,
because the PID is always alive. That is the entire reason pass 6 keys on the injected
liveness callback instead of the lock. **Know which world you are in:** if your locks
carry per-session mortal PIDs, pass 3 is enough and the callback can be a thin PID
check; if they carry an immortal daemon PID, the callback must consult an out-of-band
heartbeat (a claim ref, a session registry, a heartbeat file) and pass 3 is decorative.

---

## The bypass seam: automated merges skip the commit gate

A repo with a local pre-commit / commit-msg gate (lint, typecheck, message-format
checks) faces a conflict: the **merge commits** produced by `finalize` and the
automated close-session flow should *not* re-run that gate. The merged content was
already validated on the branches being merged; re-validating the merge commit is
redundant, and these flows predate (or run beneath) the interactive gate.

The seam is a single env sentinel, **`«BYPASS_ENV»`** (e.g. `BL_HOOK_BYPASS=1`): the
gate hooks early-exit when it is set, and only the automated git invocations set it
(`finalize`'s commit, the `bl` lifecycle commits, the close-session merge-and-push). A
**human's own `git commit` / `git push` never sets it**, so manual work always runs the
full gate. One deliberate carve-out is worth stating: the **push to the production /
release branch never sets it** — that is exactly the gate you want enforced. An env
sentinel is used over TTY detection because tooling commits are also non-interactive;
TTY-detection would wrongly bypass them too.

> CI remains the non-bypassable backstop. The sentinel relaxes only the *local*
> pre-commit hooks for merges that were already validated upstream — it is not a way to
> land unvalidated content.

---

## Parameterization seams

Every project-specific noun, in one place. Most are single constants at a script top;
two are injected callbacks.

| Slot | Source binding | Agnostic form |
|---|---|---|
| `«BASE_BRANCH»` | `main` | the integration branch sessions sync from / merge back to |
| `«LOCK_DIR»` | base repo root (script-anchored `--git-common-dir`'s parent) | shared path every worktree sees; holds both `.merge-lock` and the `.merge-lock-flock` sentinel |
| `«WORKTREE_DIR»` | `.claude/worktrees/` | directory the worktrees live under (reaper passes 2 & 6) |
| `«BYPASS_ENV»` | `BL_HOOK_BYPASS` | env-var name the commit gate honors as "skip me" |
| `«STALE_TIMEOUT»` | `3600` s | merge-lock auto-steal age; size over your worst-case merge-back window |
| `«liveness-timeout»` | `bl`'s heartbeat threshold (~90 min) | *separate* from `«STALE_TIMEOUT»`: how stale a session heartbeat may be before `«liveness»` calls it dead |
| `«GRACE_MINUTES»` | `15` min (measured off the worktree's `.git` mtime) | reaper grace for a not-yet-claimed nascent worktree |
| `«liveness»(branch)` | `bl claim-liveness` (heartbeat probe) | injected callback: is this branch's owner live? |
| `«process-kill»` | `kill-stale-vite.sh` (a Vite dev server) | injected hook: kill a session's stray subprocesses — replace with your server's killer (optional) |
| claim namespace | `refs/claims/«ID»` | already generic — owned by the companion backlog doctrine, unchanged |

Runtime dependencies of this layer: `bash`, `git`, and `jq` (the lock blob is read/written with `jq`).

The claim model itself — how `«liveness»` finds an owner, how claims are acquired with
exclusive-create git refs — is **not redefined here.** See *the claim* in the
[backlog-system doctrine](./backlog-system-doctrine.md); this workflow consumes it
through the `«liveness»` seam and otherwise treats it as a black box.

---

## How the pieces test

The two genuinely concurrent mechanisms — the exclusive-create claim and the merge
lock — are verified with a portable bash technique worth lifting on its own: spin up a
**throwaway git repo in a temp dir**, put **PATH-stubbed fake CLIs** ahead of the real
ones to force a specific interleaving (e.g. two `acquire` calls racing), and assert
exactly one wins. It needs no test framework and runs anywhere `bash`, `git`, and `jq`
do. The race conditions these scripts exist to prevent are the ones a unit test in a
single process cannot reach, so the test harness reproduces the concurrency directly.

---

## When to adopt this

This workflow pays for itself the moment **two or more sessions work one repo in
parallel** and merge into a shared branch. For a single linear worktree it is more
machinery than `git pull --rebase` needs — though `sync`'s always-merge-commit habit
and the reaper's data-loss guards are worth borrowing even solo.

The non-negotiable core, if you adopt nothing else: **sync with a real merge commit so
integrations are addressable**, **serialize the merge-back window on a shared lock
keyed by worktree (not by a claim ref that points at the wrong commit)**, and **gate
every reap behind unmerged-work / dirty-tree / holds-the-lock guards.** Those three are
what make N worktrees safe to run at once; the bypass seam and the dev-server hook are
ergonomics on top.
