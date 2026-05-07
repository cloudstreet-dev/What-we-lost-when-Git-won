# 7. Subversion

## Origin

Subversion was conceived in 2000 at CollabNet by Karl Fogel, Jim Blandy, and Brian Behlendorf with the explicit ambition of replacing CVS. The design team — which over the project's first five years included Greg Stein, Ben Collins-Sussman, Mike Pilato, Karl Fogel, Jim Blandy, Greg Hudson, Sander Striker, and others — set out to fix what was visibly broken in CVS while keeping enough familiarity that CVS users could move with a week of relearning. The 1.0 release shipped in February 2004; the project was donated to the Apache Software Foundation in 2009 and is still maintained there as Apache Subversion.

The fix list, drawn directly from CVS pain points, defined the design:

- *Atomic commits.* A commit either fully succeeds or fully fails; the repository's revision number increments only on success.
- *Repository-wide revision numbers.* `r1234` names a state of the entire tree. Every commit advances a single integer; every revision is a globally meaningful name.
- *Real rename and copy.* `svn mv` and `svn cp` are first-class; the resulting history records that a file was renamed or copied.
- *Cheap branches and tags as directory copies.* The internal storage uses copy-on-write, so creating a branch or tag is O(1) regardless of repository size.
- *Versioned metadata.* Properties (`svn:eol-style`, `svn:executable`, `svn:mime-type`, `svn:externals`, `svn:ignore`) are versioned alongside content.
- *Better network protocol.* The default transport is HTTP(S) via Apache `mod_dav_svn` (WebDAV), with `svnserve` as a lightweight alternative.
- *Working copies as a complete unit.* The working copy can be reconstructed entirely from `.svn/` metadata; no `,v` files leak into developer view.

The system shipped with these properties intact. The 1.0 launch was a serious event in the open-source world; *Version Control with Subversion* by Pilato, Collins-Sussman, and Fogel (the "SVN book," available free under a Creative Commons license) became the canonical reference. The migration from CVS over the next four years was the largest-scale tooling migration the open-source world had performed up to that point.

Subversion's later trajectory is the one DVCS-inflected developers tend to remember imperfectly. It did not stand still. Merge tracking arrived in 1.5 (2008); FSFS-based repositories replaced Berkeley DB as the default storage; the working copy format was rewritten in 1.7 (2011) to centralize the `.svn/` directory at the working copy root rather than scattering one per subdirectory; HTTP/2 support, performance improvements, and operational fixes have continued. The system in 2026 is materially better than the one most ex-Subversion users remember from 2008.

## The system on its own terms

Subversion is centralized. There is one repository, on one server, holding all history. Working copies are sparse, ordered views into one branch (one path within the repository, in Subversion's idiom) at one revision. The basic mental model is: the repository is a versioned filesystem, and a working copy is a checkout at a particular path and revision.

The repository contains every revision of every file ever committed, plus per-revision properties (commit message, author, timestamp). The internal storage is FSFS by default — a directory tree under `db/revs/` and `db/revprops/` containing per-revision packs. The format is documented and can be repaired with `svnadmin recover`, dumped with `svnadmin dump` for transport, and loaded with `svnadmin load`.

Branches and tags are not first-class concepts; they are *conventions* of where things live in the repository tree. The canonical layout is:

```
ledger/
├── trunk/
├── branches/
│   └── currency-conversion/
└── tags/
    └── v0.1.0/
```

A branch is a directory copy of `trunk/` into `branches/`. Internally, the copy is cheap (CoW on the storage backend). A tag is the same — a directory copy into `tags/` — by convention not modified after creation. The fact that branches and tags share a mechanism is elegant from the storage side and a perennial source of discipline issues from the workflow side: nothing prevents a developer from committing to a tag, and `svn:author` hooks are the only enforcement.

Revision numbers are repository-wide. `r4` is the fourth commit to the repository; if the commit added a file in `trunk/src/parser.py`, then `parser.py` at `r4` shows up at that path; at `r3` it does not yet exist. This unifies the naming question across the repository: any revision number names the *entire* repository's state at that moment.

The version-numbering integers do not always correspond to file change counts. A commit that touches `branches/foo/` advances the global counter; a query like `svn log -v trunk/` skips revisions that did not affect `trunk/`. Users learn to think in terms of revision *ranges* against paths.

Properties are first-class versioned metadata. `svn propset svn:eol-style native foo.py` sets a property on `foo.py`; the property is part of the file's versioned state and changes are tracked. The most important properties:

- `svn:eol-style`: line-ending normalization (`native`, `LF`, `CR`, `CRLF`, `none`).
- `svn:executable`: if set, the file is executable on Unix.
- `svn:mime-type`: a hint for binary handling and merge behavior.
- `svn:keywords`: which keywords are expanded in the file.
- `svn:externals`: external repository references mounted into the working copy.
- `svn:ignore`: directory-level ignore patterns.
- `svn:needs-lock`: marks files that should be lockable; checkout makes them read-only until locked.

Locking is supported as an option. `svn lock foo.bin` acquires a lock; the file becomes editable for the locker and read-only for everyone else. With `svn:needs-lock` set, files are read-only by default and require an explicit lock to edit. This is the option Git later forced binary-heavy projects to retrofit through Git LFS; in Subversion it is built in.

## Scenario walkthrough

### Operation 1 — Initial import

Aditi creates a Subversion repository on a server (or locally for the exemplar):

```
$ svnadmin create /var/svn/ledger
```

She lays out the canonical structure and imports the project tree:

```
$ mkdir -p /tmp/import/{trunk,branches,tags}
$ cp -r /path/to/ledger/* /tmp/import/trunk/
$ svn import /tmp/import file:///var/svn/ledger -m "Initial import"
Adding         /tmp/import/trunk
Adding         /tmp/import/trunk/README.md
... every file added ...
Committed revision 1.
```

The repository is now at revision 1, with `trunk/`, `branches/`, and `tags/` populated (the latter two empty). To work on it, she checks out trunk:

```
$ svn checkout file:///var/svn/ledger/trunk ledger
A    ledger/README.md
... every file checked out ...
Checked out revision 1.
```

### Operation 2 — Linear development

The three subsequent commits:

```
$ vi src/ledger/parser.py
$ svn commit -m "parser: handle blank lines and comments"
Sending        src/ledger/parser.py
Transmitting file data .
Committed revision 2.
```

Each commit advances the global revision number. `svn log src/ledger/parser.py` shows three revisions (1, 2, 3) on the file; `svn log` at the working copy root shows all four revisions touching anything.

### Operation 3 — Branch

Jonas checks out the same repository. To create the `currency-conversion` branch, *he or Aditi* runs:

```
$ svn copy file:///var/svn/ledger/trunk file:///var/svn/ledger/branches/currency-conversion -m "Create currency-conversion branch"
Committed revision 5.
```

The copy operates on the repository directly (URL-to-URL); no working copy is involved. The branch is created at `r5`. To work on the branch, Jonas switches his working copy:

```
$ svn switch file:///var/svn/ledger/branches/currency-conversion
```

Or he checks out a fresh working copy at that path. Subsequent commits on the branch advance the global revision counter (`r6`, `r7`, ...) and are reachable as `branches/currency-conversion@7` and so on.

### Operation 4 — File rename

Subversion has *real* rename:

```
$ svn move src/ledger/reports.py src/ledger/report_balance.py
A         src/ledger/report_balance.py
D         src/ledger/reports.py
$ # edit report_balance.py to remove register()
$ # create report_register.py with register() function
$ svn add src/ledger/report_register.py
A         src/ledger/report_register.py
$ svn commit -m "reports: split into balance and register modules"
Sending        src/ledger/report_balance.py
Adding         src/ledger/report_register.py
Deleting       src/ledger/reports.py
Transmitting file data ..
Committed revision 8.
```

`svn log --verbose src/ledger/report_balance.py` shows the rename in revision 8 (`A /trunk/src/ledger/report_balance.py (from /trunk/src/ledger/reports.py:7)`) and, with `svn log -v --use-merge-history` and a path argument, follows the file's history back through `reports.py`. The history is preserved across the rename, accessible by tools that know to follow copy paths.

### Operation 5 — Binary file added

Mireille adds the logo PNG:

```
$ svn add docs/logo.png
A  (bin)  docs/logo.png
$ svn commit -m "docs: add project logo"
```

Subversion auto-detects binary content and sets `svn:mime-type` appropriately, which suppresses keyword expansion and EOL conversion. For files where auto-detection fails, `svn propset svn:mime-type application/octet-stream` does the job manually. The `auto-props` configuration in the user's `~/.subversion/config` can apply properties by filename pattern.

For very large binary trees, Subversion's storage handles them better than Git's: each revision is stored compactly, and the working copy does not pay a clone cost. Game studios that adopted Subversion in the 2000s often kept binaries directly in it for years before either migrating to Perforce or staying on Subversion permanently.

### Operation 6 — Parallel edits to the same file

Aditi and Jonas both edit `README.md`. The CVS-style optimistic-concurrency rhythm survives:

```
# Aditi commits first
$ svn commit -m "README: add Quick Start"
# Jonas tries
$ svn commit -m "README: rewrite Overview"
svn: E155011: Commit failed (details follow):
svn: E160028: Item '/trunk/README.md' is out of date
$ svn update
G    README.md
Updated to revision 11.
$ svn commit -m "README: rewrite Overview"
Committed revision 12.
```

The `G` means "merged" — the update folded Aditi's changes into Jonas's working copy automatically. Conflicts that cannot be auto-merged appear as `C` and are resolved by editing the file (or by using `svn resolve --accept=...` for file-level resolution policies).

### Operation 7 — Merge with conflict

Jonas merges his branch back to trunk. From a trunk working copy at `r12`:

```
$ svn switch file:///var/svn/ledger/trunk
$ svn merge file:///var/svn/ledger/branches/currency-conversion
... merging ...
C    src/ledger/parser.py
... ...
Summary of conflicts:
  Text conflicts: 1
$ vi src/ledger/parser.py    # resolve markers
$ svn resolve --accept=working src/ledger/parser.py
$ svn commit -m "Merge currency-conversion into trunk; resolve parser conflict"
Committed revision 13.
```

Subversion 1.5 added merge tracking through the `svn:mergeinfo` property, recorded automatically on the directory being merged into. After this merge, `svn propget svn:mergeinfo trunk` shows `/branches/currency-conversion:5-12` — the range of revisions on the branch that have been merged. Subsequent merges re-merge only the new revisions. Reintegrating the branch (`svn merge --reintegrate`, deprecated as of 1.8 but still in idioms) is the formal way to mark "done with this branch."

The merge produces revision 13 on trunk. The branch tip remains at its latest revision. The branch can be deleted with `svn rm branches/currency-conversion -m "Remove merged branch"`, which records the deletion as a normal commit.

### Operation 8 — Botched commit needing rewrite

Subversion's history is, in general, immutable. There is no `--amend`. Mireille's options for the botched commit:

1. **Commit a follow-up.** Standard CVS approach: revert the bad lines, commit with a corrected message. The original commit stays in history.

```
$ vi src/ledger/report_register.py    # remove debug line
$ svn commit -m "register: remove stray debug print (typo in r17 message: --csv not --cvs)"
```

2. **Edit the log message.** With server-side hooks permitting it (`pre-revprop-change` hook), Mireille can rewrite the log message:

```
$ svn propset --revprop -r 17 svn:log "register: add --csv flag"
```

Revision properties are not versioned (this is one of Subversion's accepted asymmetries); changing them mutates the record without leaving a trace. Most production servers configure the pre-revprop-change hook to log who made the change, when, and to what. The change still happens in place.

3. **Dump and reload, surgically.** `svnadmin dump` and `svnadmin load` with `svndumpfilter` can produce an edited history. This is rare; it requires server-side access, breaks every existing checkout's revision references, and is the kind of operation that needs sign-off from the team.

For the exemplar, Mireille does (1) — a follow-up commit — because that is what is normal in Subversion. The bad debug print is in history; the correction is too. Auditors will see both. Reviewers will know what happened.

### Operation 9 — Tag

Aditi tags version 0.1.0 with a directory copy:

```
$ svn copy file:///var/svn/ledger/trunk file:///var/svn/ledger/tags/v0.1.0 -m "Tag v0.1.0"
Committed revision 18.
```

The tag is a directory in `tags/`. Nothing structurally distinguishes it from a branch; convention says you do not commit to it. Hook scripts can enforce the convention server-side.

### Operation 10 — Release

The tarball comes from the tag:

```
$ svn export file:///var/svn/ledger/tags/v0.1.0 ledger-0.1.0
$ tar czf ledger-0.1.0.tar.gz ledger-0.1.0
```

`svn export` produces a clean tree without `.svn/` metadata. Mireille writes the release notes as a final commit on trunk before tagging, or in a `tags/v0.1.0/CHANGELOG.md` after the tag (committing to a tag, which works but violates the convention). The exemplar takes the first approach.

## Model and mental load

What you have to hold:

- The trunk/branches/tags layout. Convention, but pervasive.
- The repository-wide revision number, and the difference between "revision N exists" and "revision N affected this path."
- That branches and tags are directory copies. Both are made the same way; only convention separates them.
- That revisions are immutable; that revision properties are mutable; that the asymmetry is on purpose.
- Properties as first-class versioned data. The `svn:` namespace, the user-defined namespace, and the auto-props configuration.
- That `svn update` can fold remote changes into a modified working copy, possibly producing conflicts.
- That working copies have an internal state (in `.svn/`) that can become inconsistent and require `svn cleanup`.

The mental load is materially simpler than CVS or DVCS. Most developers can be productive in two days. The trunk/branches/tags convention is bureaucratic but discoverable; the global revision number is intuitive in a way Git's hashes are not for newcomers.

## Evolution and history rewriting

History is intentionally immutable. Revision properties (`svn:log`, `svn:author`) can be rewritten with hook permission, but content cannot. The system's stance is the audit-trail one. Mistakes are corrected forward, not erased.

Later evolution has been steady. 1.5's merge tracking, 1.7's working copy rewrite, 1.8's improvements to merge, 1.9's repository performance, 1.10's LZ4 compression, 1.14's various administrative improvements. The pace is unhurried; the project ethos is "be the thing that works for your existing repository." The Subversion in 2026 is more capable than most people who left it in 2010 remember.

## Ecosystem reality

In 2026, Subversion is alive, used, and undramatic. Apache Subversion releases continue. ASF projects use it for some of their infrastructure. FreeBSD migrated off it to Git in 2020; many ASF projects have done the same. But Subversion remains the version control system in:

- A long tail of enterprise codebases — banks, insurance, telecom, defense — where the migration cost has not been worth the perceived benefit.
- Some game studios with binary-heavy trees who prefer it to Perforce on cost grounds.
- Some academic research codebases.
- Sites where the central authority model is desired for organizational reasons.

The hosting story is conservative. CollabNet is gone; Assembla, VisualSVN, Helix TeamHub, and self-hosted Apache servers cover the rest. Tooling has not modernized at the rate Git's has. The ecosystem is mature and quiet.

## When to reach for it; when not to

Reach for Subversion when:

- You want centralized control of code, with strong audit semantics, with no rewriting.
- You work with substantial binary assets and Git LFS feels like the wrong answer.
- You have a team that includes non-developers (designers, technical writers) who benefit from a simpler mental model.
- Your build/release process depends on monotonic global revision numbers.

Do not reach for it when:

- The team uses GitHub-style pull requests as their primary collaboration mode. Subversion's review story is weak by 2026 standards.
- You expect long-running parallel branches with frequent integration. Subversion's merge tracking works but is not as smooth as Git's.
- You expect contributors who already know Git and would resist learning a different model.

## Epitaph

Subversion fixed CVS, fixed it well, and is now the version control system most working programmers cannot remember the name of even when they have a `.svn/` directory in their `~/code/` from a checkout they stopped maintaining in 2014.
