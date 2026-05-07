# 18. Darcs

## Origin

Darcs was started by David Roundy, a physicist at Cornell, in 2002–2003. Roundy wanted a version control system for his own scientific computing work, was unsatisfied with the existing options, and was reading about patch theory. The first public Darcs release was in 2003. Implementation language: Haskell. The choice of Haskell shaped the project — it ensured a small, dedicated, mathematically-inclined contributor base, and it shaped the culture of the project around precise reasoning about operations on patches.

Darcs's animating idea was *patch theory*. The premise is that the right unit of version control is not a snapshot (Git, Mercurial) and not a delta against a parent (RCS, CVS, BK), but a *named patch* — an operation on the working state that has an identity, can be applied or unapplied, has explicit dependencies on other patches, and can in some cases *commute* with other patches: applied in either order, the result is the same.

The consequences are significant. If patches commute when they don't depend on each other, then *history is a partially ordered set of patches*, not a linear sequence and not even a DAG. Cherry-picking from one branch to another is trivial: copy the patch, the patch knows what it depends on, the system applies it. Merging is *just applying the same patch set*: if Aditi's repository has patches `{p, q, r}` and Jonas's has `{p, q, s}`, merging them produces a repository with `{p, q, r, s}`, regardless of the order in which `r` and `s` were originally produced.

The theory was beautiful. The implementation was famously fragile in some cases. Darcs's original commutation algorithm had pathological worst cases — the *exponential merge* problem — where certain combinations of patches forced the system into combinatorial explosions. Working Darcs users developed habits to avoid the bad cases; Darcs 2 (2008) introduced a new patch type that avoided the worst behavior; the issue continued to color Darcs's reputation.

Darcs had a real community in the late 2000s. The Glasgow Haskell Compiler used it as its primary version control system through 2013, when GHC migrated to Git. Several Haskell libraries used it. The system was known for the elegance of its model and the eccentricity of its CLI vocabulary.

In 2026, Darcs is alive but small. The current version is 2.18 (released 2024). The user base is concentrated in functional programming circles. New adoption is rare. Pijul (next chapter) is widely understood as the spiritual successor that worked out the theoretical issues and produced a practically deployable equivalent.

## The system on its own terms

A Darcs *repository* is a directory containing `_darcs/`, which holds the patch set. The patches are stored individually, with names, dependencies, and the actual operations. There is no equivalent of a commit DAG; there is a *patch order* — the order in which patches were applied — and a *dependency relation* — which patches depend on which others.

A *patch* is a named operation. Common patch types:

- *Hunk* patches: insert or delete lines at a specific position in a specific file.
- *Add file* and *remove file* patches.
- *Rename* patches: a first-class operation, not a delete-plus-add.
- *Set token* patches: change a specific identifier (somewhat like a controlled find-and-replace).
- *Merger* patches: produced when two conflicting patches must be reconciled; encode the resolution as part of the patch chain.

Each patch has a unique hash, a name (the user's commit message), an author, and a timestamp. The patch's *identity* is intrinsic: the same patch applied to different ancestor states is the same patch. This is what makes cherry-picking work: a patch can be pulled from one repository to another with no concept of "rebase" — it just gets applied, with its dependencies pulled in if needed.

Two patches *commute* if applying them in either order produces the same result. Hunks at different positions in different files commute trivially. Hunks in the same file at non-overlapping positions commute. The system's job, on operations like pull and merge, is to compute commutations and conflicts: which patches can be applied independently, which must be applied in a specific order, which conflict and require human resolution.

The CLI is `darcs`. Subcommands: `darcs init`, `darcs add`, `darcs record` (the patch-creation command, equivalent to commit), `darcs pull`, `darcs push`, `darcs send` (mails a patch bundle to a recipient), `darcs apply` (applies a received bundle), `darcs unrecord` (removes a patch locally), `darcs obliterate` (removes a patch and its dependents, with confirmation), `darcs amend-record` (adds further changes to a previously recorded patch), `darcs changes` (shows patch history), `darcs log`, `darcs diff`, `darcs annotate`.

The `darcs send` workflow is one of Darcs's distinctive features. A patch can be packaged as a self-contained bundle and emailed to a maintainer; the maintainer runs `darcs apply` to integrate. This is the *patch-by-email* workflow that Linux kernel developers used for decades, made first-class.

## Scenario walkthrough

### Operation 1 — Initial import

```
$ cd /path/to/initial/ledger
$ darcs init
$ darcs add .
$ darcs record -a -m "Initial import"
```

The first patch records the initial state; it has no parent in the patch order. Subsequent patches depend on it, transitively, until subsequent unrelated patches are recorded.

### Operation 2 — Linear development

```
$ vi src/ledger/parser.py
$ darcs record -a -m "parser: handle blank lines and comments"
```

`darcs record` interactively prompts for which changes to include in the patch (similar to `git add -p`); `-a` accepts all changes. The patch is added to the repository's patch set.

### Operation 3 — Branch

In Darcs, "branch" is conventionally a separate clone:

```
$ darcs clone . ../ledger-cc
$ cd ../ledger-cc
$ vi src/ledger/parser.py
$ darcs record -a -m "parser: store amounts as (value, currency)"
```

The clone is independent; both repositories evolve their own patch sets. To "merge a branch back," you `darcs pull` from the branch into the trunk.

### Operation 4 — File rename

```
$ darcs move src/ledger/reports.py src/ledger/report_balance.py
$ # edit report_balance.py
$ darcs add src/ledger/report_register.py
$ darcs record -a -m "reports: split into balance and register modules"
```

The rename is recorded as a `move` patch — a first-class operation. Subsequent patches that touch the renamed file commute correctly with the rename, because the rename patch is explicit about what it does.

### Operation 5 — Binary file added

```
$ darcs add docs/logo.png    # darcs detects binary
$ darcs record -a -m "docs: add project logo"
```

Darcs identifies binary files and stores them as opaque add patches. Subsequent edits to a binary are stored as full replace patches; there is no useful delta encoding for binaries.

### Operation 6 — Parallel edits

Aditi and Jonas both edit `README.md`. They `darcs record` independently in their own clones. When Jonas pulls Aditi's patches:

```
$ darcs pull
... shows Aditi's patches, asks which to apply ...
... applies them, computing commutations with Jonas's existing patches ...
```

If the patches commute (different lines, different files), they apply silently. If they conflict, Darcs records a *merger patch* that captures the resolution.

### Operation 7 — Merge with conflict

To merge `currency-conversion` back into `main`:

```
$ cd /path/to/main
$ darcs pull /path/to/ledger-cc
... lists the patches on the branch, prompts to apply ...
```

The system computes commutations with the local patch set. For the `parser.py` conflict (where the branch and main both modified the same hunk), Darcs reports the conflict, applies a merger patch, and lets the user resolve. The resolution becomes a new patch.

The result: the merged repository has *all the patches from both lines*, in some order, with the relevant merger patches recording resolutions. There is no merge commit; there is a patch set that includes everything.

### Operation 8 — Botched commit

Darcs handles this trivially. To remove the bad patch:

```
$ darcs unrecord
... interactively prompts for which patch to unrecord ...
... removes the patch from the repository, leaving its changes in the working tree ...
$ # fix the working tree
$ darcs record -a -m "register: add --csv flag"
```

`darcs unrecord` removes a patch from the repository while keeping its effect in the working tree, ready to be re-recorded. `darcs amend-record` adds more changes to a previously recorded patch, replacing it.

The key property: because patches have intrinsic identity, removing and re-recording a patch produces a *different patch* with a different name. If the bad patch had been pushed to a remote, removing it locally does not remove it remotely. To genuinely retract a published patch, the system has `darcs obliterate`, which is the more aggressive surgical operation.

### Operation 9 — Tag

```
$ darcs tag v0.1.0
```

Tags in Darcs are *meta-patches* that depend on every patch in the current state, naming a specific moment in the patch graph. They are first-class patches and travel with the patch set on pull and push.

### Operation 10 — Release

```
$ darcs dist -d ledger-0.1.0
```

`darcs dist` produces a tarball of the current state; with `-d`, it names it. The release notes are committed (recorded) before the tag.

## Model and mental load

What you have to hold:

- Patches as named, identity-bearing objects.
- The patch set as a partially ordered set, not a linear sequence.
- Commutation: when two patches are independent and can be applied in any order.
- Conflicts as the failure of patches to commute.
- The clone-as-branch convention.
- The send/apply workflow for patch-by-email.

The mental load is moderate. The model is internally clean and rewards study. New users without exposure to patch theory take some time to internalize what makes patches different from commits; users with the background find the system delightful. The CLI's idiosyncrasies — `darcs record` for commit, `darcs send` for email, `darcs obliterate` for surgical removal — are quirks that long-time users find charming.

## Evolution and history rewriting

`darcs unrecord` and `darcs amend-record` are the local rewriting tools. `darcs obliterate` is the more aggressive option. The system's stance is permissive within local repositories, conservative across published ones; remote patches retain their identity.

The exponential-merge problem in Darcs 1 was the system's defining technical embarrassment for years. Darcs 2 introduced *RepoFormat: Darcs-2*, with a new merge algorithm that handled the previously pathological cases reasonably. Darcs 2 has been the default for over a decade; the problem is, in practice, not encountered in modern usage.

## Ecosystem reality

Darcs in 2026 is a small but real project. The current version (2.18) builds with modern GHC, ships through Hackage, runs on Linux, macOS, and Windows. The mailing list has occasional traffic. The website hosts a Hub-like repository hosting service (`hub.darcs.net`) where small projects continue to run.

Migration to and from Git exists; tools are crude but functional. The community continues to use Darcs primarily for projects where the model genuinely matches the work — small libraries, individual researchers, projects where cherry-picking and rename tracking are valued.

## When to reach for it; when not to

Reach for Darcs when:

- You want to learn what patch theory feels like in a working system, and Pijul (next chapter) is too young for your tolerance.
- You have a small project where the patch-by-email workflow matches your collaborators' habits.
- You value first-class renames and free cherry-picking and the exponential-merge corner cases are not in your path.

Do not reach for Darcs when:

- You need the network effects of Git's hosting and tooling ecosystem.
- Your team is large and turnover means new contributors will need to learn the system.
- You are doing very large or very binary-heavy work; Darcs is small-project optimized.

For most readers, Pijul is the correct choice over Darcs in 2026 if the patch-theory model attracts you; Pijul has worked out the theoretical issues and is being more actively developed.

## Epitaph

Darcs was the version control system that took patches seriously as objects and built a beautiful theory on them, and is the system whose ideas are kept alive in Pijul because Darcs's implementation was not quite the answer the field needed.
