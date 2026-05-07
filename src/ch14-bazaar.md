# 14. Bazaar

## Origin

Bazaar — the system, with the command-line tool `bzr` — emerged in 2005 from Canonical Ltd., the company behind Ubuntu Linux. Mark Shuttleworth had decided that Canonical needed a usable DVCS for distributing Ubuntu development across the company's contributors, and the existing options (BitKeeper proprietary, GNU Arch unusable, Mercurial and Git both very young) did not satisfy. Canonical hired Martin Pool, who had been working on Arch's `baz` re-implementation, to lead the project. The original `bzr` was effectively a continuation of `baz` with a clean rewrite in Python. The 1.0 release came in 2008.

The system's defining commitment was *flexibility of workflow*. Bazaar took the position that a version control system should not impose centralized vs. distributed as a single mode of work; both should be first-class, and a single repository should be usable in either mode without conversion. A *branch* in Bazaar could be local-only, or it could be bound to a remote master (centralized mode, where commits to the local branch are also sent to the master immediately, much like Subversion), or it could be unbound (the standard DVCS clone-fork-merge mode).

The system also took rename handling seriously. Following Arch, files had stable IDs that survived rename and copy; `bzr log file.py` followed the file's history across the rename. This was an explicit design choice, deliberately different from Git's heuristic-at-view-time approach.

Bazaar was hosted on Launchpad, Canonical's free-software project hosting service. Launchpad was deeply integrated with `bzr`: project metadata, bug tracking, blueprint planning, and code hosting all worked together. Ubuntu development happened on Launchpad. MySQL, MariaDB, and several other projects used Bazaar at the height of its adoption.

The peak was around 2010. By 2012, Git had won enough mindshare that Bazaar's growth flatlined and then declined. Canonical scaled back its investment. By 2017, the project was officially in maintenance mode at Canonical, and a community fork called *Breezy* (`brz`), led by Jelmer Vernooij and contributors, took over active development. Breezy continues as of 2026: occasional releases, support for Python 3, integration with Git via the dulwich library, used in a long tail of mostly-Debian-adjacent projects.

The chapter covers Bazaar primarily; Breezy carries the lineage forward without changing the model.

## The system on its own terms

Bazaar's storage is a *repository* (which holds versioned data) and one or more *branches* (which point to a particular revision in the repository). A *working tree* is a checkout of a branch. The split is more explicit than Git's: a Bazaar repository may exist without a working tree (a *bare* repository in Git's terminology) and may host multiple branches.

The data model is similar to Git's at a high level: revisions form a DAG, with parent pointers, content, and metadata. The differences are in details:

- *File IDs.* Each tracked file has a stable ID. Renames preserve the ID; the system tracks the rename in the revision metadata.
- *Revision IDs.* A revision is identified by a string like `aditi@example.org-20260104120000-abcdef1234567890`, combining the committer's email, timestamp, and a content hash. This is human-readable in a way Git hashes are not.
- *Branch nicknames.* A branch has a nickname (the project team's name for it), separate from the path on disk. `bzr nick` queries or sets it.

Branches can be:

- *Independent* — a normal DVCS branch, fork-and-merge.
- *Bound* — every commit is also pushed to a master branch, enforcing a centralized commit ordering. The master can be on Launchpad or on any reachable Bazaar repository.
- *Checkout* — a working tree pointing at a remote branch, with no local history (lightweight checkout) or with local history (heavyweight checkout).

This three-mode flexibility is Bazaar's distinctive contribution. A team that wants strict centralized control uses bound branches with `bzr commit`; a team that wants standard DVCS uses unbound branches with `bzr push`; a team that wants the centralized model with offline capability uses heavyweight checkouts. The same `bzr` commands serve all three.

The CLI is `bzr` (or `brz` for Breezy). Subcommands include `bzr init`, `bzr add`, `bzr commit`, `bzr push`, `bzr pull`, `bzr merge`, `bzr branch`, `bzr checkout`, `bzr log`, `bzr diff`, `bzr status`, `bzr revert`, `bzr uncommit`, `bzr tag`. The grammar is consistent and the help is good; users coming from Git find the experience pleasant.

## Scenario walkthrough

### Operation 1 — Initial import

Aditi initializes a repository:

```
$ bzr whoami "Aditi Rao <aditi@example.org>"
$ cd /path/to/initial/ledger
$ bzr init
$ bzr add .
$ bzr commit -m "Initial import"
```

The first commit creates revision 1 in the local branch. The repository is local; no remote is configured yet. To publish, Aditi pushes to a server (Launchpad, an HTTP host, or any reachable directory):

```
$ bzr push lp:~aditi/ledger/main
```

### Operation 2 — Linear development

```
$ vi src/ledger/parser.py
$ bzr commit -m "parser: handle blank lines and comments"
```

Each commit is a revision. The local branch advances; subsequent `bzr push` pushes the new revisions to the published location.

### Operation 3 — Branch

Jonas branches Aditi's published repository:

```
$ bzr branch lp:~aditi/ledger/main ledger
$ cd ledger
```

For the `currency-conversion` work, he creates a new branch off this:

```
$ bzr branch . ../ledger-cc
$ cd ../ledger-cc
$ vi src/ledger/parser.py
$ bzr commit -m "parser: store amounts as (value, currency)"
```

Branches in Bazaar are conventionally separate directories, like early Mercurial. There are also "colocated branches" (a Bazaar feature for keeping multiple branches in one working tree), but the dominant pattern is per-directory branches.

### Operation 4 — File rename

```
$ bzr mv src/ledger/reports.py src/ledger/report_balance.py
$ # edit report_balance.py
$ bzr add src/ledger/report_register.py
$ bzr commit -m "reports: split into balance and register modules"
```

The rename is recorded with the file's stable ID. `bzr log src/ledger/report_balance.py` follows the file across the rename without heuristics.

### Operation 5 — Binary file added

Bazaar adds binaries without configuration:

```
$ bzr add docs/logo.png
$ bzr commit -m "docs: add project logo"
```

Storage uses Bazaar's pack format, which handles binary content reasonably (no special compression for binaries, but no corruption either). For very large binary files, Bazaar's performance degrades; the system was not optimized for the multi-gigabyte case.

### Operation 6 — Parallel edits

Aditi and Jonas both edit `README.md`. After Aditi's commit and push, Jonas's push is rejected:

```
$ bzr push
bzr: ERROR: These branches have diverged.
$ bzr pull
$ # or, more commonly
$ bzr merge lp:~aditi/ledger/main
$ bzr commit -m "Merge Aditi's README changes"
$ bzr push
```

The diverged-branches case is named clearly; the recovery is straightforward.

### Operation 7 — Merge with conflict

For the `currency-conversion` merge, Jonas (or Aditi) merges the branch into the main:

```
$ bzr branch lp:~aditi/ledger/main main
$ cd main
$ bzr merge lp:~jonas/ledger/cc
... merging ...
2 conflicts encountered.
$ # resolve conflicts in working tree
$ bzr resolve src/ledger/parser.py
$ bzr commit -m "Merge currency-conversion into main; resolve parser conflict"
$ bzr push
```

The merge produces a revision with two parents. `bzr log` shows the merge structurally.

### Operation 8 — Botched commit

Bazaar supports rewriting the latest commit before push:

```
$ bzr uncommit
$ # working tree now has the changes back; the commit is reverted
$ vi src/ledger/report_register.py    # remove debug print
$ bzr commit -m "register: add --csv flag"
```

`bzr uncommit` removes the latest revision from the local branch (the changes remain in the working tree, ready to be re-committed). This is the analogue of `git reset --soft HEAD^`. After push, the standard advice is to commit forward; rewriting published history requires `--overwrite` flags and team agreement.

### Operation 9 — Tag

```
$ bzr tag v0.1.0
```

Tags are first-class, attached to revisions, and travel with branches when pushed.

### Operation 10 — Release

```
$ bzr export ledger-0.1.0.tar.gz lp:~aditi/ledger/main -r tag:v0.1.0
```

`bzr export` produces a clean tarball without `.bzr/` metadata at the specified revision.

## Model and mental load

What you have to hold:

- The repository/branch/working-tree split. More explicit than Git's, less than Subversion's.
- The bound/unbound distinction. Knowing which mode a branch is in matters.
- File IDs. Mostly invisible, but the basis for rename tracking.
- Launchpad-specific paths (`lp:~user/project/branch`) if you use Launchpad.

The mental load is moderate, lower than Git's for new users by most contemporary accounts. Bazaar's ergonomics were a deliberate design priority, and they showed.

## Evolution and history rewriting

`bzr uncommit` for the latest revision; `bzr replay` and various plugins for more elaborate rewriting; `bzr-rewrite` plugin for interactive history editing. The cultural stance is moderate: rewriting is acceptable for unpublished history, frowned upon for published, with the system's tools matching that posture.

## Ecosystem reality

Bazaar proper is in maintenance. Breezy is the active fork; it builds and runs in 2026, supports Python 3, integrates with Git repositories, and is used in some Debian and Ubuntu adjacent infrastructure. The user base is small. New projects almost never adopt it.

Launchpad is still up and continues to host Bazaar repositories; Canonical has shifted most active Ubuntu development off Launchpad-Bazaar to GitHub or Git over Launchpad in recent years. The Bazaar-on-Launchpad hosting is in slow decline, with many projects mirroring to or migrating to Git.

The plugin ecosystem includes `bzr-git` (read and write Git repositories from Bazaar), `bzr-svn` (Subversion interop), and a number of integrations with editors and tools. These are valuable for migration purposes; few are actively developed.

## When to reach for it; when not to

For new work, do not. Git is mature, more tooling exists, the team you hire knows it.

If you are working in a project that uses Bazaar — some Debian packaging, some Launchpad-hosted projects, some specific tools that have not migrated — Breezy is the supported way to do it, and the experience remains pleasant. The system is not bad; it is just not winning.

If you are looking at version control as an academic exercise, Bazaar is worth installing and trying. The flexibility-of-workflow position is unmatched in any other system. The bound-branch idea, in particular, is one of the things this book counts as a small piece of what got lost.

## Epitaph

Bazaar was the DVCS that thought workflow flexibility was the central design problem and proved it could be solved without sacrificing model coherence — and it lost to a system that picked one workflow and bet everything on it.
