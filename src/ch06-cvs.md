# 6. CVS

## Origin

CVS — the Concurrent Versions System — was started by Dick Grune at the Vrije Universiteit Amsterdam in 1986. The first incarnation was a set of shell scripts wrapping RCS to provide multi-developer coordination on a project Grune was working on with two students, with the explicit goal of letting multiple people edit the same files concurrently and reconciling their changes after the fact rather than serializing through pessimistic locks.

In 1989 Brian Berliner rewrote the whole thing in C, with a small team at Prisma, and the result is what most people who used CVS in its dominant years actually used. Jeff Polk joined the maintenance shortly after. CVS-the-C-program inherited its storage from RCS — `,v` files in a `CVSROOT` directory tree — and added what RCS lacked: a project model, a network protocol, optimistic concurrency, branches and tags as project-level concepts (loosely), and a sane workflow for groups bigger than one.

CVS became the dominant open-source version control system from roughly 1995 through 2005. SourceForge ran on CVS. Mozilla, GNOME, KDE, Apache, FreeBSD, NetBSD, OpenBSD, GCC, X11 — most of the open-source canon spent its formative years on CVS. The system's deficiencies are the reason every later DVCS exists; its successes are the reason a generation of programmers learned what version control was at all.

Maintenance of CVS proper effectively ended in the late 2000s. There is no active development. CVSNT was a Windows-friendly fork that survived longer in commercial settings; it too is essentially dormant. Repositories still exist; many open-source projects took until the 2010s to migrate, and a few academic and enterprise codebases never did.

## The system on its own terms

CVS adds three things to RCS: a network protocol, a project structure, and the optimistic-concurrency workflow.

The *project structure* is a directory tree containing `,v` files arranged to mirror the working-copy layout, plus an administrative directory called `CVSROOT/` containing project metadata: the modules file (which gives logical names to subsets of the tree), the loginfo file (commit hooks), the access control list (in early versions; later mostly handled outside CVS), and a few other small configuration files. A CVS *repository* is the root directory containing the project tree and `CVSROOT/`. The user identifies a repository with a string like `:pserver:user@host:/var/cvs` or `:ext:user@host:/var/cvs` for SSH transport.

The *network protocol* operated initially over `pserver`, a custom protocol with weak password handling that should never have been used over the open internet but was, extensively. Later, the `:ext:` mode wrapped the protocol in SSH, which made it survivable. Locally, the `:local:` mode used direct filesystem access. The protocol is chatty: commands like `cvs update` walk the working copy directory by directory, sending per-file requests. On a slow link or a large tree, this is painful in a way modern VCS users have largely forgotten.

The *optimistic concurrency model* is the design's most consequential contribution. Two developers can edit the same file at the same time. The first to commit succeeds; the second's commit fails because the working copy is out of date, and they must run `cvs update` to merge in the first commit's changes (using the same RCS-derived three-way merge as `rcsmerge`) before re-committing. Conflicts surface as `<<<<<<<` markers in the working copy, to be resolved by hand. The model is, in retrospect, the single most important workflow innovation in the chapter; it is what made multi-developer coordination scale beyond a few people sharing a workspace.

What CVS *did not* fix from RCS is the per-file unit of history. A `cvs commit` of ten files is ten separate RCS check-ins under the hood, sharing a log message but not sharing a transaction boundary. A failed commit could leave half the files updated and half not, with no rollback. This is the famous CVS "non-atomic commit" problem. It is real; it caused real grief; and it is the single largest reason Subversion existed.

The version-numbering model is RCS's: per-file dotted revisions. CVS adds *branches* as named entities (tags whose revision number ends in `.0.N` to signal they are branches), but the underlying revision numbers remain per-file. Tags are also per-file, mapping a name to the revision number of each file at tag time. Both branch and tag operations iterate over every file in the tree, attaching the symbolic name to each `,v` file. This is correct; it is also slow on large trees and produces tag/branch metadata that lives spread across hundreds or thousands of `,v` files rather than in one place.

## Scenario walkthrough

The exemplar is the first one in the book that resembles a modern workflow. The seams are still per-file, but the project model is real.

### Operation 1 — Initial import

Aditi sets up a CVS repository on a server (or her laptop, for purposes of the exemplar):

```
$ cvs -d /var/cvs init
```

This creates the `CVSROOT/` administrative tree. To import the initial project:

```
$ cd ledger
$ cvs -d /var/cvs import -m "Initial import" ledger aditi start
N ledger/README.md
N ledger/LICENSE
N ledger/pyproject.toml
... (every file imported)
N ledger/tests/fixtures/small.journal

No conflicts created by this import
```

`cvs import` takes a *vendor tag* (`aditi`) and a *release tag* (`start`); these are conventions originally for tracking third-party imports through *vendor branches*, but they are required even when you are importing your own work. The import places the files into the repository under the path `ledger/`.

To use the imported tree, Aditi *checks it out fresh*:

```
$ cd /tmp
$ cvs -d /var/cvs checkout ledger
cvs checkout: Updating ledger
U ledger/LICENSE
U ledger/README.md
... (every file checked out)
```

She then deletes her original directory and works from the checkout. This dance — import, then checkout — is one of CVS's surprising rough edges, and is the first thing every CVS-era tutorial had to explain.

### Operation 2 — Linear development

Three more commits, in the standard CVS rhythm:

```
$ vi src/ledger/parser.py
$ cvs commit -m "parser: handle blank lines and comments" src/ledger/parser.py
... or, more commonly, commit everything at once ...
$ cvs commit -m "parser: handle blank lines and comments"
```

The commit operates over whatever modified files are in the working copy by default, or over an explicit list. Behind the scenes, CVS performs a `ci` on each modified `,v` file with the same log message. The commit succeeds or fails per file. If two of three files commit and the third fails (network blip, lock conflict, anything), CVS prints an error and stops; the user is left with two committed files and one not, and there is no rollback.

For the exemplar, Aditi's three commits all succeed cleanly. The non-atomic commit problem will return when it matters.

### Operation 3 — Branch

Aditi gives Jonas access to the CVS repository. He runs `cvs checkout ledger` against the same server URL.

For the `currency-conversion` branch, Jonas runs:

```
$ cvs tag -b currency-conversion
T README.md
T LICENSE
T pyproject.toml
... every file tagged ...
$ cvs update -r currency-conversion
```

`cvs tag -b` creates a branch tag (mapped to revision numbers ending in `.0.N`); `cvs update -r` switches the working copy to the branch. Subsequent commits on the branch produce branch revisions (`1.5.2.1`, `1.5.2.2`, and so on) on each affected file. The branch is a project-wide concept in CVS in a way it was not in RCS; it shows up in `cvs status` and is the first-class thing Aditi can refer to in conversation.

### Operation 4 — File rename

CVS has no rename. The conventional approach is `cvs remove` followed by `cvs add` of the new name:

```
$ cp src/ledger/reports.py src/ledger/report_balance.py
$ # edit report_balance.py to remove register()
$ cat <<EOF > src/ledger/report_register.py
... register() function ...
EOF
$ cvs remove src/ledger/reports.py
$ cvs add src/ledger/report_balance.py src/ledger/report_register.py
$ cvs commit -m "reports: split into balance and register modules"
```

The history of `reports.py` stops; `report_balance.py` and `report_register.py` start fresh. There is no link in the system between them. The commit message is the only record that the rename happened.

The *uglier* honest workaround was to copy the `,v` file directly on the server: `cp /var/cvs/ledger/src/ledger/reports.py,v /var/cvs/ledger/src/ledger/report_balance.py,v`, preserving history but corrupting branch and tag information that mentions the old name. Most teams chose the clean break and accepted the lost history.

### Operation 5 — Binary file added

Mireille adds the logo PNG. CVS's text-orientation breaks binaries by default through keyword expansion and line-ending conversion. The fix:

```
$ cvs add -kb docs/logo.png
$ cvs commit -m "docs: add project logo"
```

The `-kb` flag sets the file's keyword mode to binary, suppressing keyword expansion and keeping the bytes intact. Storage cost is per-file as before: each revision of the logo is essentially a full copy.

CVS's `cvswrappers` file in `CVSROOT/` can be configured to apply `-kb` automatically to files matching certain patterns, which is the convention for projects with many binaries. Setting it up correctly is one of the things CVS users had to learn the second week.

### Operation 6 — Parallel edits to the same file

This is the operation CVS exists for. Aditi and Jonas both edit `README.md`. Aditi commits first:

```
$ cvs update    # nothing new from server
$ cvs commit -m "README: add Quick Start"
```

Jonas commits second:

```
$ cvs commit -m "README: rewrite Overview"
cvs commit: Up-to-date check failed for `README.md'
cvs [commit aborted]: correct above errors first!
$ cvs update README.md
RCS file: /var/cvs/ledger/README.md,v
retrieving revision 1.3
retrieving revision 1.4
Merging differences between 1.3 and 1.4 into README.md
M README.md
$ cvs commit -m "README: rewrite Overview"
```

Because the two edits do not overlap on the same lines, the merge succeeds automatically. Jonas's commit lands as the next revision, with Aditi's changes already incorporated. This is the workflow: try, fail because out of date, update, possibly resolve conflicts, retry. The system trains its users into the rhythm.

### Operation 7 — Merge with conflict

Jonas merges his `currency-conversion` branch back to the trunk. The CVS idiom:

```
$ cvs update -A    # switch back to trunk
$ cvs update -j currency-conversion
... merges branch into working copy ...
RCS file: /var/cvs/ledger/src/ledger/parser.py,v
retrieving revision 1.4
retrieving revision 1.4.2.3
Merging differences between 1.4 and 1.4.2.3 into parser.py
rcsmerge: warning: conflicts during merge
cvs update: conflicts found in src/ledger/parser.py
C src/ledger/parser.py
```

The `-j` flag (for "join") applies the differences between the branch point and the branch tip into the current working copy. Conflict markers appear in the working copy; Jonas resolves them in `parser.py` and `report_balance.py`, and commits the result on the trunk:

```
$ vi src/ledger/parser.py
$ vi src/ledger/report_balance.py
$ cvs commit -m "Merge currency-conversion into trunk; resolve parser conflict"
```

The merge produces *new revisions on the trunk*, with the branch's content folded in. There is no merge commit object; there is no record on the trunk that points at the branch. The branch tag still exists, but the trunk's history shows only "edit revisions, with the merge happening to be one of them." This is the structural sense in which CVS's merge is informational, not relational. To answer "was this branch merged?" you read commit messages or maintain a separate convention.

### Operation 8 — Botched commit needing rewrite

Mireille catches her debug print and typoed message immediately. CVS does not allow rewriting committed history. Her options are:

1. Use `cvs admin -o` (which is `rcs -o` under the hood) to remove the latest revision, server-side, requiring privileged access. This is rare in practice; most CVS deployments did not give developers `cvs admin` rights.
2. Commit a follow-up that removes the debug print, with a corrected message.

She takes the second approach:

```
$ vi src/ledger/report_register.py    # remove the debug line
$ cvs commit -m "register: remove stray debug print (typo in 1.4 message: --csv not --cvs)"
```

The botched commit is *in history forever*. The correction is also in history. The audit trail is preserved; the cost is a noisy log. This is one of the CVS habits that Mercurial and Git later rebelled against.

### Operation 9 — Tag

Tagging the project at version 0.1.0:

```
$ cvs tag v0_1_0
T README.md
T LICENSE
T pyproject.toml
... every file tagged ...
```

Note the convention: CVS tags cannot contain dots or hyphens with the same flexibility as Git tags, so the project naming convention is typically `v0_1_0` rather than `v0.1.0`. The tag is per-file, mapping the symbolic name to each file's current revision. To check out the tagged state:

```
$ cvs checkout -r v0_1_0 ledger
```

### Operation 10 — Release

The release tarball comes from a clean checkout at the tag:

```
$ cvs export -r v0_1_0 -d ledger-0.1.0 ledger
$ tar czf ledger-0.1.0.tar.gz ledger-0.1.0
```

`cvs export` is `cvs checkout` minus the `CVS/` administrative directories — the right thing for a release artifact. Mireille writes the release notes into `CHANGELOG`, commits, and produces the tarball. Publication is out of band.

## Model and mental load

What you have to hold:

- The repository URL, with its `:pserver:`, `:ext:`, or `:local:` prefix. New users get this wrong.
- The `CVS/` administrative directories scattered through your working copy. They are essential; deleting one breaks the working copy.
- The fact that branches and tags are conventions over per-file revision numbers. A branch is real but second-class.
- The non-atomic commit risk. Watch for commit failures mid-tree.
- The keyword expansion rules. Binary files need `-kb`. Text files inherit the project default. Forgetting either corrupts data.
- The `cvswrappers` file, if your project uses it.
- The vendor branch convention, if your project tracks third-party code.

The mental load is moderate, much higher than RCS but lower than Subversion's because the model is more familiar (per-file with project conventions on top). New CVS users could be productive within a week. The complexity that bit them showed up later: branch maintenance, large merges, recovery from non-atomic commits, repository corruption.

## Evolution and history rewriting

Almost none. `cvs admin -o` and `cvs admin -m` (the RCS surgical operations) are available to administrators, rarely to developers. There is no cultural convention of rewriting history; the audit trail is sacred and the noisy log is the cost.

CVS's evolution as a system stalled in the early 2000s. The 1.12 series in 2004 and 2005 added a few operations and bug fixes; after that, development was a slow trickle. The community had moved to Subversion and was beginning to move further to DVCS.

## Ecosystem reality

CVS in 2026 is a legacy concern. The migration tools are mature: `cvs2git`, `cvs2svn`, `parsecvs`, and several proprietary equivalents have been used to move thousands of repositories. They preserve history with various levels of fidelity; most modern projects' Git histories that go back further than 2005 began life in CVS.

Server software is essentially abandoned. Client tools still ship in most distributions. CVSNT lives on in a few enterprise deployments. The most common thing to do with CVS in 2026 is to read it once and migrate.

## When to reach for it; when not to

Do not reach for it. The use cases CVS won are now better served by Subversion (if you want centralized) or any modern DVCS. The rare exception is reading a repository that has not been migrated; for that, install `cvs` and follow a migration guide.

## Epitaph

CVS taught the field that optimistic concurrency works, and made every later system better by being the system everyone wanted to leave.
