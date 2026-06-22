# Parameterization: every slot in one place

> **In one line.** The consolidated catalogue of every `«slot»` in the corpus — what to
> fill each with and which chapter defines it — for when you're porting a mechanism into
> a project of your own.

This is the L3 reference layer: read it when you're adapting the doctrine, not when
you're learning it. New to the corpus? Start at [`OVERVIEW.md`](OVERVIEW.md) instead;
each chapter also carries its own *parameterization / adapt-this* section for the slots
it introduces. This file is the union of all of them.

Every **configuration** slot across the corpus, grouped by concern, with the file that
defines it and a typical concrete value. (The persona *template* also uses inline `«…»`
placeholders, but those are fill-in-the-blank *authoring prompts* documented within that
file — they are not deployment config and are omitted here.)

## The `«slot»` convention

Everything project-specific in this corpus is written as a **`«slot»`** —
`«BASE_BRANCH»`, `«linter»`, `«PREFIX»`, and so on. The doctrine around a slot is meant
to be lifted **verbatim**; you fill the slot with your project's concrete value. Each
chapter ends with its own *parameterization* / *adapt this* section listing the slots it
introduces and where each binds in a reference implementation.

A few slots recur as plain prose markers rather than config: `«slot»` itself (the
generic "a project-specific noun goes here" marker) and `«gate»` (your validate step —
lint + typecheck + test; defined in the engineering doctrine and referenced everywhere).
Fill `«gate»` once, mentally, and reuse it across all files.

## Cross-cutting

| Slot | What you fill it with | Defined in |
|---|---|---|
| `«slot»` | Generic marker: any project-specific noun | all files |
| `«gate»` | Your validate step (lint + typecheck + test), run before every commit | engineering · referenced everywhere |
| `«BASE_BRANCH»` | The trunk branch to merge from / detect leaks against (e.g. `main`) | backlog · worktree · session |
| `«ci»` | Your CI gate / workflow that backstops the local gate | engineering · harness |
| `«vcs»` | Your version-control system (this corpus assumes git) | harness |

## Backlog & lifecycle

| Slot | What you fill it with | Defined in |
|---|---|---|
| `«PREFIX»` | Your id-prefix registry (features / infra / content / …) | backlog |
| `«ID»` | A work-item id (`«PREFIX»-42`) — also the filename stem | backlog |
| `«BACKLOG_DIR»` / `«SHIPPED_DIR»` | Active-items dir / terminal-archive dir | backlog |
| `«SPEC_SECTION»` / `«INTENT_SECTION»` | Section names the promotion gate fires against | backlog |
| `«RESUME_SECTION»` | The multi-session handoff-block heading | backlog · session |
| `«CAP»` | Priority ceiling (e.g. 99) | backlog |
| `«claim-next»` / `«branch-name»` / `«sync»` / `«ready-check»` / `«pause»` | Your concrete session verbs (pick-and-claim, branch derivation, merge-from-trunk, the ordered pre-work check, release-on-pause) | session |
| `«router flows»` | Your session-entry classification set (resume / start / groom / ad-hoc) | session |
| `«handoff note»` | Your durable cross-session handoff artifact | session |

## Worktrees, claims & liveness

| Slot | What you fill it with | Defined in |
|---|---|---|
| `«LOCK_DIR»` / `«WORKTREE_DIR»` | Where worktrees and their locks live | worktree |
| `«merge-lock»` / `«finalize»` / `«reaper»` | Your merge-serialization lock, finalize-merge helper, stale-worktree reaper | worktree · session |
| `«liveness»` / `«liveness-timeout»` / `«STALE_TIMEOUT»` / `«GRACE_MINUTES»` | The claim-liveness probe and its timeouts — **harness-coupled**, supply your own heartbeat source | worktree · backlog |
| `«process-kill»` | How you hard-terminate a dead session/worktree | worktree |
| `«SESSION_UUID»` | The per-session id var your harness exports (claim stamp + liveness) | backlog · worktree |
| `«BYPASS_ENV»` / `«hook-bypass»` | The env sentinel that lets automated commits skip local hooks | worktree · engineering · harness |

## Engineering gate & invariants

| Slot | What you fill it with | Defined in |
|---|---|---|
| `«linter»` / `«typechecker»` / `«test-runner»` | Your three gate tools | engineering |
| `«baseline-check»` / `«tracker»` | Pre-work clean-tree check; your work-item tracker | engineering |
| `«import-boundary-lint»` | The lint rule enforcing layer boundaries | engineering |
| `«disable-directive»` | Your lint inline-disable syntax (use sparingly, with reason) | engineering · harness |
| `«coverage-tool»` / `«coverage-meta»` / `«coverage-reporter»` | Coverage instrument + the test-attributed coverage-census mechanism | engineering |
| `«migration-list»` / `«version-field»` / `«backfill-check»` | Your save-migration array, version field, and the backfill-ban check | engineering |
| `«namespace»` / `«storage-interface»` / `«swap-risk-marker»` | Per-tenant key namespace · the swappable storage adapter contract · the comment marker flagging swap-breaking decisions | engineering |
| `«type-prefix»` | Your commit-message type prefixes (feat / fix / refactor / …) | harness · engineering |

## Harness, hooks & cost

| Slot | What you fill it with | Defined in |
|---|---|---|
| `«tool»` / `«structured-editor»` / `«batch-substitute»` / `«symbol-nav»` | A harness tool · safe string-replace editor · raw bulk-substitute tool · language-server nav capability | harness |
| `«hooks-dir»` / `«hookspath-config»` / `«install-vector»` | Where hooks live · the config pointing at them · how they self-install | harness |
| `«stop-hook»` / `«session-end-hook»` / `«budget-check-hook»` / `«catalog-check-hook»` | Your harness lifecycle hooks | harness · cost |
| `«goal-source»` / `«protected-paths»` / `«protected-files»` | Standing-goal source · paths agents must not touch · high-blast-radius lead-only files | harness · multi-agent |
| `«settings»` / `«telemetry-log»` / `«usage-insight»` | Harness settings file · the cost log · the usage-reporting tool | harness · cost |
| `«capture-script»` / `«cost-tool»` / `«session-ticket»` / `«captured-via»` | The single cost-capture point · cost CLI · ticket-attribution var · capture-source tag | cost |
| `«compact»` / `«clear»` / `«rewind»` | Your context-management commands | cost |

## Multi-agent coordination

| Slot | What you fill it with | Defined in |
|---|---|---|
| `«constitution»` | Your always-loaded agent instructions (the project brief every agent inherits) | multi-agent |
| `«team menu»` / `«team-cap»` | Your roster of named specialist + persona agents · the per-session agent cap | multi-agent |
| `«layer-rules»` / `«isolation»` | The layer import boundaries agents respect · the worktree-isolation flag for committing agents | multi-agent |
| `«path»` | The worktree path embedded in a spawn prompt | multi-agent |

## Frontend anti-slop

| Slot | What you fill it with | Defined in |
|---|---|---|
| `«player-facing»` | What counts as a user-facing surface in your product | frontend |
| `«run-the-app»` / `«fixture-bridge»` / `«viewport-set»` / `«theme-set»` | How you launch the app · seed fixture state · the viewport + theme matrix to capture | frontend |
| `«screenshot»` / `«snapshot»` / `«tmp»` | Screenshot capture · a11y-tree snapshot · the ephemeral scratch dir | frontend |
| `«house-default»` / `«model-tell-block»` / `«its-default-palette»` | Your model's default house style · the warning block naming it · its give-away palette | frontend |
| `«token-source»` / `«display-font»` / `«body-font-chain»` | Your design-token source of truth · display typeface · body/data font fallback chain | frontend |
| `«rubric»` / `«verdict-subagent»` / `«cycle-cap»` / `«stale-transform-check»` | The review rubric · the throwaway verdict agent · the edit-cycle cap · the served-transform freshness check | frontend |

## Persona reviewers

| Slot | What you fill it with | Defined in |
|---|---|---|
| `«archetype»` / `«archetype-slug»` | A user archetype and its slug | persona guide · template |
| `«your harness's shared agent-foundation file»` | The file every spawned agent inherits | persona template |
