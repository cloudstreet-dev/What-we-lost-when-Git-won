# 20. Why "Commits as Snapshots" Was a Choice, Not a Discovery

This chapter is argument, not history. It tries to make explicit a position that the rest of the book has been gesturing at, and that working programmers under thirty often do not realize is even a position. The position: the dominant mental model of version control — that history is a sequence (or DAG) of commits, that each commit names a snapshot of the working tree, that branches are names attached to commits — is *one design choice among several*, with specific consequences, and was not arrived at by ruling out alternatives. It was arrived at by Linus Torvalds in early April 2005 because it was the model that fit the constraints he was working under, and was kept by the field because Git won.

The alternatives are not extinct. They are, on technical merits, in some respects superior. They lost because of a network effect rooted in the shape of the open-source community, the timing of GitHub, and the gravitational pull of social-coding hosting. The snapshot model is *good*; it has real merits; this chapter is not arguing it is *wrong*. The chapter is arguing that "commits are snapshots" is not a discovery about the nature of version control. It is a stance.

## The snapshot model

A snapshot is a complete recording of the working tree's state at one moment, content-addressed for deduplication. A commit is a snapshot plus parent pointers plus metadata. History is a DAG of commits.

Operations in the snapshot model are tree comparisons and tree manipulations. `git diff` is computed by comparing two snapshots. `git log` walks the DAG. `git blame` walks the parents of a file's tree position, looking for which commit introduced each line. `git merge` does a three-way diff between two snapshots and their best common ancestor. `git rebase` produces new commits by applying snapshots-of-changes to a different base.

The model has clear advantages. It is *intuitive*: most programmers, asked what version control does, will say "saves the state of my files at points in time." It is *fast for the common case*: looking up a file at a commit is cheap because the snapshot has it directly. It is *robust*: a commit's identity does not depend on subtle algebraic properties of patches; it is just the hash of its bytes. It is *simple to implement*: Git's storage model, once described, fits in a chapter (this book gave it one).

The model also has costs. The costs are visible if you know to look for them.

## What the snapshot model makes hard

*Cherry-picking is mechanical, not structural.* In Git, `git cherry-pick HASH` replays a commit's changes onto the current branch, producing a *new commit* with a different hash. The cherry-picked commit and the original have no structural relationship in the graph; they are independent objects with similar contents. Tools like `git cherry` (no relation to the verb) compare commits by patch ID to detect that the same change has been applied in two places, but the comparison is heuristic and fails when the changes are non-trivial.

In a patch system, cherry-picking is simply *applying the same patch* in a different repository. The patch has the same identity in both places. There is no need for `git cherry`-style heuristics; the system knows the patches are the same because they *are* the same.

*Renames are heuristic.* Git's no-rename-metadata position is a deliberate choice with a specific cost: a rename combined with substantial editing of the renamed file produces a commit where Git's similarity heuristic fails to detect the rename, and `git log --follow` loses the file's history. Mercurial avoided this by recording renames at commit time. Patch systems avoided it by representing the rename as a first-class operation.

The Git position has a defense: the heuristic is good enough most of the time, and recording renames at commit time forces the user to declare what changed when often the user is wrong about it. Both positions are reasonable. They are *positions*, not discoveries.

*Free reordering is impossible in general.* Git's `rebase` rewrites history by replaying commits in a new order. The replay can fail (rebase conflicts), and the replayed commits have different hashes. The commits *do not commute* — there is no notion in Git of "this commit and that one are independent and could be applied in either order with the same result."

In a patch system, commutation is the basic operation. Two patches commute if they touch independent parts of the tree. The system can answer "is the order of these patches load-bearing?" structurally. Reordering is free where commutation holds, and a structural conflict where it does not. There is no equivalent of a rebase that "happens to work."

*Merging across genuinely independent histories is awkward.* Git's three-way merge requires a common ancestor. Two repositories with no common ancestor cannot be merged structurally; the user must construct an artificial merge with `git merge --allow-unrelated-histories`, which produces a commit with two parents but no shared base. The result is a tree that contains both inputs, with conflict resolution forced for any overlap.

In a patch system, the question "merge these histories" is *just "apply this patch set to that one"*. The system does not need a common ancestor; it needs to decide which patches commute and which do not. Two repositories can share a partial set of patches, or none, and the merge is the union with appropriate conflict resolutions.

*Cross-cutting changes are recorded as their final shape, not their evolution.* In Git, a refactor that moves a function from one file to another and edits it produces a single commit (or a few commits) recording the final state. The structural fact "the function `parse_amount` moved from `parser.py` to `tokens.py` and was edited" is not directly recorded; it is reconstructed by tools running on the snapshots after the fact.

In a patch system, the structural fact is the patch. The history *says* the function was moved and edited; tools that walk history use that information directly.

*Merge commits are first-class but tell you only the final reconciliation.* Git's merge commit records that two histories joined at this point; it does not record *which conflict was hit, which resolution was chosen, why*. A patch system's resolution patches are normal patches, with messages and authors and dependencies; the resolution is a thing in history, not an attribute of a commit object.

## What the patch model makes hard

The patch model has its own costs.

*Performance at large scale is harder.* Operations on snapshots benefit from straightforward indexing: build a tree of trees, look up a path, you have your file. Operations on patches require reasoning about commutations and dependencies, which are operations over the patch set rather than direct lookups in a tree. Darcs's exponential-merge problem was a worst-case manifestation of this; Pijul's pushout-based model avoids the worst cases but the average case still requires more cleverness than Git's tree comparison.

For very large repositories with millions of patches, the patch-system tooling has not been pushed nearly as hard as Git's. Whether the patch model can scale to Linux-kernel sizes with the same performance Git delivers is an open question. There are reasons to believe it can; nobody has done the work.

*The user model is less familiar.* "A patch with computed dependencies that may or may not commute with other patches" is a more abstract object than "a snapshot of my files." Programmers who have learned Git first take a while to internalize the patch model; the time investment is real, and is one of the costs of any patch-system migration.

*Tooling assumes snapshots.* Every diff viewer, every code search engine, every blame UI, every CI system, every IDE integration assumes the snapshot model. A patch system has to either translate to that model on the fly (losing structural information) or persuade the tool to support its native model (which has rarely happened). Git's tooling ecosystem is not just larger; it is *shaped* around the snapshot model in ways that make rebuilding for the patch model costly.

*The conceptual surface is larger.* A snapshot model has commits, trees, blobs, and parent pointers. A patch model has patches, dependencies, channels, conflicts as objects, resolutions as objects, and the algebra of commutation. The cleaner theoretical foundation means more concepts at the surface for the user to encounter. Pijul has worked hard to keep this manageable; it remains larger than Git's, irreducibly.

## The hybrid models

Mercurial's revlog is a hybrid of snapshot and delta: snapshots taken periodically, deltas in between, with chain-length caps to bound retrieval cost. The hybrid is a practical compromise that delivers most of the snapshot model's properties at delta-encoding's storage costs.

BitKeeper used per-file SCCS-style storage with project-level changesets binding them. The unit-of-history was the changeset (snapshot-like in semantics) but the storage was deltas (per-file SCCS).

Sapling and Jujutsu (covered later) layer change-IDs on top of snapshots: the underlying storage is snapshot-based (in fact compatible with Git's), but operations like rebase and amend track *change identity* across rewrites. This is one of the patch-theoretic ideas — that a change has identity beyond its current bytes — implemented on a snapshot substrate. The result is a system that looks snapshot-like on disk and patch-like in operations.

These hybrids are evidence that the snapshot/patch axis is not a binary. The choices live in a continuum. Git is at one end; Pijul at the other; everything in this book lives somewhere on the line.

## What about the rest of the design space?

The snapshot-vs-patch axis is the most visible, but it is not the only place where Git has made a choice that is treated as a discovery.

*The DAG-as-history model.* Git treats history as a directed acyclic graph of commits, with merge commits as nodes with multiple parents. This is a choice. Patch systems do not have a DAG-as-history model; they have a partially ordered set of patches. Centralized systems (CVS, Subversion) have linear-with-branches; the relationship between branches is conventional, not graph-structural.

The DAG model has the advantage of being a single shape that captures both linear and branching history. It has the disadvantage of being unfamiliar to humans, who tend to think about history linearly. The visual representations of Git history (`git log --graph`, gitk, GitHub's network view) are valued because they expose the structure in a form humans can reason about; the existence of those tools is evidence that the underlying model is not natively legible.

*The blame-by-line-tracking model.* `git blame` answers "who last touched this line?" by walking history, comparing each commit's tree to its parent's, and finding where each line was last modified. The model treats lines as the unit of attribution, with rename and copy heuristics at view time.

A patch system records *where each line came from* structurally, because line creation and movement are patches. Line-level provenance, in a patch system, is a query against the patch graph, not a heuristic walk over snapshots.

*The single-author commit model.* Every Git commit has one author and one committer. Co-authorship is recorded conventionally in the commit message (the `Co-Authored-By:` trailer). This is a choice; the system does not require it to be one author. Some systems (notably Gerrit's review model, covered in Chapter 30) treat reviewer signatures as part of the commit's authorization.

*Linear text as the primary unit.* Git's diff is a line-based three-way diff. Hunks are the unit of merge. This works well for code; it works poorly for prose, for structured data, for binary content. Tools layered on Git (semantic diff for some languages, special tools for Word documents, Git LFS for binaries) try to address this. A version control system designed for non-textual content might have made different choices at the unit level. CRDT-based collaborative editing tools (Google Docs, Figma) have made very different choices, and they are doing version control even if they are not on this book's list.

## Why the choice was made the way it was

Linus's design notes from April 2005 are clear about his constraints. He needed a system that could run against Linux kernel scale. He needed it fast, on commodity hardware, with no central server. He needed a model that was simple enough to implement in a week and prove correct by running it. The snapshot model gave him all of those: trees and blobs as content-addressed objects, parent pointers as the only structural complexity, performance from direct lookup rather than from algebraic reasoning.

The choice was a constraint-driven one. Other constraints would have produced other choices. If Linus had needed to support free cherry-picking as the primary collaboration mode (as some open-source projects with tightly-managed integration would have wanted), the patch model might have been more attractive. If he had needed file-level locking for binary content, the centralized model might have made more sense. The constraints he had — kernel scale, speed, no central server, written in a week — pointed at snapshots.

The snapshot model was *fit for purpose*. Treating it as the only purpose is the error.

## What the field forgot

Three concrete pieces of intellectual content have been buried by the snapshot model's dominance.

*Patches as identity-bearing objects.* The Darcs/Pijul lineage's central insight is that a *change* is a thing in the world, with identity that survives rebasing, cherry-picking, and reordering. A change is not a delta against a particular parent; it is an operation that can be applied wherever its dependencies are satisfied. This idea is alive in Sapling and Jujutsu's change-IDs and in patch systems' patches. It is not present in Git, and most Git users have not encountered it.

*Commutation as a structural property.* Two patches commute if they touch independent parts of the tree. This is a *checkable* property in a patch system; it is the foundation of "can these be reordered without changing the result?" A snapshot system can ask "if I cherry-pick A and then B, do I get the same thing as B then A?" but the answer is computed by replaying snapshots, not derived from the structure.

*Conflicts as first-class objects.* Pijul records conflicts in the repository state and resolves them with explicit changes that have authors and timestamps. The structural fact "this is a resolution of conflict X by author Y" is in history. Git's conflicts are transient; the resolution is folded into a merge commit, with no record of which lines were the conflict, which resolution was chosen, or why.

These three ideas are not the same; they are not Git's deficiencies; they are positions the field could have taken and largely did not. The book's later chapters treat them as part of the inventory of what got lost.

## What this means for choosing

This chapter does not say "use Pijul." Pijul's tooling is small, and most teams have constraints that point at Git. The chapter says: when a team is choosing a version control system, "we use Git because Git is what version control is" is not the same as "we use Git because Git fits our constraints." The first is a category error. The second is a design decision.

A team whose work is text and snapshots and forks-and-pull-requests has constraints that match Git well. The choice is overdetermined; Git is the right answer.

A team whose work involves heavy cherry-picking between long-lived release branches, or whose operations are dominated by reordering and amending stacks of related changes, has constraints that point at the patch model or at the change-ID hybrids (Sapling, Jujutsu). The choice is less overdetermined; the team has to evaluate trade-offs.

A team whose work is binary-heavy has constraints that point at Perforce or Plastic SCM, regardless of the snapshot/patch axis.

A team that wants the integrated forge-as-substrate experience without the per-seat licensing has Fossil as a real candidate.

The chapter's claim is just this: the choices are real, the trade-offs are real, and the answer is not always Git. Treating the snapshot model as the only intelligible model makes the question *which version control system does this team need* invisible, because the question reduces to *which Git workflow are we using*. The reduction is a loss of expressive range.

## What comes next

Part VI begins after this chapter, covering specialty and domain systems where the snapshot model's defaults are wrong: Plastic SCM (binaries, branches, non-developer workflows), Git LFS and git-annex (binaries on a snapshot substrate), DVC (ML data and experiments), Helix Core (the full art pipeline), Sapling (change-IDs at scale), Jujutsu (change-IDs as the foundation). Each represents a position the design space has visible occupants in, and most of those occupants are working systems with active users.

Read Part VI as the field's continued investigation of where the snapshot model fits and where it does not, even now, even in 2026, even with Git as the default everywhere.
