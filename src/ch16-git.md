# 16. Git

## Origin

Linus Torvalds began work on Git on April 3, 2005, the week after BitMover revoked the kernel project's free BitKeeper license. The first self-hosting commit — Git tracking its own source — is dated April 7, 2005. By mid-April, the kernel was being developed against Git. Junio C Hamano joined as a contributor and took over as primary maintainer in July 2005, a role he has held continuously since.

Linus's initial design notes are part of the Git source tree (in commit messages and in `Documentation/` files preserved through the project's life). The priorities he stated, in roughly the order he stated them: *speed*, *simple data structure*, *strong support for non-linear development*, *fully distributed*, *able to handle large projects efficiently*. The first three were lessons from BK; the fourth was the position the kernel community had landed on; the fifth was the kernel scaling problem reified.

The design that resulted is, by the standards of the systems in this book, structurally simple. Git stores four kinds of object — *blobs* (file contents), *trees* (directory listings), *commits* (snapshots with parents and metadata), and *tags* (annotated pointers) — in a content-addressed store, with each object named by the hash of its contents. Refs (branches, tags, the HEAD pointer) are files containing object hashes, kept in the `.git/refs/` tree and the `.git/HEAD` file. That is essentially the whole storage model. Everything else — the index, packs, the network protocols, the rich command set — is built on top.

The user-visible system that grew on top of that simple core is the system of legend. The plumbing/porcelain split — low-level commands suitable for scripting (`hash-object`, `update-index`, `read-tree`) and high-level commands suitable for users (`commit`, `branch`, `merge`) — produced a tool that is composable in a way no other system in this book is. It also produced a CLI vocabulary of fifty-plus user-facing commands, with overlapping flags and historical idioms accreted over twenty-one years, that is the most-mocked CLI in working programmer culture.

Git won. The chapter describes the system on its own terms, walks the exemplar, and notes where Git's design choices have specific consequences. The judgments come later, in Chapter 34.

## The object model

The four object types are stored in `.git/objects/`. In the unpacked form, each object is a single file at `.git/objects/AB/CDEFGHIJ...`, where `ABCDEFGHIJ...` is the SHA-1 (or SHA-256, in modern configurations) hash of the object's contents. The first two hex characters form the directory; the rest, the filename. Objects are zlib-compressed.

A *blob* is a file's content, with no metadata about filename, mode, or location. Two files with identical content share a blob.

A *tree* is a list of (mode, name, hash) entries, where the hash refers to either a blob (a file) or another tree (a subdirectory). Trees deduplicate at the directory level; if two commits share an unchanged subdirectory, they share the same tree object.

A *commit* references a tree (the snapshot of the working state), a list of parent commits (zero for the root commit, one for a normal commit, two or more for a merge), an author, a committer, timestamps, and a message. The commit's hash is the hash of all of these together.

A *tag* (the *object*, distinct from the *ref*) is an annotated tag: a hash plus a target reference, a tagger, a timestamp, and a message. Lightweight tags are just refs pointing at commits; annotated tags are objects pointing at commits with their own metadata.

The content-addressing has properties Git's chapter has to state plainly. Identical files anywhere in history share the same blob. Identical subtrees share the same tree. A commit's hash is a one-way fingerprint of the entire history reachable from it: changing any byte of any ancestor commit changes the descendant commit's hash. This is the property Monotone introduced and Git inherited; it is the property that makes Git's distributed model trustworthy across mutually-distrusting peers.

The hash transition from SHA-1 to SHA-256 has been in progress for years; SHA-1 collisions exist in theory and in adversarial proofs of concept, and the project has the infrastructure for SHA-256 repositories with a translation table for legacy. Most working repositories in 2026 are still SHA-1; new high-value repositories increasingly use SHA-256.

## The reference model

A *ref* is a name pointing at a commit. Branches and tags are both refs.

`.git/refs/heads/main` contains the hash of the commit that `main` points at. Branches in Git are simply files in `.git/refs/heads/`. Creating a branch means creating a file; deleting a branch means deleting a file. There is no graph structure storing "the set of branches"; there is the filesystem listing of `.git/refs/heads/`.

`.git/HEAD` contains either a ref name (`ref: refs/heads/main`) — meaning "we are on the `main` branch" — or a commit hash directly, the *detached HEAD* state. When `HEAD` is on a branch and a commit is made, the branch's ref file is updated to point at the new commit. When `HEAD` is detached, the new commit becomes a free-floating object and the user is responsible for tagging or branching from it before checkout, or it becomes garbage.

Remote-tracking refs (`refs/remotes/origin/main`) record where remote branches were last seen. They update on `git fetch`; they are read-only locally.

Tags live in `.git/refs/tags/`. Lightweight tags point directly at commits; annotated tags point at tag objects.

Reflogs (`.git/logs/refs/...`) record every change to every ref, with timestamps and reasons, for a configurable retention period (default 90 days). The reflog is the safety net: if you accidentally `git reset --hard` away a branch, the reflog has the previous commit, and `git reflog show` plus `git checkout HASH` recovers it.

## The index

The *index* (also called the *staging area* or the *cache*) is `.git/index`, a binary file recording the next commit's content. `git add` updates the index; `git commit` reads the index, builds a tree from it, and creates a commit pointing at the tree.

The index is a third "tree" alongside the working directory and HEAD. It is one of Git's distinctive features and one of its most criticized: users routinely confuse "added to the index" with "committed," and the model has been called overcomplicated for the use case Git's userbase actually has. The official Git defense is that the index allows you to compose a commit precisely from a working tree containing both committable and uncommittable changes — `git add -p` to selectively stage hunks within a file is the high-leverage case.

The three-tree model — HEAD, index, working tree — is the mental scaffold for understanding most Git commands. `git diff` defaults to showing index-vs-working; `git diff --cached` shows HEAD-vs-index; `git diff HEAD` shows HEAD-vs-working. `git checkout` updates working tree from HEAD or a ref; `git reset` updates HEAD (and optionally index and working) to a different commit.

## Pack files and garbage collection

Loose objects (one file per object) are inefficient at scale. Git periodically packs them: `git gc` runs `git pack-objects` to bundle reachable objects into `.git/objects/pack/pack-*.pack` files, with delta compression between similar objects. The pack format is documented and stable.

Unreachable objects — orphaned by branch deletion or history rewriting — are kept until garbage collection (default 30 days, configurable). The reflog keeps them reachable for the default 90 days. After both expire, `git gc` deletes them.

## Network protocols

Git speaks several protocols. `git://` is a custom TCP protocol; `ssh://` wraps the same protocol over SSH; `https://` and `http://` use a smart protocol layered on HTTP that streams pack files. The smart HTTP protocol is what almost all modern Git interactions use; it works through firewalls, supports authentication, and is what every Git host implements.

The protocol design is substantially simpler than Subversion's WebDAV layering or Perforce's stateful TCP protocol. The server-side responsibilities are minimal: receive packs, send packs, run hooks. Mirroring a Git repository requires only the ability to copy `.git/`.

## The CLI

Git's commands divide into *plumbing* (low-level, scriptable, stable) and *porcelain* (high-level, user-facing, evolving). The split was a real design intention; in practice, plumbing is rarely used directly, and porcelain has accumulated complexity over twenty-one years.

The high-level commands every user knows: `init`, `clone`, `add`, `commit`, `push`, `pull`, `fetch`, `merge`, `rebase`, `branch`, `checkout`, `switch` (newer, simpler version of checkout for branches), `restore` (newer, simpler version of checkout for files), `log`, `diff`, `status`, `reset`, `revert`, `tag`, `stash`, `cherry-pick`, `bisect`. The list of commands a working programmer touches in a year is twenty to thirty; the full set is over a hundred when plumbing is included.

The man pages are dense. The `gittutorial` and `giteveryday` documents are good introductions, and most working users learn by absorbing patterns from coworkers and Stack Overflow rather than from primary documentation.

## Scenario walkthrough

The exemplar in Git is, by 2026, the workflow most readers will know best.

### Operation 1 — Initial import

```
$ cd /path/to/initial/ledger
$ git init
$ git config user.email "aditi@example.org"
$ git config user.name "Aditi Rao"
$ git add .
$ git commit -m "Initial import"
```

The first commit creates the root commit, which has no parents. The commit's hash is computed; `main` is created pointing at it; `HEAD` references `main`. To publish:

```
$ git remote add origin git@github.com:cloudstreet-dev/ledger.git
$ git push -u origin main
```

### Operation 2 — Linear development

```
$ vi src/ledger/parser.py
$ git add src/ledger/parser.py
$ git commit -m "parser: handle blank lines and comments"
```

Each commit advances `main`. `git log --oneline` shows the linear history.

### Operation 3 — Branch

Jonas clones:

```
$ git clone git@github.com:cloudstreet-dev/ledger.git
$ cd ledger
$ git switch -c currency-conversion
```

The branch is local; `git push -u origin currency-conversion` publishes it. He commits twice on the branch:

```
$ vi src/ledger/parser.py
$ git commit -am "parser: store amounts as (value, currency)"
$ vi src/ledger/reports.py
$ git commit -am "reports: convert amounts to display currency"
$ git push
```

### Operation 4 — File rename

```
$ git mv src/ledger/reports.py src/ledger/report_balance.py
$ # edit report_balance.py
$ # create report_register.py
$ git add src/ledger/report_register.py
$ git commit -m "reports: split into balance and register modules"
```

Git records no rename metadata; the commit records new tree state. `git log --follow src/ledger/report_balance.py` follows the file's history across the rename using diff-based heuristics. The default similarity threshold (50%) catches the rename in this case; for cases where rename + heavy editing breaks the heuristic, `--find-renames=N` tunes the threshold.

The "no rename metadata" choice is one of Git's most-debated. Linus's argument was that the heuristic is good enough in practice, that recording renames at commit time forces the user to declare what changed when often it is more useful to figure it out later, and that the model is simpler without it. The counter-argument — that some renames *cannot* be recovered heuristically and should have been recorded — is real, especially for code with substantial structural reorganization.

### Operation 5 — Binary file added

```
$ git add docs/logo.png
$ git commit -m "docs: add project logo"
```

Git stores the binary as a blob. There is no built-in special handling. Subsequent revisions store full new blobs; pack-time delta compression sometimes helps, sometimes does not.

For projects with many or large binaries, Git LFS (separate chapter) is the conventional answer. Without it, repositories with binary content grow without bound, and the cost shows up as longer clone times and large pack files.

### Operation 6 — Parallel edits

Aditi and Jonas both edit `README.md`. Aditi commits and pushes first. Jonas's push is rejected:

```
$ git push
To github.com:cloudstreet-dev/ledger.git
 ! [rejected]        main -> main (fetch first)
error: failed to push some refs to ...
hint: Updates were rejected because the remote contains work that you do
hint: not have locally.
$ git pull --rebase    # or git pull (which merges)
... auto-rebases his commit on top of Aditi's ...
$ git push
```

The choice between merge and rebase on pull is one of the cultural fault lines in Git practice. The exemplar uses rebase; the team's policy is "linear main, no merge commits from pulls."

### Operation 7 — Merge with conflict

For the `currency-conversion` merge:

```
$ git switch main
$ git merge currency-conversion
Auto-merging src/ledger/parser.py
CONFLICT (content): Merge conflict in src/ledger/parser.py
Automatic merge failed; fix conflicts and then commit the result.
$ vi src/ledger/parser.py    # resolve <<<<<<< markers
$ git add src/ledger/parser.py
$ git commit -m "Merge currency-conversion into main; resolve parser conflict"
```

The merge produces a commit with two parents. `git log --graph` shows the structure. For teams that prefer linear history, `git rebase main` on the branch before merging produces a fast-forward instead of a merge commit.

### Operation 8 — Botched commit

Mireille's botched commit, before push:

```
$ vi src/ledger/report_register.py    # remove debug line
$ git add src/ledger/report_register.py
$ git commit --amend -m "register: add --csv flag"
```

`--amend` rewrites the latest commit, replacing it with a new commit (different hash, same parents) incorporating the new tree and message. The original commit is in the reflog for 90 days.

After push, the published commit cannot be silently rewritten. The options:

1. Force-push (`git push --force-with-lease`) — rewrites the remote ref, requires that no one else has pulled the bad commit.
2. Revert (`git revert HEAD`) — creates a new commit reversing the bad change. Audit trail preserved.

For the exemplar, Mireille catches the mistake before push and uses `--amend`.

### Operation 9 — Tag

```
$ git tag -a v0.1.0 -m "First public release"
$ git push origin v0.1.0
```

Annotated tags create tag objects with metadata. Lightweight tags (`git tag v0.1.0`) just create refs.

### Operation 10 — Release

```
$ git archive --format=tar.gz --prefix=ledger-0.1.0/ v0.1.0 -o ledger-0.1.0.tar.gz
```

Or, on GitHub, the tag automatically appears as a release entry; uploading additional artifacts is done through the web UI or `gh release create`.

## Model and mental load

What you have to hold:

- Four object types, content-addressed.
- Refs as files in `.git/refs/`.
- The three-tree model: HEAD, index, working tree.
- The branch as a movable pointer; the difference between a fast-forward and a true merge.
- Detached HEAD: what it means, when you end up there, how to recover.
- The reflog: 90 days of recovery for ref movements.
- Push, pull, fetch, merge, rebase, and the policy choices among them.
- History rewriting tools: `--amend`, `rebase -i`, `reset`, `cherry-pick`, `revert`.
- The commit graph as a DAG.

The mental load is high. Git rewards investment: a programmer who has spent a year doing daily Git operations has internalized the model and can reason about most situations. A programmer who has spent a week is still being surprised. The cliff between novice and journeyman is steeper than for any other system in this book except possibly ClearCase.

The CLI's design has also accumulated technical debt. `git checkout` does too many things; `git switch` and `git restore` were added in 2019 to split it. `git reset` has three modes (`--soft`, `--mixed`, `--hard`) with non-obvious differences. `git pull` has a long-running argument about whether its default should be merge or rebase. These are not philosophical disagreements; they are the visible marks of a CLI that grew over twenty-one years without breaking compatibility.

## Evolution and history rewriting

Git's stance is the most permissive of any mainstream system in this book. Local history is yours to reshape; published history is yours to coordinate with the team about. The reflog provides 90 days of recovery for accidents.

`git rebase -i` (interactive rebase) is the canonical rewriting tool: pick, edit, squash, fixup, drop, reword, exec. Combined with `--autosquash` and `commit --fixup=HASH`, the rebase workflow lets users compose commits during development that automatically merge into the right places at rebase time.

`git filter-repo` (the modern replacement for `git filter-branch`) rewrites the entire history surgically: remove a file, change author email, drop a directory. The original history is preserved if needed; the rewritten history has new hashes and is, for distribution purposes, a different repository.

The reflog and `git fsck --lost-found` are the recovery surface. Many a Git user has found their seemingly-deleted commit through `git reflog show`.

## Hooks, submodules, and other appendages

Git ships hooks (`pre-commit`, `pre-push`, `post-receive`, etc.) — shell scripts run at lifecycle points. Server-side hooks (`pre-receive`, `update`, `post-receive`) enforce policy on push. Hosted forges expose subsets through their UIs; `pre-commit` (the framework, distinct from the Git hook) wraps the hook system in a config-driven tool that has become near-universal in serious projects.

Submodules (`.gitmodules`) embed one repository as a versioned reference within another. The model is famously confusing: a submodule is a commit hash recorded in the parent repository, plus a clone of the referenced repository inside the parent's working tree. Updating the submodule is a two-step process (update the submodule, commit the new pointer in the parent). Most teams that try submodules eventually adopt subtree merges, monorepo packaging, or vendoring instead.

Sparse checkouts (`git sparse-checkout`) check out only part of a tree. Partial clones (`--filter=blob:none`) clone only the commit graph, fetching blobs on demand. Shallow clones (`--depth=1`) fetch only recent history. These three together let Git scale to repositories larger than its original design contemplated, at the cost of significant complexity in the user's mental model.

## Ecosystem reality

Git is the version control system. GitHub is its dominant forge; GitLab, Gitea, Bitbucket, Sourcehut, Codeberg, and self-hosted forges round out the ecosystem. Tooling assumes Git: CI/CD systems, IDE integrations, documentation generators, code search, blame UIs, dependency-pinning tools, security scanners. Working programmers in 2026 who have not used Git are uncommon enough to be a hiring signal, for better or worse.

The Git project itself is healthy. Junio C. Hamano remains the maintainer. Releases come on a roughly quarterly cadence. New features arrive — `switch` and `restore`, `sparse-checkout` improvements, partial clone, the SHA-256 transition, `commit-graph` for performance — and the project's stability and backwards compatibility are exemplary.

Git's blind spots — locking, large binaries, very large monorepos — have produced a thicket of adjacent tooling: Git LFS (Chapter 22), Sapling at Meta (Chapter 25), VFS for Git at Microsoft (now a Git core feature, partial clone), Helix4Git for Perforce shops, Mononoke at Meta, Spack and several other workflow layers. The pattern is that Git's gaps create opportunities for layered tools rather than for replacing Git.

## When to reach for it; when not to

Reach for Git when:

- The team's contributors expect Git, and the cost of training them on something else exceeds the cost of accepting Git's limitations.
- The hosting story matters and you want any forge in the ecosystem to work.
- The codebase fits Git's design space — primarily text, manageably large, with a workflow that benefits from cheap branching.

Do not reach for Git when:

- The team needs locking on binary assets and Git LFS is not enough. Reach for Perforce or Plastic SCM.
- The codebase is at Google or Meta scale. Reach for Sapling, or for a Mononoke-fronted Git, or accept the cost of partial-clone-plus-sparse-checkout.
- The team's primary contributors are non-developers (artists, technical writers without a CLI background) and the model is too steep. Plastic SCM's Gluon UI or a forge with strong UI investment can help.
- The work is heavily binary and Perforce's pricing is acceptable.

## Epitaph

Git is the version control system that won by combining a clean storage model with an unsurprising-once-you-learn-it CLI built on top, and the field has spent a decade and a half debating which half of that sentence to take seriously.
