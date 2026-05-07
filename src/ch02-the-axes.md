# 2. The Axes

A version control system is a bundle of decisions. Some are visible at the command line, some are buried in storage formats, and a surprising number are cultural — encoded not in the code but in what the system makes easy and what it makes painful enough that a community avoids it. The axes in this chapter are the ones the rest of the book uses to compare systems. They are not orthogonal. Several are correlated by physics, several more by history, and a few are correlated only because the same group of people designed several systems in a row.

The axes are: distribution, atomicity, the unit-of-history model, identity, conflict semantics, locking, branching cost, rename handling, history mutability, signed history, scaling, network model, working-copy model, hosting model, and *what kind of question is easy to ask of history.* Each is described below, with notes on the choices systems have actually made, and what the choice rules in or out.

## Distribution

A centralized system has one canonical repository. Working copies are views into it; checking out a file means asking the server. CVS, Subversion, Perforce, ClearCase, Visual SourceSafe, Vault, and Plastic SCM in its default deployment are all centralized.

A distributed system gives every working copy a full repository, with full history, that can synchronize peer-to-peer. BitKeeper introduced this idea to the mainstream; Monotone, Arch, Bazaar, Mercurial, Git, Fossil, Darcs, Pijul, Sapling, and Jujutsu are all distributed.

The split is less clean than it looks. Distributed systems are almost always *deployed* against a central server — GitHub, an internal Gitea, a Mercurial server with hooks. The "distributed" property is then a property of the protocol and storage, not of the social structure. Conversely, Plastic SCM lets you clone full history if you want, blurring the centralized line in the other direction. The honest version of the axis is: *can a working copy operate fully offline, including making and inspecting commits?* That is what people usually mean by distributed in 2026.

## Atomicity

When you commit a change touching ten files, does the system record those ten files as a single transactional unit, or as ten separate updates that happen to share a timestamp and a message?

CVS got this wrong. A failed CVS commit could leave the repository with five of your ten files updated, no record of the other five, and no built-in way to recover. Subversion's first selling point was atomicity: the commit either applies fully or doesn't apply at all, with a single revision number for the whole operation. Every modern system is atomic per commit; this axis only matters when reading the older systems on their own terms.

## Unit-of-history model

There are three serious answers to "what is the unit of recorded history?"

*Snapshot*. A commit records the full state of the tracked tree, with deduplication of unchanged content. Git, Mercurial, Sapling, and Jujutsu use snapshots. Operations like diff and blame are computed by comparing snapshots; storage is kept reasonable by content addressing and packing.

*Delta*. A commit records the difference from the previous version. RCS used reverse deltas (latest revision is full text, older revisions are reconstructed by applying reverse deltas backward). SCCS used interleaved deltas, where every revision lives in a single weave file marked with which deltas each line belongs to. CVS inherited RCS's per-file deltas. Subversion used a custom binary delta format, fsfs, that is conceptually similar.

*Patch*. A commit records a named operation: rename this file, delete these lines, insert these lines, with rules for when two patches commute and when they conflict. Darcs and Pijul use this model. The advantage is that history becomes a partially ordered set of patches that can be cherry-picked freely between branches and reordered when they don't depend on each other. The disadvantage is that the math is hard, and Darcs's original implementation ran into pathological merge cases that took years to address. Pijul rebuilt the model on a sound theoretical foundation and a working implementation.

The choice of unit determines almost everything else. A snapshot system handles renames by detecting them heuristically (Git does this at view time, not at commit time). A delta system can record renames if you tell it to (Mercurial does, with `hg mv`). A patch system represents the rename as a first-class patch and merges it cleanly when others edit the renamed file.

## Identity

How does the system name a version?

*Sequential integer*. CVS used per-file revision numbers (`1.4`, `1.5`, `1.5.2.1` for branches). Subversion gave the whole repository a single monotonically increasing revision number. Perforce gives each commit a *changelist number*, also monotonic.

*Hash of content*. Monotone introduced the SHA-1 content-addressed model that Git later adopted: the name of a commit is the hash of the commit's bytes, which transitively names the tree, which names the file blobs. This makes history tamper-evident: you cannot change a commit without changing every commit that descends from it. Mercurial uses SHA-1 for changeset IDs as well, but also exposes a per-repository sequence number for ergonomics.

*Hash of patch*. Darcs and Pijul name commits by patch identity, with the property that the same patch applied to different ancestor states keeps the same name. This is what enables free cherry-picking: a patch is a thing that exists, not a delta against a particular parent.

*Stable change ID separate from commit hash*. Sapling and Jujutsu introduce a *change ID* that is stable across rewrites. If you amend a commit, the commit hash changes but the change ID stays the same, so the system can track "the same logical change" through edits. This is the cleanest answer to a problem Git addresses with `--force-with-lease` and the reflog: how do you know that this rebased commit *is* the commit you meant to rebase?

## Conflict semantics

What does the system do when two changes touch the same place?

The naïve answer — flag the conflict, mark it with `<<<<<<<` and `>>>>>>>`, and let the human resolve it — is what almost every system shows the user. The differences are in *how* the system represents and detects conflicts.

A snapshot system computes conflict at merge time by running a three-way diff between the two heads and their common ancestor. The result is good enough for code most of the time; it is poor for some cases (e.g., a function moved from one file to another while another edit changes its body), and it cannot handle cases where there is no clear common ancestor.

A patch system represents conflicts as the algebraic failure of two patches to commute. The system can in principle tell you not just "these lines conflict" but "the rename of `core.c` to `kernel.c` and the edit to line 47 of `core.c` cannot be applied independently because the second depends on the first." Pijul actually does this; Darcs does it for some cases.

A locking system avoids conflicts by preventing concurrent modification. Perforce and ClearCase support this; SCCS shipped with it as the default. We will return to locking below.

## Locking

Optimistic concurrency — let everyone edit, resolve conflicts at merge — is the default for code. Pessimistic concurrency — only one person can edit a file at a time, by checking out a lock — is the default for art assets, schematics, and any file format where automated merge is impossible.

The history of locking in version control is a microcosm of the field. SCCS required you to "lock" a file before editing it (this was a workspace concern, not a network lock — there was no network). RCS continued the practice. CVS abandoned it for code. Subversion brought it back as an option for binary files and gave it a network protocol. Perforce made it central: every file is read-only on disk by default, and `p4 edit` checks the file out for modification, optionally with an exclusive lock if `+l` is set on the filetype. Plastic SCM does the same in its art-asset-friendly mode. Git has no first-class locking; Git LFS bolted on a locking server in 2016 because film and game studios refused to use Git without it.

Locking is the axis where Git is genuinely worse than several systems it supplanted, and any honest version of the design space has to account for that.

## Branching cost

Branching cost has two flavors: *cheap to create* and *cheap to manage*.

CVS branches were a tag plus a revision policy. Creating one was cheap; merging back was difficult enough that teams avoided long-lived branches.

Subversion branches were directory copies. Cheap to create; easy to read, since `svn log /branches/foo` worked. Merging required tracking which revisions had been merged, which Subversion eventually added but never made painless.

Mercurial originally treated each clone as a branch (a stylistic choice, since branches were named in commit metadata but the dominant pattern was clone-per-feature). Later it added bookmarks, which behave like Git branches.

Git branches are pointers into the commit graph. Creating one is essentially free; managing one requires the user to understand the graph well enough to rebase, merge, and reset without losing work. The mental load is high; the mechanical cost is low.

Plastic and Sapling lean into stacked branches and topic branches with first-class support. Jujutsu rethinks the question entirely: there is no separate "create a branch" step; every commit is a branch tip until something else points beyond it.

## Rename handling

If `core.c` is renamed to `kernel.c`, can the system recover the history of the file across the rename?

Git records nothing about renames at commit time and detects them at view time using diff heuristics. This works most of the time and fails subtly when a rename coincides with substantial editing.

Mercurial records renames as commit metadata when you use `hg mv`. Forgetting the `hg mv` is a common mistake; the system has tools to add the metadata after the fact.

Plastic SCM tracks renames as first-class operations and shows file history across them as a continuous line.

Darcs and Pijul represent renames as named patches. A rename is a thing that happened in history, not a heuristic guess.

## History mutability

Is the recorded history immutable? Can it be rewritten? If it can, what are the safety mechanisms?

Mercurial defaulted to "history is permanent." Rewriting required extensions (`mq`, later `histedit` and `evolve`). The default was a strong cultural signal: history is an audit trail.

Git defaulted to "history is yours to rewrite, until you push." `git rebase`, `git commit --amend`, `git filter-branch` (and now `git filter-repo`) all let you reshape history at will. The reflog gives you 90 days to recover from mistakes. The cultural signal is: shape history into a clean narrative before others see it.

Sapling and Jujutsu treat history mutability as a primary feature. Both have stable change IDs that survive rewrites, so the system can show you "this commit, edited four times" rather than four loose commits. Both have an operation log that records the rewrites themselves, so you can undo a rebase that went wrong.

Fossil takes the opposite view forcefully: commits are immutable, and the system will not let you rewrite published history without an explicit "shun" operation that leaves a record of what you did. This is part of Fossil's stance that history is for keeping.

## Signed history

Can commits be cryptographically signed? Is signing first-class or bolted on?

Monotone built signing in from the start: commits are not authoritative until certificates are attached saying who reviewed them and what their status is. Git supports GPG-signed commits and tags, but the support is bolted on and inconsistent across hosts. Fossil signs every commit with the committer's public key as a matter of course.

The axis matters more in regulated industries and security-conscious open-source projects than it does for most commercial development. It will matter more in the future than it does now, as supply chain attacks make provenance a regulatory concern.

## Scaling

The relevant questions: how large can a single repository get before the system breaks down? How large can history get? How large can a single file get?

Git's worst-case scaling is well known: very large monorepos (Linux kernel scale is fine; Google scale is not) and very large binary files force teams onto LFS, partial clones, sparse checkouts, or off Git entirely. Mercurial scales similarly, with Sapling extending the model to handle Meta's monorepo. Perforce scales to terabytes of files comfortably; this is part of why it is the default in games and film. Plastic SCM scales similarly. Subversion scales reasonably for medium repositories; very large ones become slow.

## Network model

How does the system talk over the wire?

CVS used a custom protocol over `pserver`, with optional SSH wrapping; the protocol is chatty and has known weaknesses. Subversion used WebDAV variants and later its own `svn://` protocol. Perforce uses a stateful TCP protocol. Git uses several: `git://`, SSH, HTTPS smart, and dumb HTTP. Mercurial uses HTTP(S) with its own wire format and SSH.

The interesting axis is *whether the network protocol leaks the storage format*. Git's smart HTTP protocol streams pack files, which is fast but exposes Git internals. Sapling abstracts the wire from the storage, which is part of what lets it run against a non-Git backend.

## Working-copy model

What does a checkout look like on disk?

Most systems give you a full tree of files plus a metadata directory (`.git`, `.hg`, `.svn`). Some systems give you something stranger.

ClearCase's MVFS made the working copy a virtual filesystem; what you saw on disk was determined by a *config spec* that selected versions of files dynamically. This was extraordinary in concept and a maintenance nightmare in practice.

Sparse checkouts (Git) and narrow clones (Mercurial, Sapling) let you check out only part of a tree. Virtual filesystems for very large repos (VFS for Git, EdenFS for Sapling) page files on demand.

Perforce's default is a full local "workspace" mapped from a server-side view. The mapping is server state, which means the workspace is auditable from the server side in a way Git working copies are not.

## Hosting model

A hosted forge — GitHub, GitLab, Bitbucket, Gitea, Sourcehut, and the rest — is not part of the version control system per se. But the hosting model has come to be inseparable from the user experience of the system, and a chapter on Git that ignores GitHub is incomplete.

The axis is whether hosting is *imposed* by the system (Fossil ships its own server; Sourcehut's `hut` is built around `git send-email`), *standard* (Git assumes a forge but does not ship one), or *agnostic* (Mercurial, Subversion, Perforce all run against vanilla servers without an opinionated hosted UI).

## What kind of question is easy

The last axis is the most subjective and the most important. Some questions are trivial in some systems and hard in others. Examples:

- *Who wrote line 47 of `parser.c` in March 2019?* Git: `git blame -L 47,47 parser.c | git log`-ish, easy. Perforce: `p4 annotate`, easy. Darcs: doable but unusual. SCCS: trivially easy.
- *Which patches in this branch have been applied upstream, regardless of order?* Darcs and Pijul: the question makes sense, the system can answer it. Git: requires `git cherry` or careful patch-id comparisons.
- *What was the state of the repository on June 5th?* Subversion: `svn co -r {2019-06-05}`, trivial because revisions are global and timestamped. Git: possible but awkward.
- *Show me every change that affected the `auth/` directory.* Git, Mercurial, Sapling: easy. CVS: not coherent because there is no per-tree history.
- *What is the relationship between this branch and main?* Git: graph traversal, sometimes confusing. Jujutsu: directly visible because every commit's relationship to its anchor is part of the model.

These questions are not equally important to every team. But the menu of *which questions are cheap* shapes the team's attention and, over time, what the team chooses to look at. A team on Subversion looked at revision-by-revision history because that was the cheap query; a team on Git looked at branch diffs because that was the cheap query; a team on Darcs looked at patch sets because that was the cheap query. The system's answer to "what is easy" became the team's habit.

## How the format chapters use these axes

Each format chapter — Chapters 4 through 26, plus 28 to 30 for the forge era — covers the system's choices on these axes implicitly, by showing the exemplar repository operations, and explicitly in the *Model and mental load* section. The reader should accumulate a sense, over the format chapters, of which axes Git fixed at one position and which it left open. Chapter 32 returns to the axes as a decision framework. Chapter 34, the closing inventory, picks up the axes where Git's settled position has cost the field something.

The axes are tools. They are not a complete description of any system. They are sufficient to compare. With them in hand, Chapter 3 sets up the scenario every later chapter renders, and the actual work of the book begins.
