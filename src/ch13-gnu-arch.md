# 13. GNU Arch and tla

## Origin

GNU Arch was Tom Lord's project, started around 2001 and active through about 2006. The name *arch* described an idea about version control architecture; *tla*, the C implementation, stood for *Tom Lord's Arch*, a name everyone in the project insisted was self-deprecating. There was a Python re-implementation called *baz* that briefly forked, and later Canonical's Bazaar product borrowed the name without sharing much code. *GNU Arch* was the umbrella name once the project moved under the Free Software Foundation banner.

The system mattered because it was the first serious attempt at a free-software distributed version control system. BitKeeper was the existence proof that DVCS could work; everything proprietary about it was the problem. Lord set out to build a free-software DVCS that took the same architectural position — distributed, with full local history, with peer-to-peer synchronization — and made it available under terms that the kernel community and the rest of the free-software world would consider acceptable.

The problem with Arch was usability. The system was conceptually rigorous, in some ways more so than any of its successors, and the UX was famously among the worst in any version control system the field has produced. Naming conventions were baroque (`lord@emf.net--2003/arch--devo--1.0--patch-23`), commands were dense, error messages were unhelpful, and the storage layout was enough of a maze that working sysadmins rebelled. The conceptual contribution was enormous; the practical contribution was that the next generation of DVCS designers learned to do better.

Active development tapered off around 2007. The system has been effectively dead since 2010. Repositories that survive in any form are mostly archived snapshots; migration tooling is crude and rarely needed.

## The system on its own terms

Arch's storage was an *archive*: a directory tree containing per-revision files representing changes. Archives lived on a filesystem and were accessed over filesystem paths or via FTP, HTTP, SFTP, or rsync — Arch did not ship its own server. This made distribution maximally flexible (any HTTP server could host an archive, with no special software) and synchronization correspondingly slow over the wire.

The naming convention was the system's signature flaw. A *full archive name* identified the archive (`lord@emf.net--2003`), the *category* within it (`arch`), the *branch* (`devo`), the *version* (`1.0`), and the *revision* (`patch-23`). Concatenated, this produced strings like `lord@emf.net--2003/arch--devo--1.0--patch-23`. Every command that needed to refer to a remote revision needed the full name; teams developed shell aliases and shorthand systems to avoid typing the strings repeatedly.

The unit of history was the *changeset*, equivalent to a project-level commit, with patches against parent revisions. The changesets were stored as `tar.gz` archives by default. Operations like log walked the archive's revision list to construct history.

Branches were first-class. A branch was a *version* in the namespace above; branching meant declaring a new branch and committing changesets to it. Merging used `tla star-merge`, which performed a three-way merge tracking which patches had been applied to which branches.

The CLI was `tla`. Subcommands included `tla make-archive`, `tla register-archive`, `tla my-id` (set user identity), `tla init-tree`, `tla import`, `tla add`, `tla commit`, `tla update`, `tla replay`, `tla star-merge`, `tla missing`, `tla logs`, `tla cat-archive-log`. The grammar was idiosyncratic; the help output ran to many pages.

## Scenario walkthrough

The exemplar in Arch is largely a study in elaborate command lines.

### Operation 1 — Initial import

```
$ tla my-id 'Aditi Rao <aditi@example.org>'
$ tla make-archive aditi@example.org--ledger /var/lib/tla/aditi-ledger
$ tla archive-setup ledger--main--0.1
$ cd /path/to/initial/ledger
$ tla init-tree ledger--main--0.1
$ tla add-id $(find . -type f -not -path '*/{arch}/*')
$ tla import
... commits to base-0 ...
```

The first commit is *base-0*, the archive's foundation revision. Every subsequent commit is a *patch-N* incrementing from there. The naming convention starts here and never lets up.

### Operation 2 — Linear development

```
$ vi src/ledger/parser.py
$ tla commit
... patch-1 ...
```

`tla commit` opens an editor for the log message. The patch number increments; the archive accumulates a `patch-1` directory.

### Operation 3 — Branch

For Jonas's `currency-conversion` branch, the team uses Arch's tag mechanism to create a new line of development:

```
$ tla tag aditi@example.org--ledger/ledger--main--0.1 \
         aditi@example.org--ledger/ledger--currency-conversion--0.1
```

Jonas registers the archive on his own machine, gets the tagged revision as a starting point, and commits to the new branch:

```
$ tla register-archive aditi@example.org--ledger http://aditi-host/ledger
$ tla get aditi@example.org--ledger/ledger--currency-conversion--0.1 work
$ cd work
$ vi src/ledger/parser.py
$ tla commit
```

The archive now has two parallel lines of development with different names.

### Operation 4 — File rename

Arch supports rename via *file IDs*. Each tracked file has a unique ID in the `{arch}/=tagging-method` configuration. When a file is renamed, its ID stays the same, and Arch tracks the rename:

```
$ tla mv src/ledger/reports.py src/ledger/report_balance.py
$ # edit report_balance.py
$ tla add src/ledger/report_register.py
$ tla commit
```

The history of the renamed file is preserved by ID. `tla logs` filtered by file ID follows the file across the rename.

### Operation 5 — Binary file added

Binaries are added without ceremony; Arch's storage handles them by treating the bytes as opaque. There is no special file-type mechanism. The PNG is added with `tla add` and committed.

### Operation 6 — Parallel edits to the same file

Aditi and Jonas both edit `README.md` in their respective working trees. Arch supports this in the standard DVCS rhythm: each commits to their own copy of the archive, then the changesets are exchanged via `tla update` and merged with `tla star-merge`.

### Operation 7 — Merge with conflict

```
$ tla star-merge aditi@example.org--ledger/ledger--currency-conversion--0.1
... merges, reports conflicts in parser.py ...
$ vi src/ledger/parser.py
$ tla commit
```

The merge is recorded as a patch; Arch tracks which patches have been merged where via the *patch-log* mechanism, where each branch records which patches from other branches it has applied. This was an early version of merge tracking that worked correctly more often than the contemporary alternatives.

### Operation 8 — Botched commit

Arch supports `tla undo` to back out the latest commit on the local branch (before sharing). After a `tla commit` is published — written to the archive — it is in history. Mireille's correction is a follow-up commit, with the audit trail preserving the bad commit.

### Operation 9 — Tag

`tla tag` creates a new version (in the Arch sense) at a particular revision; the result is a named branch with a single revision. This is the closest Arch came to lightweight tagging. Symbolic tags (Subversion-style) were not first-class; the convention was to use a dedicated archive for releases.

### Operation 10 — Release

`tla get` of the tag's revision produces a clean working tree. Tar it, ship it. The conventional Arch release workflow involved a separate "releases" archive into which release artifacts (the tagged trees) were committed by the release manager.

## Model and mental load

What you have to hold:

- The full archive naming convention. Every remote revision needs the full string until you set up shorthand.
- The split between archive and working tree. The archive lives on disk; the working tree is a checkout from it.
- The patch-log mechanism. Merging tracks which patches have been seen by which branches, and reasoning about it is part of merging.
- The sync conventions. Pushing changes means committing to an archive that is reachable by your collaborators (filesystem path, HTTP, FTP).

The mental load is high *because of the naming and tooling*, not because the underlying ideas are exotic. The ideas were sound. The grammar was unusable.

## Evolution and history rewriting

Arch had `tla undo` for local revisions and various surgical operations on the archive. The cultural stance was conservative; rewriting was discouraged.

## Ecosystem reality

Dead. The Arch and tla projects have not seen meaningful releases in over a decade. The free-software world that needed a usable DVCS got Bazaar, Mercurial, and Git instead, and the Arch ecosystem evaporated as those alternatives matured.

Bazaar, Canonical's later DVCS, took some of Arch's better ideas (rename tracking by file ID, the patch-log style merge tracking) and shed almost all of its UX. Bazaar is in the next chapter. The Arch lineage in 2026 is essentially the lineage that fed forward into Bazaar and then mostly stopped.

## When to reach for it; when not to

Do not reach for it. If you find an old Arch archive, the realistic move is to convert it to Git via one of the (rough) migration tools and treat the conversion as an artifact extraction.

## Epitaph

GNU Arch was the right idea with the wrong fingers, and is the system that taught the next generation what UX cost looks like when the underlying design is correct.
