# 5. RCS

## Origin

The Revision Control System was written by Walter F. Tichy at Purdue University, with the first public release in 1982 and the design described in his ICSE paper *Design, Implementation, and Evaluation of a Revision Control System* the same year. Tichy framed RCS explicitly as a successor to SCCS: same problem, different storage, different licensing, and a deliberate attempt to fix the parts of SCCS that had aged poorly in a decade of use.

The key technical change was the storage format. SCCS used interleaved deltas; RCS used *reverse deltas*. The latest revision on the trunk is stored in full; each older revision is stored as a unified-diff-style patch that, applied in reverse to its successor, reproduces the older state. Branches store forward deltas from their branch point. The practical consequence is that retrieving the head of the trunk costs nothing — it is the file as it appears in the history file — and retrieving older or branch revisions costs proportionally to how many deltas have to be applied. For a codebase whose dominant access pattern is "give me the current version," reverse deltas are faster than interleaved.

The other key change was licensing. RCS was free software in the sense that mattered: redistributable, modifiable, includable in other distributions. SCCS was AT&T property and remained so until parts of it eventually emigrated through the various BSD lawsuits and Solaris open-sourcing efforts. For a university or a research lab that wanted to use a version control system in the 1980s, RCS was the option that did not require AT&T's lawyers to bless your operating environment.

RCS continues to ship with most Unix-like systems in 2026. It is a GNU project, currently maintained at a low pace by Thien-Thi Nguyen and contributors. New projects do not adopt it. Old single-file scripts and configuration files often still live under it; the `,v` suffix on a config file is the most common sign of a sysadmin who learned their craft in the 1990s and has not seen a reason to change.

## The system on its own terms

RCS, like SCCS, is per-file. The history of `parser.py` lives in a single text file named `parser.py,v` (the `,v` extension is universal, sometimes kept in a sibling `RCS/` directory, sometimes alongside the source). The history file contains a header with metadata and a body containing the latest revision's full text plus reverse deltas for all earlier revisions. Branches are nested under their branch points; the format is documented in `rcsfile(5)` on most systems and is straightforward to read with a text editor.

Revision numbers are dotted, in the same shape as SCCS SIDs but with different conventions. `1.1` is the first revision; `1.2`, `1.3` follow. A branch off `1.2` produces revisions `1.2.1.1`, `1.2.1.2`, and so on. Tagged names ("symbolic names" in RCS terminology) are mapped to revision numbers in the history-file header: `release-0.1.0:1.7` is a typical line.

The locking model is per-file and pessimistic, like SCCS, but with a useful escape hatch. `co -l file` checks the file out for editing and acquires the lock. `co file` checks it out read-only. `co -u file` checks it out, prompts for confirmation if there are local changes, and unlocks. `rcs -u file` breaks a lock held by another user — a privileged operation, but a normal one in practice. The project's `rcs` configuration can also enable *strict* locking (the default for shared files) or non-strict (where the file owner can commit without first holding the lock).

The command surface is small. `ci` checks in (commits). `co` checks out. `rcs` is the administrative command for symbolic names, locking modes, mode changes, and metadata edits. `rcsdiff` shows differences between revisions. `rlog` shows revision history. `rcsmerge` performs a three-way merge between two revisions of a file and a common ancestor. `ident` extracts keyword strings from a file. That is essentially the whole CLI.

## Scenario walkthrough

The exemplar is recognizable in RCS, with the same fundamental constraint as SCCS — per-file history — but with somewhat better tools to work with it.

### Operation 1 — Initial import

Aditi creates the project on her laptop. To bring it under RCS:

```
$ mkdir RCS
$ ci -t-"Initial import" -u README.md
RCS/README.md,v  <--  README.md
initial revision: 1.1
done
$ ci -t-"Initial import" -u parser.py
... (and so on for every file)
```

The `-t-` flag attaches a description to the file (RCS distinguishes between the per-file *description*, which is a one-time metadata field, and per-revision *log messages*). `-u` keeps the working copy after check-in, unlocked. Without `-u`, `ci` deletes the working copy after committing — a behavior that has surprised every RCS user at least once.

There is no concept of a multi-file commit. Each `ci` is independent. Like SCCS, RCS users in practice maintain a separate `ChangeLog` file under RCS, updated by hand on every set of related changes, to serve as the project-level history.

### Operation 2 — Linear development

For the second commit:

```
$ co -l parser.py
RCS/parser.py,v  -->  parser.py
revision 1.1 (locked)
done
$ vi parser.py
$ ci -u parser.py
RCS/parser.py,v  <--  parser.py
new revision: 1.2; previous revision: 1.1
enter log message, terminated with single '.' or end of file:
>> parser: handle blank lines and comments
>> .
done
```

The same dance for the next two commits. Each is per-file. The `ChangeLog` is updated alongside, in its own check-in cycle.

### Operation 3 — Branch

Aditi shares the repository with Jonas via NFS or by emailing the `RCS/` directory. Jonas's `currency-conversion` work creates per-file branches. The branch is created when Jonas checks out an older revision and commits against it:

```
$ co -l -r1.4 parser.py
RCS/parser.py,v  -->  parser.py
revision 1.4 (locked)
done
$ vi parser.py
$ ci -u -r1.4.1 parser.py
RCS/parser.py,v  <--  parser.py
new revision: 1.4.1.1; previous revision: 1.4
... message ...
```

After the first branch commit, subsequent commits on the same file's branch increment the last component: `1.4.1.2`, `1.4.1.3`. Jonas does this on `parser.py` and `reports.py`. The "branch" is two file-level branches that he and the team agree to think of as one logical branch. The agreement is maintained in commit messages and the `ChangeLog`.

### Operation 4 — File rename

RCS, like SCCS, has no native rename. To rename `reports.py` to `report_balance.py`, Aditi moves the history file:

```
$ mv RCS/reports.py,v RCS/report_balance.py,v
$ co -l report_balance.py    # produces report_balance.py from new history
$ # edit to remove the register() function
$ ci -u report_balance.py
... message: "reports: split into balance and register modules" ...
$ ci -t-"register report" -u report_register.py
... initial revision: 1.1 ...
```

The history of `report_balance.py` extends back through the rename. The history of `report_register.py` starts fresh. There is no link between them in the version control system; the `ChangeLog` and the commit messages are the only record that the split happened.

The honesty problem from SCCS recurs here: the history file's internal *Phrase* field can record the original filename. RCS does not surface this in normal output. `rlog report_balance.py,v` shows the history under the new name as if the name had always been new.

### Operation 5 — Binary file added

Mireille's logo PNG is added with `ci -i logo.png` (or `ci -u logo.png` if she wants to keep the working copy). Critically, before any edits, she runs:

```
$ rcs -kb logo.png
```

The `-kb` flag sets the *keyword expansion mode* to binary, which disables the otherwise-default behavior of expanding `$Id$`, `$Header$`, and other keywords inside the file. Without `-kb`, RCS would corrupt the binary by trying to expand keyword strings in the middle of compressed image data.

Storage cost is similar to SCCS for binaries: each revision is essentially a full copy because reverse deltas of binary data do not compress meaningfully. Teams in the RCS era kept binaries out of version control by convention; the exemplar puts the logo in to demonstrate the cost.

### Operation 6 — Parallel edits to the same file

The locking model means parallel edits are again prevented by default. With non-strict locking, the file owner can edit without acquiring the lock first; with strict locking (the default for shared files), the lock is required.

The conventional workaround in the RCS era was for both editors to take *unlocked* checkouts, edit independently, and use `rcsmerge` to reconcile their changes:

```
$ co -p -r1.6 README.md > /tmp/aditi-base.md     # common ancestor
$ # Aditi has produced README.md with her edits
$ # Jonas has produced /tmp/jonas-version.md with his edits
$ rcsmerge -p -r1.6 -r1.7 README.md > /tmp/merged.md
```

This works for text. The output of `rcsmerge` includes the standard `<<<<<<<` `=======` `>>>>>>>` conflict markers when the merge cannot proceed automatically — a convention introduced by RCS that survives in every system since.

For the exemplar, the parallel edits to `README.md` are non-overlapping and `rcsmerge` resolves them automatically. The merge is committed as a normal `ci` on the merged file.

### Operation 7 — Merge with conflict

For the `parser.py` conflict, Jonas runs `rcsmerge` against the branch tip and the trunk tip with the branch point as common ancestor:

```
$ co -l parser.py     # gets trunk tip 1.6
$ rcsmerge -r1.4 -r1.4.1.2 parser.py
RCS file: RCS/parser.py,v
retrieving revision 1.4
retrieving revision 1.4.1.2
Merging differences between 1.4 and 1.4.1.2 into parser.py
rcsmerge: warning: conflicts during merge
$ vi parser.py     # resolve <<<<<<< markers
$ ci -u parser.py
... message: "Merge currency-conversion into trunk" ...
```

The result is revision `1.7`, which has *one parent* in the RCS history (revision `1.6`). The branch revision `1.4.1.2` is not recorded as a parent of `1.7`. The merge is in the comment, not in the structure. This is the same constraint as SCCS, with a better tool to perform the merge mechanically.

### Operation 8 — Botched commit needing rewrite

Mireille's botched check-in is correctable in two ways. To remove the bad revision entirely (only allowed for the latest on a branch, only if it has no symbolic names attached):

```
$ rcs -o2.3 RCS/report_register.py,v
RCS/report_register.py,v  <--  report_register.py
deleting revision 2.3
done
```

The `-o` flag *outdates* (removes) the specified revision. Subsequent revisions are not renumbered; her next `ci` will produce 2.4, jumping over the deleted 2.3. The history file no longer contains 2.3.

To change the *log message* without removing the revision:

```
$ rcs -m2.3:"register: add --csv flag" RCS/report_register.py,v
```

For the exemplar, Mireille uses `-o` to remove the bad revision (with the debug print), re-does the work cleanly, and commits as a fresh revision with the correct message. The botched revision is gone from history; nobody else has seen it because she has not shared since the bad commit.

### Operation 9 — Tag

RCS has *symbolic names*, which are first-class enough to be useful. Aditi tags version 0.1.0 with:

```
$ for f in RCS/*,v ; do rcs -nv0.1.0:$(rlog -h $f | awk '/^head:/{print $2}') $f ; done
```

The `-n` flag attaches a symbolic name to a revision. The shell loop iterates over every history file and tags the head revision of each. The tag is per-file, not per-project; the team's convention is that the same name across all files constitutes a project tag.

Tags can be moved (`rcs -N` to override) or deleted (`rcs -n NAME` with no revision). They appear in `rlog` output and can be used in `co -r NAME` to retrieve the tagged revision.

### Operation 10 — Release

The release tarball is generated from a clean working tree at the tag:

```
$ mkdir /tmp/release && cd /tmp/release
$ for f in /path/to/project/RCS/*,v ; do co -rv0.1.0 $f ; done
$ tar czf ledger-0.1.0.tar.gz *
```

There is no release object in RCS. Mireille writes release notes by hand into a `CHANGELOG` file under RCS and updates it with a normal check-in cycle. The tarball is published out of band.

## Model and mental load

RCS adds a few things to SCCS's mental load and removes a few. The dotted revision numbers are similar; the locking model is similar; the per-file model is identical. What is different:

- The reverse-delta format means the head is "free" to retrieve and old revisions cost proportionally to their depth. Practically, this means `co` of the current version is fast and `co -r1.1` of a long-history file is slow.
- The keyword expansion is more elaborate than SCCS's. `$Id$` expands to a one-line summary including the filename, revision, date, author, and state. `$Log$` expands to the entire log history embedded in the file, which is a perennial source of merge conflicts (because two branches both grow the embedded log) and is one of the things later systems explicitly avoided.
- Symbolic names are easier than SCCS's release/level convention.
- `rcsmerge` is a real merge tool, in a way SCCS provided nothing equivalent to.

The remaining mental load is the per-file model. Every operation is one file. Every project-level concept — multi-file commits, branches, tags — is an ad-hoc convention layered on top.

## Evolution and history rewriting

`rcs -o REV` removes a revision (the latest on a branch only). `rcs -m REV:msg` rewrites a log message. `rcs -e USER` removes a user from the access list. These are surgical, audit-aware operations. Bulk history rewriting was not a feature; teams that needed it invoked perl on the `,v` files, which worked because the format is documented and approachable.

RCS's design has not evolved in any user-visible way since the 1990s. Maintenance has continued; features have not.

## Ecosystem reality

In 2026, RCS shows up:

- On systems administrators' laptops, used to track configuration files in `/etc`. The pattern is older than `etckeeper` and persists for the same reasons: a `co -l /etc/resolv.conf` is faster to type than initializing a Git repository, and the user only cares about that one file.
- In the corner of academic codebases that started in the 1980s and never migrated.
- As a building block. CVS is RCS-over-network; many of CVS's behaviors are RCS's behaviors with multi-user coordination layered on. Reading the CVS chapter without having read this one would lose context.

The reference implementation is GNU RCS. Most BSDs ship OpenRCS, a permissively-licensed re-implementation. Both read and write the same `,v` format.

## When to reach for it; when not to

Use RCS in 2026 in two cases. First, if you are tracking a small set of single-file configurations and the entire mental model of "this is one file with versions" is what you want; the overhead of a directory-level system is genuine, and a single `,v` file beside a config is honestly less work than `git init`. Second, if you are reading historical material whose primary record is in `,v` files and you want to see what the original users saw.

For new projects of any kind larger than a single file, reach for something else. The per-file model that worked when projects were a directory of self-contained Unix utilities does not match how anyone organizes work in 2026.

## Epitaph

RCS is the system that decided what a `<<<<<<<` conflict marker looks like, and the rest of the field has been arguing about everything except that ever since.
