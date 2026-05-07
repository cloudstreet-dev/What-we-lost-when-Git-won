# 15. Mercurial

## Origin

Mercurial was started by Matt Mackall in early April 2005, the same week BitKeeper revoked the kernel project's free license and Linus Torvalds began work on Git. Mackall, a kernel developer himself, had been thinking about the same problem from the same angle. His first public release, on April 19, 2005, came twelve days after Linus's first Git commit; for several months the kernel community considered both systems as candidates before settling on Git. Mercurial took a different path: it found a different community.

The system's design priorities were *correctness*, *clarity*, and *speed*, in that order. Mackall came to the project from a long history of work on free software — he had written the `bart` patch manager and was a familiar figure on linux-kernel — and his sensibility shows in Mercurial's bones. Where Git is a heap of useful tools held together by a content-addressed object store, Mercurial is a single coherent program with a deliberately curated user surface. The phrase "principle of least surprise" appears more than once in the project's design discussions; the project tried, and largely succeeded, in producing a DVCS where doing the obvious thing usually worked.

Mercurial gathered users steadily through the late 2000s and reached its high-water mark around 2012. Mozilla migrated all of `mozilla-central` from CVS to Mercurial in 2007 and ran on it for thirteen years. Facebook adopted Mercurial in 2014 for what became the largest production Mercurial deployment in history; the work that started there eventually became Sapling, covered in its own chapter later. Bitbucket, Atlassian's hosted forge, was Mercurial-only initially and continued to support both Mercurial and Git for many years.

The decline began when GitHub's network effects became insurmountable. New developers learned Git first; new tooling assumed Git; new projects defaulted to Git. Bitbucket dropped Mercurial support in 2020. Mozilla announced its own migration off `mozilla-central` to Git in 2023, completing it in 2024–2025. By 2026 Mercurial is alive — actively maintained, occasionally improved, with releases continuing — but quietly, in a way that matches its temperament. Facebook's monorepo work has its own future as Sapling. The independent Mercurial community is small, deeply expert, and not in growth.

## The system on its own terms

A Mercurial repository is a `.hg/` directory containing the *revlogs*. A revlog is Mercurial's append-only storage format for a versioned object. Each tracked file has its own revlog (`.hg/store/data/...`); the *changelog* (a special revlog) records the sequence of changesets; the *manifest log* records the trees. The format is documented and stable; reading it from outside the program is a tractable exercise.

Each entry in a revlog stores either a full snapshot or a delta against an earlier entry, with a chain length cap so that no entry takes more than a bounded number of delta applications to reconstruct. The cap balances storage cost against retrieval latency. The choice — neither pure snapshot (Git) nor pure delta (RCS) but a hybrid with caps — is one of the design decisions that gives Mercurial its predictable performance.

Changesets are identified by SHA-1 hashes (with a SHA-256 transition in progress, similar to Git's). The local repository also assigns a per-clone *integer revision number* (`0`, `1`, `2`, ...) that serves as a human-friendly shorthand. The integer is local — `r42` in Aditi's clone might be `r45` in Jonas's — but for casual reference within a single clone it is more convenient than typing hashes. Most `hg` commands accept either form, and `hg log -r 42:50` is a perfectly valid range query.

Branches in Mercurial come in three flavors:

- *Named branches*. The branch name is part of changeset metadata; commits "belong to" a named branch by construction, and the relationship survives push/pull.
- *Bookmarks*. Git-style movable references that point to a changeset. Bookmarks travel between repositories on push/pull (with explicit flags) and behave like branches in any DVCS that uses them.
- *Anonymous heads*. Two changesets with the same parent and the same branch name are *both* heads of that branch. Mercurial reports this as a divergence; tools work with both heads simultaneously.

Most modern Mercurial workflows use a mix of bookmarks and named branches: named branches for long-lived release lines, bookmarks for feature work.

Extensions are first-class. Many features that other systems consider core are Mercurial extensions, enabled in a configuration file:

- *rebase* — `hg rebase`, equivalent to `git rebase`.
- *histedit* — interactive history editing.
- *mq* — Mercurial Queues, a patch-management system that predates Git's notion of an interactive rebase. Originally by Chris Mason, modeled on `quilt`.
- *evolve* — Pierre-Yves David's mutable-history extension that records the *evolution* of changesets across rewrites, allowing `hg evolve` to re-apply edits, propagate amendments to descendants, and recover from rebases gone wrong. One of the design ideas that fed into Sapling and Jujutsu.
- *largefiles* and *lfs* — large-file handling.
- *strip* — removing changesets from history.

The CLI is `hg`. Subcommands are concise and consistent. Help text is unusually good. The error messages prefer to suggest a corrective command rather than complaining cryptically.

## Scenario walkthrough

### Operation 1 — Initial import

```
$ cd /path/to/initial/ledger
$ hg init
$ hg add .
$ hg commit -u "Aditi Rao <aditi@example.org>" -m "Initial import"
```

The first commit is changeset 0 in this repository. To publish, `hg push` to a server (an HTTP server with `hg serve` or a static server with hg's flat repository format).

### Operation 2 — Linear development

```
$ vi src/ledger/parser.py
$ hg commit -m "parser: handle blank lines and comments"
```

Each commit advances the local rev counter and produces a hash.

### Operation 3 — Branch

Jonas clones:

```
$ hg clone http://aditi-host/ledger
```

For the `currency-conversion` work, the modern idiom is a bookmark:

```
$ hg bookmark currency-conversion
$ vi src/ledger/parser.py
$ hg commit -m "parser: store amounts as (value, currency)"
```

The bookmark moves with each commit. To push the bookmark:

```
$ hg push -B currency-conversion
```

For projects that use named branches, the alternative is:

```
$ hg branch currency-conversion
$ hg commit -m "..."    # records branch in changeset metadata
```

### Operation 4 — File rename

```
$ hg mv src/ledger/reports.py src/ledger/report_balance.py
$ # edit report_balance.py
$ hg add src/ledger/report_register.py
$ hg commit -m "reports: split into balance and register modules"
```

Renames are recorded in changeset metadata; `hg log --follow src/ledger/report_balance.py` follows the rename. Forgetting the `hg mv` and instead using OS rename leaves Mercurial seeing a delete-and-add; the `hg addremove --similarity 80` command tries to recover the rename heuristically.

### Operation 5 — Binary file added

By default, Mercurial adds binaries without ceremony:

```
$ hg add docs/logo.png
$ hg commit -m "docs: add project logo"
```

The revlog format handles binary content; storage is reasonable. For very large binaries, the team would enable the `lfs` extension, which moves files matching a pattern to an LFS server and stores only pointers in the repository.

### Operation 6 — Parallel edits

Aditi and Jonas both edit `README.md`. After Aditi's commit and push, Jonas's push is rejected:

```
$ hg push
abort: push creates new remote head 5a3c...
(merge or see 'hg help push' for details about pushing new heads)
$ hg pull
$ hg merge
$ hg commit -m "Merge Aditi's README changes"
$ hg push
```

The error message is informative; the recovery is standard.

### Operation 7 — Merge with conflict

For the `currency-conversion` merge:

```
$ hg update default     # switch to main branch (the default named branch)
$ hg merge currency-conversion
... merging parser.py ...
warning: 1 conflicts while merging src/ledger/parser.py! (edit, then use 'hg resolve --mark')
$ # resolve markers in parser.py
$ hg resolve --mark src/ledger/parser.py
$ hg commit -m "Merge currency-conversion into main; resolve parser conflict"
```

The merge produces a changeset with two parents.

### Operation 8 — Botched commit

Mercurial's default stance is conservative: published history is permanent, and the standard tools treat it that way. To rewrite *unpublished* history, the tools are extension-driven:

With `histedit` enabled:

```
$ hg histedit -r .~1::.
... opens an editor with a list of changesets and actions: pick, edit, fold, drop, mess ...
```

Or, simpler, to amend the latest commit:

```
$ hg commit --amend -m "register: add --csv flag"
```

For Mireille's botched commit, before push, `hg commit --amend` after editing the file to remove the debug line is the smooth path. After push, `hg backout` creates a follow-up changeset that reverses the bad change; the original is preserved in history.

The cultural stance is the audit-trail one. Rewriting unpublished history is fine; rewriting published history requires extensions and team agreement.

### Operation 9 — Tag

```
$ hg tag v0.1.0
```

Tags in Mercurial are an interesting design choice: they are stored in a versioned file (`.hgtags`) rather than as out-of-band metadata. Adding a tag is itself a commit that updates `.hgtags`. The tag travels with the repository in a way that is auditable and survives clones cleanly. Some users find this elegant; some find it surprising that tagging produces a changeset.

### Operation 10 — Release

```
$ hg archive -r v0.1.0 ledger-0.1.0.tar.gz
```

`hg archive` produces a clean tarball at the specified revision. The release notes are committed to `CHANGELOG.md` before the tag.

## Model and mental load

What you have to hold:

- The revlog format, conceptually: append-only per-file history with snapshot-or-delta entries.
- Changesets, identified by hash and by local integer revision number.
- The branch model: named branches, bookmarks, anonymous heads.
- The extension model: which extensions are enabled, what each does.
- The cultural stance on history mutability: rewrite local, preserve published.

The mental load is moderate, lower than Git's by most accounts. The CLI is forgiving; errors guide; the model is internally consistent. Users coming from Subversion adapted to Mercurial more easily than to Git in the late 2000s, and the difference was not narrative — it was real.

## Evolution and history rewriting

The `evolve` extension is the most interesting piece of work in this area in any system. Rather than rewriting history in place, `evolve` records that a changeset has been *obsoleted* and *replaced* by another. The graph still contains the old changeset, marked as obsolete; the new changeset is its successor. Tools that follow the evolve model can show the user "this is the third version of the changeset that started as X" rather than "X is gone and Y is here." Rebases preserve the link; pulls resolve obsolescence; the model is more nuanced than Git's "rewrite and rely on the reflog" approach.

`evolve` was experimental for years and has not been folded into core Mercurial. It is, however, one of the design ideas that informed Sapling and Jujutsu, both of which build something like obsolescence into their core models. Reading evolve's documentation is one of the more rewarding exercises in this book.

## Ecosystem reality

Mercurial in 2026 is alive, maintained, and quiet. Releases come from the project (recently 6.x); the core team is small. `mercurial.mozilla.org` is a read-only archive; Mozilla is on Git. Bitbucket has been Git-only since 2020. Heptapod, a GitLab fork that adds Mercurial support, exists and is used by some projects (notably the Octave project). Some self-hosted instances continue.

The Sapling project at Meta carries the Mercurial intellectual lineage forward in a substantially evolved form; Mercurial proper continues for the projects that were on it and have not migrated. The independent open-source Mercurial community is small but committed.

## When to reach for it; when not to

Reach for Mercurial when:

- Your team values a CLI that does not surprise.
- You have an existing Mercurial repository that is working fine.
- You want to use Sapling-style features (covered later) but cannot deploy Sapling proper, and `evolve` plus careful workflow gives you most of what you wanted.

Do not reach for Mercurial for new public open-source work. The hosting story is constrained; the contributor pool that would prefer Mercurial is small. The migration cost will be paid eventually.

## Epitaph

Mercurial was the DVCS that thought about UX, picked the friendlier defaults, and produced the system the field would recommend to a friend if recommendations weren't downstream of network effects.
