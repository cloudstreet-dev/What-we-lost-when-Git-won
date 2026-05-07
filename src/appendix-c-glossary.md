# Appendix C. Glossary

Terms used in the book, with definitions calibrated to how the term is used in the text. Where systems use overlapping terms with different meanings, the system is named.

**Amend** — Replace the most recent commit with one that has different content or a different message. Git: `git commit --amend`. Mercurial: `hg commit --amend`. Sapling: `sl amend`. Jujutsu: implicit in `jj describe` plus working-copy edits.

**Annotate** — Show, for each line of a file, which revision last changed it and by whom. Synonym in many systems: blame.

**Archive** — Produce a clean tarball or zip of a repository at a given revision, without version-control metadata. `git archive`, `hg archive`, `svn export`.

**Atomic commit** — A commit that either fully succeeds or fully fails. Subversion's first selling point over CVS.

**Bare repository** — A repository without a working copy. Used for serving over the network. In Git: a directory whose contents are what `.git/` contains in a normal clone.

**Bisect** — Binary search through history to find the commit that introduced a bug. `git bisect`, `hg bisect`. Requires a test that reports good or bad.

**Blob** — In Git's object model, a file's content with no metadata. Content-addressed by hash.

**Bookmark** — Mercurial's term for a Git-style movable branch reference. Distinct from Mercurial's *named branch*.

**Branch** — A named line of development. The exact mechanism varies: a directory copy in Subversion, a movable ref in Git, a tag value in Fossil, a per-element branch in ClearCase, a stream in Perforce.

**Branch protection** — Platform-side rules that restrict who can push to a branch, what status checks must pass, etc. GitHub, GitLab, etc.

**Bundle** — A binary artifact containing a portable subset of a repository's history. Fossil ships it natively; Git has `git bundle`; Mercurial has `hg bundle`.

**Cert** — A signed assertion attached to a revision. Monotone's primary metadata mechanism.

**Changelist** — Perforce's term for a commit. Pending changelists hold work-in-progress; submitted changelists are in history.

**Changeset** — A project-level commit, in DVCS lineage; the term Mercurial and BitKeeper use; Pijul uses *change*.

**Channel** — Pijul's term for a branch. A named subset of the patch store.

**Check in** — To record changes back into the repository. Verb form of *commit*. SCCS, RCS, ClearCase, Perforce, VSS use this term; DVCS use *commit*.

**Check out** — To extract content from the repository to a working copy. Also: to acquire a lock for editing (in lock-based systems).

**Cherry-pick** — Apply a single commit's changes to a different branch. `git cherry-pick`, `hg graft`, `sl rebase` of a single change.

**Clone** — Make a full copy of a repository, including history. The DVCS analogue of *checkout* in centralized systems.

**Co-Authored-By** — A trailer convention in commit messages for crediting additional contributors. Recognized by GitHub for displaying multiple authors.

**Commit** — A recorded state of the repository, with metadata (author, message, parents). The unit of history in most modern systems.

**Commutation** — Property of two patches that they can be applied in either order with the same result. Foundational to patch-theoretic systems.

**Config spec** — ClearCase's declarative rule for what version of every file appears in a view.

**Conflict** — A situation where automatic merging cannot determine the correct result; requires human resolution. Marked in working trees with `<<<<<<<` `=======` `>>>>>>>`.

**Content addressing** — Naming objects by the hash of their content. Monotone introduced this to version control; Git followed.

**DAG** — Directed Acyclic Graph. The shape of commit history in Git, Mercurial, and most modern systems.

**Delta** — The difference between two revisions. Storage strategy in RCS, BitKeeper, and (partially) Mercurial.

**Depot** — Perforce's term for the repository.

**Detached HEAD** — Git state where HEAD points directly at a commit rather than a branch. Commits made in this state are unreachable when the user moves HEAD elsewhere.

**Diff** — The textual difference between two states. Output format documented in `diff(1)`. Three-way diff requires a base, an "ours", and a "theirs."

**DVCS** — Distributed Version Control System. Each working copy contains full history. Git, Mercurial, Bazaar, Pijul.

**EdenFS** — Sapling's virtual filesystem. Files page on demand; cloning is constant-time.

**Element** — ClearCase's term for a versioned file.

**Evolve** — Mercurial's mutable-history extension. Records obsolescence; tracks lineage across rewrites.

**Fast-forward** — A merge in which the target branch's tip is an ancestor of the source's tip; the merge moves the target's pointer forward without producing a merge commit.

**Fetch** — Retrieve commits from a remote without merging into the local working state. `git fetch`, `hg pull` (without update), `bzr pull` (depending on configuration).

**File ID** — Bazaar's per-file stable identifier, used to track files across renames.

**Filter** — Git LFS's content interception mechanism: a filter intercepts add/checkout to swap pointer files for actual content.

**Fork** — (1) An independent line of development split from a project. (2) A copy of a repository, on a platform, owned by a different user; one click on GitHub.

**Forge** — A hosted platform combining version control with adjacent functions (issues, review, CI). GitHub, GitLab, Bitbucket, Gitea, Forgejo, Sourcehut.

**FSFS** — Subversion's filesystem-based storage backend (succeeded Berkeley DB).

**Garbage collection** — Removal of unreferenced objects. Git's `git gc`. Mercurial does not run a periodic GC; cleanup is implicit.

**Gluon** — Plastic SCM's simplified UI for non-developers.

**HEAD** — The current commit / branch pointer in Git. The "you are here" of the repository.

**Hook** — A script run at a specific lifecycle point (pre-commit, post-receive, etc.).

**Index** — Git's staging area: the prepared content for the next commit. Stored in `.git/index`.

**Integration** — The act of bringing changes from one branch into another. Sometimes a synonym for merge, sometimes a richer concept (Perforce: `p4 integrate`).

**LFS** — Large File Storage. Git's bolt-on for handling binary files.

**Line of Development** — BitKeeper's term for a branch.

**Lock** — A claim on a file (advisory or enforced) that prevents others from editing. Pessimistic locking is the default in Perforce, ClearCase (with reserved checkouts), VSS, etc.

**Mainline** — The main line of development. Often synonymous with *trunk* (centralized) or *main* (Git).

**Manifest** — In Mercurial and Git, a representation of a tree state: the list of (path, blob-hash) pairs.

**Merge** — Combine two lines of development. Three-way in most systems; algebraic in patch systems.

**Merge commit** — A commit with two or more parents, recording the integration of branches.

**Mononoke** — Sapling's server, a Rust reimplementation of the Mercurial server protocol.

**MVFS** — ClearCase's Multi-Version FileSystem; the kernel-level virtualization layer.

**Obsolescence marker** — Mercurial evolve / Sapling concept: metadata recording that one commit has been replaced by another.

**Operation log** — Jujutsu's log of every operation performed on the repository. Enables undo of any past operation.

**Origin** — The default name for the primary remote in Git's conventions.

**Pack** — Git's compressed bundle of multiple objects with delta compression. Stored in `.git/objects/pack/`.

**Patch** — A textual representation of a change, applicable with `patch(1)` or `git apply`. In patch theory: a named, identity-bearing operation.

**Patch set** — Gerrit's term for one revision of a change in response to review.

**Patch theory** — The mathematical framework for treating patches as objects with explicit dependencies and commutation rules. Darcs, Pijul.

**Pull** — Retrieve and integrate changes from a remote. `git pull`, `hg pull --update`, `svn update`.

**Pull request** — A platform-level object representing a request for the maintainer to integrate the contributor's branch. GitHub's invention.

**Push** — Send local commits to a remote. `git push`, `hg push`, `bzr push`.

**Rebase** — Move a series of commits to a new base commit, replaying them. Rewrites history. Git's `rebase`, Mercurial's `hg rebase` (extension).

**Reconcile** — Perforce: `p4 reconcile` sweeps the workspace and identifies new, modified, deleted, or moved files for opening.

**Record** — Darcs and Pijul: the verb for committing.

**Ref** — A name pointing at a commit. Branches and tags are refs in Git's terminology.

**Reflog** — Git's log of every change to every ref. Default 90-day retention. The recovery mechanism for lost work.

**Repository** — The store of versioned data plus history.

**Revert** — Reverse the effects of a commit by creating a new commit that negates it. `git revert`, `hg backout`, `svn merge -c -N`.

**Revision** — A specific recorded state. Centralized systems often use sequential revision numbers; DVCS use hashes plus optional local sequences.

**Revlog** — Mercurial's per-file storage format: append-only with snapshot-or-delta entries and chain length caps.

**Shelve** — Server-side or local stash of in-progress changes. `p4 shelve`, `hg shelve`, equivalent to Git's `git stash` with subtle differences.

**SID** — SCCS Identifier. Dotted revision number: 1.1, 1.2, 1.2.1.1, etc.

**Sigstore** — Certificate transparency log used for keyless code signing. Increasingly used for Git commit and tag signing.

**SLSA** — Supply-chain Levels for Software Artifacts. A framework for software-supply-chain provenance.

**Smart HTTP** — Git's HTTP-based protocol that streams pack files. The dominant Git network protocol.

**Smartlog** — Sapling's default log view: the changes that matter to you, not the entire history.

**Snapshot** — A complete record of a tree state at one point. Git, Mercurial (logically), Sapling, Jujutsu.

**Sparse checkout** — A working copy containing only a subset of the repository's tree. `git sparse-checkout`, Mercurial's narrow extension, Perforce's view restrictions, Sapling's native support.

**Squash** — Combine multiple commits into one. `git rebase -i` with `squash` or `fixup`.

**Stacked diff** — A workflow with an ordered sequence of small commits, each reviewed independently. Common at Meta, increasingly in Sapling and Jujutsu projects.

**Stash** — Git's local-only mechanism for setting aside in-progress changes.

**Stream** — Perforce and AccuRev: a first-class branch with declared parent and inheritance rules.

**Submit** — Perforce: ship a pending changelist atomically. Gerrit: integrate an approved change.

**Submodule** — Git's mechanism for embedding one repository in another by reference.

**Subtree merge** — Git's alternative to submodules: copy embedded code into the parent's history.

**Sync** — Update working copy from remote, in various systems. Perforce: `p4 sync`. Fossil: `fossil sync`.

**Tag** — A named pointer to a commit, typically marking a release. May be lightweight (just a ref) or annotated (with metadata) and signed.

**Three-way merge** — Merge using a common ancestor plus two heads. The dominant text-merge algorithm.

**Topic branch** — A short-lived branch for a specific feature or fix. Conventional in DVCS workflows.

**Trunk** — The main line of development in centralized systems. CVS, Subversion, ClearCase, Perforce.

**Typemap** — Perforce's table mapping filename patterns to file types (text, binary, binary+l, etc.).

**View** — ClearCase: the user's working perspective, configured by a config spec. Perforce: the workspace's mapping of depot to local.

**VOB** — ClearCase's Versioned Object Base. The storage unit.

**Weave** — SCCS's interleaved-delta storage format. Lines marked with which deltas added or removed them.

**Working copy** — The directory tree of files the user works in. Synonym in many systems: working tree, workspace.

**Workspace** — Perforce's name for a working copy, with associated server-side state. Plastic SCM also uses this term.
