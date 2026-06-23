# Maintaining this corpus

This doctrine is a **snapshot**, not a live mirror. It was extracted from a working
project's operating rules at a point in time, genericized, and published for others
to read. It does **not** track every change to those source rules — a
per-source-change sync obligation would be overkill for a secondary shareable.

Instead it is **refreshed on demand, right before each publish**. The procedure
below *is* that refresh. Run it as a checklist whenever you re-publish; skip it
otherwise.

## Why on-demand, not continuous

The corpus's value is the distilled doctrine and the *why* behind each mechanism —
both of which are stable. The source rules churn at the speed of one project's daily
work; most of that churn is project-specific detail that never reaches the
generalized form. Mirroring it continuously would cost constant effort to keep a
secondary artifact in lockstep with a primary one almost nobody reads in real time.
On-demand refresh spends that effort once, when it actually matters: at publish.

Revisit this stance only if the corpus gains an external audience that expects
continuous freshness. Until then, snapshot-before-publish is the right ceremony for
the stakes.

## Transport: vendoring via a clean tree-export

This corpus can be **read** as a published snapshot, or **vendored** directly into a
consuming repository as a real subdirectory of files (not a submodule pointer), so
every clone and worktree of the consumer gets the corpus with zero extra lifecycle
steps. When vendored, the publish/refresh ceremony below moves that subdirectory's
tree to and from the corpus remote with git plumbing — `commit-tree` and `read-tree`.

Let `<vendor-dir>` be the subdirectory the corpus lives in, `<corpus-remote>` the
corpus repository (a git remote name or URL), and `<branch>` its publish branch.
Fetch the remote first (`git fetch <corpus-remote> <branch>`) so the operations
below parent/replace against its current tip.

- **Publish improvements up (push up):** create exactly one commit whose tree *is*
  the local vendor tree, parented on the remote tip, and push it:
  ```bash
  NEW=$(git commit-tree "$(git rev-parse HEAD:<vendor-dir>)" \
          -p <corpus-remote>/<branch> -m "doctrine: refresh corpus snapshot")
  git push <corpus-remote> "$NEW:refs/heads/<branch>"
  ```
- **Refresh a vendored copy (pull down):** replace the local subdirectory wholesale
  with the remote tree, in one commit:
  ```bash
  git rm -r --cached <vendor-dir>
  rm -rf <vendor-dir>
  git read-tree --prefix=<vendor-dir>/ -u <corpus-remote>/<branch>
  git commit -m "docs: pull doctrine corpus refresh"
  ```

Both directions are last-writer-wins snapshot semantics — they match the on-demand
philosophy above (a refresh you choose to run, never a continuous mirror) and attempt
no content merge. If the two trees have diverged, sync the *other* direction first.

### Do NOT use `git subtree push`

`git subtree push` is **banned** for this corpus — it is structurally wrong here, not
merely risky:

- **It replays full ancestry.** `git subtree push` runs `git subtree split`, which
  rewalks the consumer's *entire* commit history. The original `git subtree add` records
  a merge (`Merge commit '…' as '<vendor-dir>'`) that defeats a clean cut, so the split
  replays the consumer's mainline into the public corpus repo — thousands of unrelated
  commits, authored by the consumer's contributors, polluting a corpus that should carry
  one snapshot commit per refresh. (This corpus was cleaned of exactly such a pollution:
  7,756 commits collapsed back to one.)
- **`--squash` does nothing on push.** `--squash` only affects merges *into* a repo on
  `add`/`pull`. On push it is silently ignored — it cannot prevent the replay above.

The plumbing recipe above sidesteps both: `commit-tree` takes the tree object directly,
so there is no ancestry to walk and exactly one commit lands per sync.

### Watch-item that still applies

- **Never use a forge's "Squash and merge" button on a PR that touches `<vendor-dir>`.**
  A line-ending normalization in some forges (the 2024 GitHub CRLF case is the known
  example) silently rewrites the corpus's bytes, breaking a clean byte-for-byte compare
  on the next sync. Merge such PRs with a normal merge commit.

## Refresh-before-publish checklist

1. **Find what landed since the last snapshot.** Identify the date/commit the corpus
   was last frozen (the previous publish, or the extraction). List the work items and
   commits to the source operating rules since then:

   ```bash
   git log --since=<last-snapshot-date> --oneline -- <your-rules-dir> <your-claude-config>
   ```

   The relevant sources are wherever your operating doctrine lives — agent
   instructions, rule files, lifecycle scripts, and the backlog/feature records that
   explain *why* a rule changed.

2. **Triage each change against the genericization bar.** For every change, ask: *is
   this a transferable doctrine shift, or project-specific detail?*
   - **Transferable** (a new invariant, a changed model, a hard-won gotcha that any
     team would hit) → port it into the matching corpus file, in generic `«slot»`
     form.
   - **Project-internal** (tool internals, a renamed local path, a one-off) → skip it,
     and note below the bar that you consciously skipped it (so the next refresh
     doesn't re-litigate).

3. **Re-run the genericization litmus.** No project-specific noun should survive.
   Adapt this grep to your project's identifiers:

   ```bash
   grep -rniE "<your-project-name>|<your-domain-nouns>|<your-core-type-files>" .
   ```

   Any hit is a candidate to genericize. Legitimate generic uses of a word (e.g.
   "type *contract*", a persona *roster*) are fine — judge by whether a reader
   outside your project would be confused or would just read it as the common term.

4. **Re-check the slot inventory.** If you added or changed slots, update:
   - the **per-file** parameterization / "adapt this" section in the file you touched, and
   - the **consolidated table** in [`PARAMETERIZATION.md`](PARAMETERIZATION.md).

   List every slot and confirm each appears in the consolidated table:

   ```bash
   grep -rho "«[^»]*»" *.md | sort -u
   ```

5. **Re-read the navigation table.** If you added, removed, or retitled a file,
   update the navigation table and the "read it when" column in the README.

6. **Re-sync the altitude headers.** The corpus carries a layered on-ramp — two sibling
   on-ramps, [`OVERVIEW.md`](OVERVIEW.md) (the anatomy: ideas named one at a time) and
   [`A-DAY-IN-THE-LIFE.md`](A-DAY-IN-THE-LIFE.md) (the physiology: those ideas shown in
   motion across one slice), plus a standardized header on each chapter; when a body
   changes, the layers above it can silently go stale. For every chapter you edited:
   - Re-check its **`TL;DR — the mechanisms`** block still matches the body's actual
     mechanisms (a removed/added Part means a removed/added bullet).
   - Re-check the **`In one line`** and **`Read this when`** lines still hold.
   - If the change touched one of the seven big ideas, re-check that idea's paragraph in
     `OVERVIEW.md`, the chapter's `Big ideas` / `Depends on` back-links, **and** the
     beat-index table in `A-DAY-IN-THE-LIFE.md` (it labels each beat with an idea + chapter,
     so a renamed idea or retitled chapter stales a cell or a link there).
   - If you added, removed, or retitled a chapter, update the `OVERVIEW.md` map table and
     the chapter's row, add/remove its altitude header, and re-check the
     `A-DAY-IN-THE-LIFE.md` beat-index links into it.

7. **Run the consistency check.** A grep-level sweep that catches header drift, broken
   cross-links, and orphaned glossary terms. None of these should print anything:

   ```bash
   # (a) every chapter has the three required header elements
   for f in *-doctrine.md stakeholder-persona-guide.md stakeholder-persona-template.md; do
     for needle in '> **In one line' '**Read this when' '**Big ideas:'; do
       grep -qF "$needle" "$f" || echo "MISSING [$needle] in $f"
     done
   done

   # (b) every cross-link into an OVERVIEW heading resolves to a real heading slug
   grep -rhoE 'OVERVIEW\.md#[a-z0-9-]+' *.md | sort -u | sed 's#OVERVIEW.md##' | while read -r a; do
     grep -E '^#{1,3} ' OVERVIEW.md \
       | sed -E 's/^#+ //; s/[^A-Za-z0-9 -]//g; s/.*/\L&/; s/ +/-/g; s/^/#/' \
       | grep -qx "$a" || echo "BROKEN anchor $a"
   done

   # (c) every Depends-on / sibling chapter link points at a file that exists
   grep -rhoE '\]\(([a-zA-Z0-9_-]+\.md)\)' *.md | sed -E 's/.*\(([^)]+)\)/\1/' | sort -u \
     | while read -r m; do [ -f "$m" ] || echo "DEAD link target $m"; done

   # (d) every bolded glossary term in OVERVIEW.md appears in some chapter body
   chapters="$(ls *-doctrine.md) stakeholder-persona-guide.md stakeholder-persona-template.md"
   grep -oE '^- \*\*[^*]+\*\*' OVERVIEW.md | sed -E 's/^- \*\*([^*]+)\*\*/\1/; s# /.*##' \
     | while read -r term; do
       grep -rilqF "$term" $chapters || echo "ORPHAN glossary term: $term"
     done
   ```

   A `MISSING`/`BROKEN`/`DEAD`/`ORPHAN` line is a real defect — fix it before publishing.
   (Check (d) is advisory: a glossary term may legitimately appear only in OVERVIEW's own
   pitch or map, not in a chapter body — confirm by eye rather than auto-trusting it.)

8. **Re-publish the snapshot.** Push the refreshed files to the publish target and
   update the recorded URL / date wherever you track it. State the new snapshot date
   in the README's provenance section if you keep one there. *(If the corpus is vendored,
   this "push" is the clean tree-export `commit-tree` recipe — see
   [Transport](#transport-vendoring-via-a-clean-tree-export); never `git subtree push`.)*

## What never gets ported

- **Runnable tool internals** (CLI argument parsing, file-path constants, framework
  glue). The doctrine documents the *contract*; the code stays at the source.
- **Project-specific content and examples.** Genericize or drop.
- **Churn that didn't change the doctrine.** A refactor that left the invariant
  intact is not a corpus change.

Keep **model-specific** guidance (e.g. a current model's behavioral defaults) — that
transfers to anyone using the same model, even though it's not "project-agnostic" in
the strict sense.
