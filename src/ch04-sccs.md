# 4. SCCS

## Origin

SCCS — the Source Code Control System — was written by Marc J. Rochkind at Bell Labs in 1972. The original implementation was in SNOBOL4 and ran on IBM System/370 mainframes under OS/MVT; the C reimplementation that became canonical arrived a year or two later, after the team migrated to Unix. Rochkind's 1975 paper *The Source Code Control System* in *IEEE Transactions on Software Engineering* describes the design and motivation in clear, calm prose; readers who want primary sources should start there.

The motivation Rochkind described was almost entirely about *change tracking* in a regulated, audited engineering environment. Bell Labs was AT&T, and AT&T was a regulated monopoly whose source code was, in some literal sense, the property of telephone-system rate-payers. Knowing who changed what, when, and why was not a developer convenience; it was paperwork the company had to keep. SCCS's design reflects this: every change is timestamped, attributed, and accompanied by a Modification Request (MR) number and a comment, and the resulting history is, by intent, immutable in normal use.

The other motivation was disk space. In 1972, the cost of storing every revision of every file as a complete copy was prohibitive. SCCS's contribution was a clever encoding — the *interleaved delta*, sometimes called the "weave" — that stored all versions of a file in a single, linearly-readable text file. The encoding made any version reconstructable in time roughly proportional to the file size, which was the figure that mattered when the bottleneck was tape I/O. Fifty-four years later, SCCS's storage format remains an unusually elegant answer to that constraint, and it is one of the ideas the field walked away from too quickly.

SCCS is still a POSIX-standard utility. It is included in IBM AIX, Solaris (where the GNU CSSC reimplementation also runs), and several BSDs. It is not used for new projects in 2026. It is used to read history that nobody has migrated, in places where nobody is willing to migrate it.

## The system on its own terms

The SCCS storage unit is a single file per source file, prefixed with `s.`. If your source is `parser.py`, the SCCS history file is `s.parser.py`, kept in a sibling directory named `SCCS/` (the convention is universal but not strictly enforced). The history file is plain text, structured as a header containing per-revision metadata followed by the body, which interleaves all versions' lines with markers indicating which revisions each line belongs to.

A revision is identified by a *SID* — a SCCS Identifier — in dotted form: `1.1` is the first revision, `1.2` the second on the trunk, `1.3.1.1` the first revision of the first branch off `1.3`. The format permits four levels of nesting (release, level, branch, sequence). Tags are keywords, not first-class, and the system maintains them through string substitution at extraction time.

SCCS's command surface is a family of small programs, originally invoked as `admin`, `get`, `delta`, `prs`, `prt`, `rmdel`, `cdc`, `sact`, `unget`, and `val`. Later, a wrapper `sccs` was added that subsumes them: `sccs create`, `sccs edit`, `sccs commit`, `sccs info`, `sccs diffs`, and so on. Commands operate per-file. There is no notion of *project* in SCCS at the system level. A project is whatever directory contains a `SCCS/` subdirectory; the structure of the project is whatever the team agrees it is.

The locking model is pessimistic and per-file. To edit, you run `get -e`; this creates a writable working copy from the history file and creates a *p.file* (prefix `p.`) recording that the file is checked out. Until you run `delta` (which records the changes back into the history file and removes the lock) or `unget` (which discards them), nobody else can edit the same file. There is no notion of an exclusive lock per *user*; a single user holding the lock is the model.

There is also no networked operation. The history file lives on a filesystem; sharing history means sharing the filesystem. In practice, teams used NFS-mounted SCCS directories, or copied `SCCS/` trees around, or used tools like RCS-on-NFS conventions to coordinate. The version control system stopped at the filesystem boundary, and the question of distribution was the team's problem.

## Scenario walkthrough

The exemplar from Chapter 3 stresses SCCS in instructive ways. Some operations are natural and clean; some require the team to invent conventions on top of the system.

### Operation 1 — Initial import

Aditi creates the project on her laptop, with the file layout from Chapter 3. To bring the files under SCCS, she creates a `SCCS/` directory and runs `admin -i` for each file:

```
$ mkdir SCCS
$ admin -ifile1 -aaditi s.README.md
$ admin -iLICENSE -aaditi s.LICENSE
$ admin -ipyproject.toml -aaditi s.pyproject.toml
... and so on for every file
```

The `-i` flag specifies the file to import; `-a` adds an authorized user. The result is one `s.*` file per source file. If she wants to see the working copy back, she runs `get s.parser.py`, producing `parser.py` as a read-only file with SCCS keywords expanded. To start editing, she runs `get -e s.parser.py`.

There is no commit message at the import. SCCS records the comment "date and time created" automatically. There is no notion of a multi-file commit; each `s.` file is an independent history. This is the first place the SCCS chapter has to admit that the *unit of history* in SCCS is a file, not a project, and the consequences echo through the rest of the operations.

To approximate a project-level history, teams used a *changelog file* under SCCS: a plain text file (often named `ChangeLog`) updated by hand on each change, recording the cross-file relationship. The convention is older than SCCS; it predates Unix. For the exemplar, Aditi creates a `ChangeLog` file as part of the import and updates it on every subsequent change.

### Operation 2 — Linear development

To make the second commit (`parser: handle blank lines and comments`), Aditi runs:

```
$ get -e s.parser.py
1.1
new delta 1.2
26 lines
$ vi parser.py
$ delta s.parser.py
comments? parser: handle blank lines and comments
1.2
3 inserted
0 deleted
23 unchanged
```

The same dance plays out for the other two commits. The only file changing in each is the file changing; SCCS does not record that the second and third commits are related to the first. The `ChangeLog` file is the only artifact that connects them, and updating it requires its own `get -e` / `delta` cycle.

### Operation 3 — Branch

There are two senses of "branch" relevant here. SCCS supports *branches* in its SID space: `get -e -b s.parser.py -r1.4` creates a branch off revision 1.4, and the next delta gets SID `1.4.1.1`. This is per-file branching, and the team must coordinate the branch across files manually. There is no system-level branch object.

The other sense — Aditi sharing the repository with Jonas so Jonas can develop against it — is not really an SCCS operation. It is an operating-system operation: Aditi gives Jonas access to the directory, by copying it, by mounting it over NFS, by tarring it up and emailing it. The exemplar takes the NFS approach: Aditi's `SCCS/` lives on a shared server, Jonas mounts it, and the two of them coordinate with the locking that SCCS provides per file.

For Jonas's `currency-conversion` work, he creates per-file branches:

```
$ get -e -b s.parser.py -r1.4
1.4
new delta 1.4.1.1
54 lines
```

He edits, runs `delta`, gets SID `1.4.1.1`. He repeats the same for `s.reports.py`. He has a *currency-conversion branch*, but only in the sense that two specific files have new branch SIDs corresponding to his work. Coordinating the branch concept across the project is the team's responsibility, and is typically maintained in commit messages, in the `ChangeLog`, and in the human heads of the people doing the work.

### Operation 4 — File rename

SCCS has no native rename. To rename `reports.py` to `report_balance.py`, Aditi has two options. The *honest* option is to move the SCCS history file: `mv SCCS/s.reports.py SCCS/s.report_balance.py`, which preserves the file's history under the new name. The *clean* option is to use `admin` to import a fresh history file from the latest contents of `reports.py` under the new name and to remove the old `s.` file, breaking the history chain.

The honest option is what most SCCS users did. It works. It also fails in subtle ways: the SCCS history file's internal record of which file it represents (the `M` flag, set with `admin -fm`) does not update unless Aditi explicitly resets it. Some tools (`prs`, `prt`) extract the recorded filename and report against it; if those reports still say `reports.py`, the team's bookkeeping is misaligned with reality.

For the *split* — adding `report_register.py` from the `register()` function — Aditi runs `admin -i` on a new file. The `register()` function's history pre-split is now spread between `s.report_balance.py`'s rename history and a fresh `s.report_register.py` with no prior history at all.

This operation, more than any other in the exemplar, exposes the limits of per-file version control. SCCS does what it does well; what it does does not include "track a function as it moves between files."

### Operation 5 — Binary file added

Mireille's logo, a 24 KB PNG, is added with `admin -ilogo.png -amireille s.logo.png`. SCCS's text-oriented storage is not gracious about binary content. The file is stored as the binary bytes wrapped in the SCCS file format; subsequent revisions store full deltas (because the interleaved-delta encoding cannot meaningfully diff binary), and any revisions to the file balloon the history file to multiples of the source size.

In practice, teams kept binaries out of SCCS or used an out-of-band store for them. The exemplar puts the binary into SCCS to demonstrate the cost. The first revision of `s.logo.png` is roughly 32 KB (24 KB of payload plus SCCS overhead). A second revision, even if only one pixel changes, would add another 24 KB.

There is no `+l` flag or filetype mechanism here as there is in Perforce. The binary is a text file as far as SCCS is concerned, and the team's convention is the only thing keeping the storage from degrading.

### Operation 6 — Parallel edits to the same file

The locking model means parallel edits to the same file *cannot happen* in SCCS. If Aditi runs `get -e s.README.md`, the `p.README.md` file records her lock, and Jonas's `get -e` will fail until she runs `delta` or `unget`. The two are forced to serialize.

This is not a defect; it is a design choice. SCCS encoded the assumption that the way to handle conflict is to prevent it. For text under active edit by two people, the assumption was wrong, and the entire next phase of version control history (RCS, CVS, the optimistic concurrency model) is the field's response to learning it was wrong.

For the exemplar, Aditi edits the README first; Jonas waits, then edits afterward, with full visibility of her changes. The "parallel" part of Operation 6 cannot be staged. The chapter notes the cost: every multi-file commit in SCCS-land is implicitly a coordination meeting, because the locks have to be acquired in order, and a held lock is a person blocking another person's work.

### Operation 7 — Merge with conflict

Without parallel edits, there is no conflict to resolve in the README. The `parser.py` conflict between Jonas's branch (`1.4.1.1`) and Aditi's main-line work needs explicit reconciliation when the branch is merged back.

SCCS does not perform merges. The conventional approach is:

```
$ get -r1.4.1.2 s.parser.py     # extract branch tip
$ cp parser.py /tmp/branch-version
$ get -e -r1.6 s.parser.py      # check out main-line tip for editing
$ # manually merge /tmp/branch-version into parser.py using diff and editor
$ delta s.parser.py
comments? Merge currency-conversion branch into trunk
1.7
```

The merge is a hand operation. There is no record in the new revision that says "this is a merge of 1.6 and 1.4.1.2"; only the comment encodes the intent. SCCS does not have merge commits as a concept; it has revisions, each with one parent, and the human has to remember which revision was the merge.

### Operation 8 — Botched commit needing rewrite

Mireille's botched commit — the stray `print("DEBUG:", row)` and the typoed message — is correctable in SCCS, but the correction has limits.

To fix the *comment*, she runs `cdc -r2.3 s.report_register.py`:

```
$ cdc -r2.3 s.report_register.py
new comments?
register: add --csv flag
```

The `cdc` command (Change Delta Commentary) updates the comment for an existing delta. The original is preserved in the history; the new comment is added.

To fix the *content* — to remove the stray debug line — she has more options. If the bad delta is the most recent on its branch, she can run `rmdel` to remove it entirely:

```
$ rmdel -r2.3 s.report_register.py
```

This *erases* the delta from history. Subsequent SIDs still increment from the prior delta; the bad change is gone. SCCS allows this only for the latest revision on a branch, and only if no subsequent deltas reference it.

If the bad delta is not the latest, the correction is a *new* delta that undoes the stray change. SCCS history then contains both the bad delta and the correction, in sequence, with no facility for collapsing them.

Mireille catches the mistake immediately, before any other changes, so `rmdel` works. She removes the bad delta and re-commits with the correct content and comment.

This is the SCCS chapter's first concrete touch on the *history mutability* axis. SCCS is conservative: it allows targeted correction but does not allow the rewriting of arbitrary spans. The audit trail wins by default.

### Operation 9 — Tag

SCCS has no first-class tag object. Tags are *named symbols* maintained through keyword substitution. To tag the project at version 0.1.0, Aditi runs `admin -fr` on each file to set the *release* component of the SID, and the tag is recorded in the history-file header.

In practice, teams maintained a separate convention: a `RELEASES` file under SCCS recording the tag name and the SID of each file at the moment of tag, or a tarball of `SCCS/` taken at the moment of tag. The exemplar takes the second approach: Aditi creates a tarball `ledger-sccs-v0.1.0.tar.gz` from the SCCS directory and the working copy at the tag point.

### Operation 10 — Release

The release artifact is the tarball generated from the tagged working copy. There is no release object in SCCS; the team writes a shell script that does `get` for every `s.` file, producing the working copy, and tars it up. Mireille writes the release notes by hand into a `CHANGELOG` file under SCCS, increments the SID with a delta, and runs the tarball script.

Publication is out of band: an email to the project list, or an FTP upload, or whatever the team has agreed. None of this is in the version control system.

## Model and mental load

To use SCCS, you have to hold:

- A per-file mental model. The project is files; the version is per file; coordination is your problem.
- The lock state. `sact s.foo` tells you who holds the lock on `foo` and at what SID; running it before every edit is a habit.
- The SID convention. `1.4.1.1` is a branch; `1.4` is a trunk revision; the next delta after `1.4.1.1` is `1.4.1.2`. Confused parsing here is a common source of mistakes.
- The keyword substitution rules, if you use them. `%I%`, `%W%`, `%M%` expand at `get` time and stay expanded in the working copy. Re-expanding them later requires running `get` again, which discards uncommitted edits.

The mental load is high *for someone in 2026*. For its original users, the load was low: the system was a small set of orthogonal commands, and the per-file model was native to how Bell Labs organized work. A team that had used SCCS for two years could move through it as fluidly as a Git user moves through `git status` and `git commit`. The complaint that SCCS is unintuitive is partly a complaint that it does not match the user's existing model.

## Evolution and history rewriting

SCCS allowed `rmdel` for the latest delta on a branch, `cdc` for comment changes, and the `admin` command's various flags for metadata changes. It did not allow arbitrary history rewriting; it did not allow history erasure. The audit trail was the point.

The system stopped evolving in any meaningful sense by the late 1980s. RCS replaced it for most users, and CVS replaced RCS for users who needed multi-user coordination. Sun shipped SCCS on Solaris into the 2010s; IBM still ships it on AIX. The GNU CSSC project maintains a free reimplementation that reads and writes the same s-files.

## Ecosystem reality

In 2026, SCCS is found in three places. Long-running enterprise codebases at telecom and aerospace companies that have not migrated. POSIX environments where developers occasionally hit a `s.` file in the corner of a vendor tree. And historical archives — Bell Labs, Plan 9, early BSD — where the s-files are the primary record of who did what when.

CSSC (Compatibly Stupid Source Control, an early-internet humor name) is the GNU drop-in replacement and reads the original format faithfully. Tools like `cvs2git` and `cvs2svn` have SCCS readers (sometimes via an intermediate CVS step). Migration is mechanical and well-trodden.

## When to reach for it; when not to

You do not reach for SCCS in 2026. You may *encounter* it, when you inherit a codebase whose history is in s-files. In that case, your job is to read the history, not to use the system. CSSC will read the s-files; you can extract the historical state of any file at any SID, and you can usually reconstruct enough of the project's history to migrate it to whatever modern system you are moving to.

The only reason to keep using SCCS today is to maintain compatibility with a tool or process that requires its specific behaviors — and even those cases usually have viable migrations.

## Epitaph

SCCS was the first version control system that worked, and its weave format is the most elegant thing in this book that nobody copied.
