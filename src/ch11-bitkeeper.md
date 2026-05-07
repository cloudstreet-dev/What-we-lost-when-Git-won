# 11. BitKeeper (and the Kernel Story, Properly Told)

## Origin

BitKeeper was the first commercial distributed version control system. Larry McVoy founded BitMover in 1997 to build it; the system shipped in 2000. McVoy had spent years thinking about version control at scale — at Sun, at Silicon Graphics, and in long correspondence with kernel developers about what the Linux kernel project needed if it were going to use version control at all. The 1998 paper *The SourceManagement Problem*, circulated by McVoy on linux-kernel and elsewhere, laid out a thesis that became BitKeeper's design: scale comes from distribution, distribution requires per-file changeset histories that can be replicated and merged, and the centralized server model that everything else used was the wrong shape for very large open-source projects.

BitKeeper's storage was per-file in the SCCS lineage — McVoy was an SCCS partisan, and the BK file format was an extended SCCS weave. Atop that storage, BK introduced the *changeset* — a project-level transaction binding per-file deltas into an atomic unit. A clone of a BK repository carried full history of every file plus the changeset graph that wove them together. Two clones could be reconciled with a `bk pull` that fetched missing changesets and merged where the heads diverged. This is the same pattern Mercurial, Git, Bazaar, and Monotone would all later implement, with various degrees of independence in the design.

BitKeeper's commercial model was selective free licensing. Closed-source teams paid; open-source teams could use BK free under a license that restricted reverse engineering, prohibited derivative works, and required participation in BK's hosted public repository service. The license terms were a continuous source of friction with parts of the free-software world; they were also genuine, and BitMover stuck to them.

The system itself was good. Performance was strong; the model was sound; the user experience was usable for people willing to learn it. Several open-source projects adopted it. The most consequential was the Linux kernel, in February 2002.

## The system on its own terms

A BitKeeper repository is a directory containing a `BitKeeper/` subdirectory with metadata and per-file `s.*`-derived history files (extended from the SCCS format with BK's additions). The changeset history is itself a versioned BK file, recording the parent changesets and the per-file deltas comprising each changeset.

Clones are full copies. `bk clone source dest` produces a directory containing every file at every revision. Operations are local: `bk log`, `bk diff`, `bk annotate`, `bk changes` all run against the local clone with no network involvement. Synchronization is two-step: `bk pull` fetches missing changesets from a remote and merges; `bk push` sends local changesets to a remote.

The unit of work is the changeset, identified by a key combining the user, the timestamp, and a hash. BK has notable distinctive features:

- *Per-file SCCS-derived storage* with delta-against-prior-revision encoding. Storage is compact for text; binary files balloon as in any delta system.
- *Changesets as first-class graph objects* with multiple parents on merges. The changeset graph is a DAG.
- *Includes/excludes* for partial pulls. A BK repository can pull a subset of changesets — most importantly, only those affecting a given path or matching a given filter — which means you can have a repository that selectively mirrors part of a larger one.
- *LOD (Lines of Development)* — BK's branching model. A repository has one default LOD; branching creates a new LOD with its own changeset history.
- *Cset filters* and *triggers* — pre- and post-operation hooks for enforcing policy.

The CLI is `bk`, with subcommands. `bk new` opens a file for edit (the file is read-only by default, like Perforce); `bk delta` records the change; `bk commit` creates a changeset binding deltas; `bk pull`, `bk push`, `bk clone`, `bk log`, `bk changes`, `bk csettool` (a graphical changeset browser).

## The kernel story

This is the part of the chapter that has been mythologized in both directions. The accurate version, sourced from the public record — linux-kernel archives, McVoy's blog, Andrew Tridgell's lkml posts, contemporaneous press, and a great deal of Usenet — runs like this.

Linus Torvalds resisted using a version control system for the Linux kernel for over a decade. He had reasons. CVS was, in his view, structurally broken for a project of the kernel's shape. Subversion, when it arrived, did not solve the distribution problem he believed mattered most. Various academic systems (Arch in particular) had the right ideas but were not usable by the people he needed to work with. He maintained the kernel with patches, mailing lists, and a small set of trusted lieutenants who curated and forwarded patches to him.

By 2001, the kernel was big enough and the patch volume high enough that the system was visibly straining. Patches were getting lost. Lieutenants were spending more time triaging than coding. McVoy and Torvalds had been talking about BitKeeper for years; in early 2002, Torvalds tried it. In February 2002 he announced he was using it for kernel development.

The decision was controversial. The kernel was the most prominent free-software project in the world; it had just adopted, as its primary infrastructure, a piece of proprietary software with a license that some prominent free-software figures considered actively hostile. Richard Stallman wrote about it. The BK license's reverse-engineering clauses were specifically pointed to as the problem: kernel developers, as kernel developers, had a professional interest in being able to write tools that talked to BK's wire protocol, and the license forbade certain kinds of analysis required to do so cleanly.

For three years, the arrangement worked. The kernel's development pace accelerated. Andrew Morton began the `-mm` tree (initially as a BK repository) and the patch ecosystem stabilized. Many kernel developers learned BK; some, including significant figures, refused on principle and continued to work via patches, with the BK-using maintainers acting as a bridge.

In April 2005, the situation broke. Andrew Tridgell — best known as the author of Samba, but a long-running figure in the free-software community and someone who had been watching the BK situation with unease for years — wrote a small program that connected to BitKeeper's daemon (`bkd`, listening on port 14690) and extracted metadata. It was not, by most reasonable definitions, *reverse engineering*: it interacted with a publicly-listening service using the protocol the service spoke. Tridgell and others maintained that the interaction was no different from `telnet host 14690` and reading what came out.

McVoy disagreed, sharply, and his disagreement was a position he had stated repeatedly and predictably for years: the BK free license forbade certain kinds of analysis of the BK system, and Tridgell's tool was, in McVoy's reading, exactly the kind of analysis the license forbade. McVoy revoked the free-for-OSS license in early April 2005. Linus and the kernel project had a problem: their current infrastructure was being withdrawn, and the open-source-licensed alternatives all had performance or model problems Linus had previously rejected.

What happened next is the well-known part. Linus took roughly a week to decide what to do, then over the next ten days wrote the first version of Git. The first Git commit is dated April 7, 2005. By the end of April, Git was self-hosting; by mid-May, Linus had moved kernel development onto it. Mercurial — which Matt Mackall had begun working on for the same reason at almost the same moment — was a serious contender in the discussions and went its own way.

The mythology runs in two directions. Some accounts cast Tridgell as the villain who destroyed a productive arrangement out of free-software absolutism; some cast McVoy as the villain who was always going to revoke the license and was waiting for an excuse. Neither account is right.

Tridgell had stated his concerns about a single closed proprietary system controlling a key piece of free-software infrastructure for years; his work on the extraction tool was consistent with his stated position. The tool itself was technically modest. He did not believe he was breaking the BK license; many lawyers reading the license terms agreed. Reasonable people disagreed about the license, and Tridgell, having considered the question, acted on his reading.

McVoy was running a business. The free-for-OSS license was a marketing channel and a goodwill investment, and BitMover's commercial customers paid for the engineering that made BK work. The license terms were ones McVoy believed his business required, and he had told the kernel community as much during the years of discussion before adoption. When the license was, in his view, violated by a high-profile member of the community he had been licensing free, his response was the response he had always said it would be.

The disagreement was real, was rooted in genuinely different priorities, and was not resolvable by either side conceding. Both Tridgell and McVoy have written about the period since; both have, in interviews and writings over the next decade, expressed regret about specific aspects without conceding the central position. The story is *not* one with a villain. It is one of structurally incompatible ideas about how shared infrastructure should work.

What the story produced is what matters for this book. Git exists because BitKeeper was the wrong fit for an open-source project at the moment a different system was needed; Git's design is *visibly shaped* by the BitKeeper experience, with its content-addressed snapshot model an explicit alternative to BK's per-file delta model. Mercurial exists because Matt Mackall, in the same period, had the same problem and reached for the same solution at almost the same moment, with different design choices. The DVCS era's two dominant systems were both born in the BitKeeper-revocation week.

BitKeeper's own coda: McVoy continued to develop and sell BK through the next decade. In 2016, he announced that BitKeeper would be released as open source under the Apache 2 license, primarily because the commercial model was no longer working at scale. He retired BK as a commercial product. The open-source release sits on GitHub today, a usable artifact for anyone willing to install it and learn it. Almost nobody does. The system that made the modern DVCS world possible has been a historical curiosity since the year of its open-source release.

## Scenario walkthrough

For continuity, the exemplar in BK looks like this.

### Operation 1 — Initial import

Aditi sets up a BK repository:

```
$ bk setup -c .bk-config ledger
$ cd ledger
$ # populate with project files
$ bk ci -i .         # check in initial files
$ bk commit -y "Initial import"
```

Or, more idiomatically, `bk import -t plain /path/to/initial/ledger ./` from a fresh repo, which sweeps the directory and imports every file as the first changeset.

### Operation 2 — Linear development

```
$ bk new src/ledger/parser.py    # not needed for existing files; equivalent to checkout for edit
$ bk edit src/ledger/parser.py
$ vi src/ledger/parser.py
$ bk delta -y "parser: handle blank lines and comments" src/ledger/parser.py
$ bk commit -y "parser: handle blank lines and comments"
```

`bk delta` records the change against the file; `bk commit` ties pending deltas together into a changeset. The two-step rhythm distinguishes BK's per-file deltas from project-level changesets.

### Operation 3 — Branch

Aditi shares the repository with Jonas. He clones:

```
$ bk clone bk://aditi-host/ledger ledger
```

For the `currency-conversion` work, Jonas creates an LOD or, more commonly in modern BK usage, simply works in a clone:

```
$ bk clone ledger ledger-cc
$ cd ledger-cc
$ # do the work, multiple commits
$ bk push bk://shared-host/ledger-cc      # publish the branch as its own repository
```

The "branch" in BK culture is often a separate repository, with `bk pull` and `bk push` synchronizing them. This is the same convention early Mercurial used.

### Operation 4 — File rename

```
$ bk mv src/ledger/reports.py src/ledger/report_balance.py
$ bk new src/ledger/report_register.py
$ # edit files
$ bk commit -y "reports: split into balance and register modules"
```

Renames are first-class. `bk log` follows the file across the rename.

### Operation 5 — Binary file added

BK handles binaries via file flags. To add the logo:

```
$ bk new -bb docs/logo.png    # -bb for binary
$ bk commit -y "docs: add project logo"
```

Storage is full copies for binaries; BK has no equivalent of Perforce's `+l` exclusive lock by default, though triggers and policies can implement one.

### Operation 6 — Parallel edits to the same file

Aditi and Jonas both edit `README.md` in their respective clones. The first to push succeeds. The second, on `bk push`, gets rejected:

```
$ bk push bk://shared-host/ledger
Push detected diverging changesets; pull first
$ bk pull bk://shared-host/ledger
... auto-merges where possible ...
$ bk push bk://shared-host/ledger
```

`bk pull` fetches the missing changesets and runs the merge tool (`bk csettool` in interactive mode, or automated for non-conflicting cases). The merge produces a merge changeset with two parents, recorded structurally in the changeset graph.

### Operation 7 — Merge with conflict

The `parser.py` conflict is resolved during `bk pull` or `bk resolve`. The merge changeset records both parents.

### Operation 8 — Botched commit needing rewrite

BK allows uncommitting the latest changeset (`bk unpull` for not-yet-pushed changes; `bk csettool` and various surgical operations for more elaborate edits). Mireille can `bk unpull` the bad changeset, fix the issue, re-commit. After push, history is intended to be immutable; the audit trail wins.

### Operation 9 — Tag

```
$ bk tag v0.1.0
```

Tags are first-class in BK and travel with the changeset graph in pulls and pushes.

### Operation 10 — Release

A `bk export` to a clean directory plus tarballing produces the release artifact.

## Model and mental load

What you have to hold:

- Per-file delta storage with project-level changesets.
- Clones as the unit of branching, often.
- The pull/push synchronization rhythm.
- Reservation of `bk new` and `bk edit` for files-to-be-modified — the workflow is more explicit than Mercurial's or Git's.

The mental load is moderate. BK was designed for kernel-scale work and required some discipline; it was not a system for the casual user.

## Evolution and history rewriting

Modest. Pre-push history can be edited with `bk csettool`-driven surgical operations. Post-push history is sacred.

## Ecosystem reality

BitKeeper is open-source as of 2016 and dormant in 2026. The repository on GitHub builds and runs on Linux. There are no production projects using it; there is no active community of any size. The historical interest is high; the present-day usage is essentially zero.

## When to reach for it; when not to

You do not reach for BitKeeper in 2026. It is a museum piece. Read about it; understand its place in the lineage; do not deploy it.

## Epitaph

BitKeeper was the first DVCS that worked at scale, was the system Linus left, and is the system whose absence made the modern DVCS world possible.
