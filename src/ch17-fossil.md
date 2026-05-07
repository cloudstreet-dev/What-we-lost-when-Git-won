# 17. Fossil

## Origin

Fossil was created by D. Richard Hipp in 2007. Hipp is best known as the author of SQLite, and Fossil grew out of SQLite's needs as a project: Hipp wanted a version control system that would let him manage SQLite's own source code, its bug tracker, its wiki, and its development website as a single integrated artifact, with the property that anyone in the world could clone the entire project — code, bugs, history, wiki — to their laptop in seconds and operate it offline. CVS was the conventional answer at the time and was wrong on every axis. Git existed but optimized for different things and required a separate ecosystem of forge tools to fill the gaps Fossil wanted filled by the system itself.

Fossil's design priorities, drawn from Hipp's own writing on the topic: *simple deployment* (one binary, no external dependencies), *complete project artifact* (code plus issues plus wiki plus forum), *durable history* (commits are intended to be permanent), *self-contained hosting* (the same binary that serves a working developer is the binary that runs the server), and *small footprint*. The result is unlike anything else in this book.

The first public Fossil release was in 2007. Versions have continued steadily; Fossil 2.x is current in 2026. The system is used by SQLite (of course), by Tcl/Tk, by a handful of small open-source projects, and by individual developers who have come across it and stayed. Fossil has not won a market in the sense Git has, and Hipp does not appear interested in trying. The system is stable, used, maintained, and small.

## The system on its own terms

Fossil is one binary. The same `fossil` executable runs the developer commands, runs the web UI, runs the HTTP server, performs synchronization, and reads the storage format. The binary is around ten megabytes; it has no runtime dependencies beyond a C library; it builds from source with a single `make` on any modern Unix or Windows system.

The storage backend is SQLite. A Fossil repository is a single SQLite database file (`.fossil` by convention). Schemas are stable; the format can be read by anyone with a SQLite library; backups are file copies. The choice of SQLite is the through-line of Hipp's design philosophy: pick a small, transactional, well-understood storage layer; build a clean schema over it; make the result portable.

A *checkout* is a directory tree extracted from a repository. The operation is `fossil open path/to/repo.fossil` from within the directory; it populates the working tree with the latest revision and creates a `.fslckout` file (a small SQLite database itself) to track the checkout's state. Multiple checkouts of the same repository can coexist; the repository file is the source of truth.

The unit of history is the *check-in* — Fossil's term for a commit. Check-ins form a DAG with parent pointers, hashed by SHA-256 (formerly SHA-1; SHA-256 is the current default and has been since 2017). Each check-in records the tree state (as a manifest file referencing artifacts in the SQLite blob store), parents, author, timestamp, and comment.

Branches are a property of check-ins, recorded as a *tag* on the check-in. A check-in's branch is whatever its tag says. Branches "exist" by virtue of having check-ins tagged onto them; switching branches is `fossil checkout BRANCHNAME`. Tags themselves are first-class, with several kinds: `branch` tags (which name branches), `cancel` tags (used for the deletion of other tags), `closed` tags (which mark a branch as closed without deleting it), and arbitrary user tags.

Synchronization is *autosync*. By default, every commit pushes to the remote and every checkout pulls from it. This is the opposite of Git's defer-and-batch model and is one of Fossil's most distinctive choices. Hipp's argument, in the manual: most of the bugs DVCS users hit are caused by working on stale data, and forcing sync at every operation is the simplest way to prevent that. Autosync can be disabled; it usually is not.

The web UI is built into the binary. `fossil ui` launches a local server on a random port and opens a browser; `fossil server` runs a long-running server suitable for hosting. The UI shows commits, files, blame, branches, tags, the timeline, the wiki, the bug tracker, the forum, the technical notes (Fossil's structured "tech notes"), search, and administrative controls. It is rendered as plain HTML with a few CSS niceties; it loads in 50 milliseconds; it works without JavaScript.

The bug tracker is a set of SQL tables in the repository database. Bugs (called "tickets" in Fossil) are SQL rows; comments and status changes are SQL rows; the schema is customizable. A clone of the repository contains every ticket. A push or pull of a Fossil repository synchronizes the tickets along with the code.

The wiki is similar: pages are stored in the repository database, versioned alongside code, synchronized by push and pull. The forum (added in later versions) is the same. The integration is the system's defining property.

## Scenario walkthrough

### Operation 1 — Initial import

Aditi creates the repository and checks it out:

```
$ mkdir ~/fossils && cd ~/fossils
$ fossil init ledger.fossil --admin-user aditi
$ mkdir ~/work/ledger && cd ~/work/ledger
$ fossil open ~/fossils/ledger.fossil
$ # populate working tree with project files
$ fossil add .
$ fossil commit -m "Initial import"
```

The repository is `~/fossils/ledger.fossil`, a SQLite file. The working tree is `~/work/ledger`. Subsequent commits go to the repository file via `.fslckout`'s tracking.

To publish, Aditi runs `fossil server ~/fossils/ledger.fossil` (or sets up `fossil server` behind a reverse proxy) on a host her collaborators can reach.

### Operation 2 — Linear development

```
$ vi src/ledger/parser.py
$ fossil commit -m "parser: handle blank lines and comments"
... autosync pushes to the configured remote ...
```

Each commit creates a check-in and (with autosync) immediately syncs to the remote.

### Operation 3 — Branch

Jonas clones:

```
$ fossil clone http://aditi-host/ledger ~/fossils/ledger.fossil
$ mkdir ~/work/ledger && cd ~/work/ledger
$ fossil open ~/fossils/ledger.fossil
```

For the `currency-conversion` work, he creates a branch by committing with a `--branch` flag:

```
$ vi src/ledger/parser.py
$ fossil commit --branch currency-conversion -m "parser: store amounts as (value, currency)"
```

The check-in is tagged `branch=currency-conversion` (replacing the inherited `branch=trunk` from the parent). Subsequent commits in this checkout, until a new branch is named, stay on `currency-conversion`. To switch branches:

```
$ fossil checkout trunk
```

Fossil's branch model is unusual: a branch is the value of a tag on check-ins, not a separate object. There are no "empty branches"; a branch exists when it has at least one check-in.

### Operation 4 — File rename

```
$ fossil mv src/ledger/reports.py src/ledger/report_balance.py
$ # edit report_balance.py
$ fossil add src/ledger/report_register.py
$ fossil commit -m "reports: split into balance and register modules"
```

Renames are first-class. Fossil's web UI shows the file's history across the rename in the timeline.

### Operation 5 — Binary file added

Binaries are added without ceremony:

```
$ fossil add docs/logo.png
$ fossil commit -m "docs: add project logo"
```

Storage is the SQLite blob store with deduplication by content hash and zlib compression on text. Binary content is stored as bytes; if the file changes, both the old and new content live in the database. The deduplication helps for unchanged binaries; the compression does not help for already-compressed formats (PNG, JPEG, etc.).

For very large binary files, Fossil's storage degrades, but more gracefully than Git's because the SQLite engine handles the IO patterns well. Repositories with hundreds of megabytes of binary content are routine; multi-gigabyte cases stress the system.

### Operation 6 — Parallel edits

With autosync on, the parallel-edit case is shorter than in deferred-sync systems. Aditi commits and the commit autosyncs to the server. Jonas's commit, started against Aditi's parent, fails the autosync because it would create a fork:

```
$ fossil commit -m "README: rewrite Overview"
... autosync to server ...
fossil: would fork; pull and merge first
$ fossil pull
$ fossil merge
$ fossil commit -m "Merge Aditi's README changes"
```

The forking case is detected immediately, not at the end of a long offline session.

### Operation 7 — Merge with conflict

Merging the `currency-conversion` branch into trunk:

```
$ fossil checkout trunk
$ fossil merge currency-conversion
... reports conflicts ...
$ # resolve conflicts in working tree
$ fossil commit -m "Merge currency-conversion into trunk; resolve parser conflict"
```

The merge produces a check-in with two parents.

### Operation 8 — Botched commit

Fossil's stance on history is strong: check-ins are intended to be permanent. Mistakes are corrected forward, not erased. There is `fossil shun`, which marks an artifact as never-to-be-distributed; the artifact stays in the local database (so the SHA-256 chain stays valid) but is not synchronized to other clones, and is hidden from normal display. Shunning is logged; it leaves a record of what was shunned and by whom.

For Mireille's botched commit, she commits a follow-up that removes the debug line:

```
$ vi src/ledger/report_register.py
$ fossil commit -m "register: remove stray debug print from previous check-in"
```

The bad commit stays. The audit trail is preserved. The team's culture, supported by the system's defaults, is the audit-trail one.

For check-ins where the team really wants something gone (an accidental commit of a credential, for instance), `fossil shun` plus rebuild-from-bundle is the path. It is rare; Fossil's design discourages history mutation more strongly than Subversion's does.

### Operation 9 — Tag

```
$ fossil tag add v0.1.0 trunk
```

Tags are first-class. The web UI shows tags on the timeline, and check-ins can be referenced by tag.

### Operation 10 — Release

Fossil has a built-in release workflow via *bundles*. A bundle is a portable package containing a subset of a repository's check-ins, suitable for distribution:

```
$ fossil tarball v0.1.0 ledger-0.1.0.tar.gz
```

`fossil tarball` produces a clean tarball of the checked-out tree at the named revision. For a *bundle* (Fossil's distribution format), `fossil bundle export -branch trunk -from BEGIN -to v0.1.0 ledger.bundle` produces a binary artifact containing the relevant check-ins; another Fossil instance can import it with `fossil bundle import`.

The release artifact, the release notes, and the announcement can all be hosted from the Fossil instance itself. The web UI's "Tarball" link on a tag's page is what the project's website actually shows users.

## Model and mental load

What you have to hold:

- Repository file vs checkout. The repository is one SQLite file; the checkout is a directory tree.
- Autosync. Operations sync immediately by default.
- Branches as tag values on check-ins, not as separate objects.
- Tickets, wiki, forum as part of the repository, synchronized with code.
- Shunning vs forward correction.

The mental load is low. Fossil has fewer commands than Git, fewer concepts to learn, and the integrated UI provides answers to most "what's going on?" questions in a browser. New users are productive in a day. The autosync default protects against a class of forking-and-rebasing problems that DVCS users routinely create for themselves.

## Evolution and history rewriting

Conservative, by design. `fossil shun` for genuine erasure, with logging. `fossil amend` lets the latest check-in's metadata (comment, branch, author) be changed; the change is recorded as a new artifact, not an in-place edit. Historical content is intentionally hard to alter.

## Ecosystem reality

Fossil-the-project is healthy. Releases come on a roughly annual cadence. The mailing list is active. The user base is small. Hipp uses it for SQLite and several other projects; Tcl/Tk uses it; a long tail of small projects uses it. Migration tools exist in both directions: `git-fossil` and `fossil import git` and the equivalents for Mercurial.

There is no Fossil-specific cloud forge in the GitHub mold; there is no Fossil Enterprise consulting market; there is no Fossil ecosystem of CI/CD products. The system is largely self-contained, and the people using it like that.

## When to reach for it; when not to

Reach for Fossil when:

- You are running a small-to-medium project (one developer to a small team) and want code, bugs, wiki, and forum in one artifact, on one server, with one piece of software.
- You want immutable history with a strong cultural default.
- You want autosync semantics: working on stale data is the bug class you want to eliminate.
- You want a tool that runs anywhere, including on resource-constrained hosts, with no external dependencies.

Do not reach for Fossil when:

- The team's contributors expect Git and the cost of training them is high.
- The codebase is large and binary-heavy.
- The hosting story matters: Fossil's UI and ecosystem are minimal compared to GitHub.
- The team needs heavy CI/CD integration: most CI products assume Git.

## Epitaph

Fossil is the version control system that thought "the project is the artifact" was a stronger position than "the source code is the artifact" — and is the smallest, most coherent piece of software in this book.
