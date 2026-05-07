# 26. Jujutsu

## Origin

Jujutsu — `jj` at the command line — was started by Martin von Zweigbergk at Google around 2019, with the first public commits appearing in 2020. Von Zweigbergk had previously worked on Mercurial at Google and had encountered a similar problem to the one that produced Sapling at Meta: at scale, operating in a Git-shaped world but wanting better workflow primitives, what would the right design be? Sapling and Jujutsu are the two answers, developed roughly in parallel at the two companies, with overlapping but distinct design choices. The Jujutsu repository is publicly hosted at `github.com/jj-vcs/jj` (formerly `martinvonz/jj`), under an Apache 2 license.

The system is the youngest serious version control project in this book. Adoption is growing visibly: in 2025 the user base outside Google was small but enthusiastic; through 2025 and into 2026 the project has gathered momentum, with substantial discussion in the technical community, an active community on Discord and GitHub, and a growing list of tools that explicitly support it. As of mid-2026, Jujutsu is the project the book most strongly recommends for readers who want to *experiment* with a different version control workflow without leaving the Git ecosystem.

The reason for the recommendation is straightforward: Jujutsu is *backed by Git*. The default backend stores commits in a `.git/` directory; remote operations use Git's protocols; pushes go to GitHub or any Git host without translation. A Jujutsu user can collaborate with Git users transparently. The choice to embrace Git as substrate, rather than to replace it, is what makes Jujutsu deployable to a team in 2026 in a way Pijul or Sapling-with-Mononoke cannot match.

The interesting parts of Jujutsu are the parts that are *not* Git: the change-ID model, the operation log, the absence of a staging area, first-class conflicts, the working-copy-is-a-commit model, and the auto-rebase semantics. Each is a deliberate departure from Git's defaults, and each is the visible argument that the snapshot model can host workflow primitives Git was not designed for.

## The system on its own terms

A Jujutsu working copy is a directory containing a `.jj/` subdirectory and (if using the Git backend) a `.git/` directory. The Git directory holds Git's standard objects; the JJ directory holds JJ's metadata: the operation log, change-ID mappings, working-copy commits.

The model has a few pieces that take a paragraph each to explain.

*The working copy is a commit.* In Git, the working copy is a not-yet-committed staging area; in Mercurial and Sapling, it's a candidate for the next commit. In Jujutsu, the working copy *is* a commit — a commit that the system updates automatically as you edit files. Save a file, and the working-copy commit's content updates. There is no `jj add`; there is no staging area. The commit is real, has a hash, has a change ID, and can be referenced by other commands.

The working-copy commit is initially a *child* of whatever commit you started from, with no description. To "commit" in the traditional sense, you describe the working-copy commit (`jj describe -m "..."`) and then create a new working-copy commit on top of it (`jj new`). The new commit is empty until you start editing.

The model has a startling consequence: the entire history, including the work you have not "committed" yet, is part of the same graph. There is no "uncommitted changes that don't exist as commits" state. Every change you have made is in some commit; you can refer to it, rebase it, split it, fold it into a different commit. The dichotomy between "saved" and "not saved" disappears.

*Change IDs are stable across rewrites.* As in Sapling, every commit has a change ID separate from its commit hash. Amending the commit changes the hash; the change ID is preserved. Tools and references that work with change IDs continue to work after rewrites without explicit updates.

*Conflicts are first-class objects in commits.* A commit can be *in conflict* — that is, contain conflict markers in some files — and still exist as a normal commit in the graph. You can rebase a stack onto a different base, get conflicts in three intermediate commits, and the system records the conflicts inside those commits. You can fix the conflicts later, in any order, and the descendants update.

This is genuinely novel for a snapshot system. In Git, a conflict during rebase halts the operation; you must resolve before continuing. In Jujutsu, the rebase completes; the conflicted state is preserved in the affected commits; you resolve when convenient. The system does not lose your place; the conflicted commits are visible in the log; you address them as work items.

*The operation log records every operation, with replay.* Every command that mutates state — commit, rebase, abandon, squash, anything — is recorded in the *operation log*. `jj op log` shows the history of operations. `jj op restore` rolls back to a prior operation; `jj op undo` reverses the most recent operation. The semantics are richer than Git's reflog: where the reflog tracks ref movements, the operation log tracks *every operation* as a structured object, with the ability to undo arbitrary edits to history.

*Auto-rebase of descendants on amend.* When you amend a commit, all descendants automatically rebase onto the amended commit. The user does not run `git rebase --onto`; the system does it. The change IDs are preserved; descendants are now slightly different commits (different hashes) representing the same logical changes on top of the new parent.

*Anonymous branches.* Every commit is implicitly a branch tip. The system does not require named branches for working with parallel lines of development; you can have ten "branches" with no names, navigate between them with `jj edit COMMIT`, and the system tracks them through their change IDs. Named bookmarks (`jj bookmark`) are available when you want to pin a name, and integrate with Git refs for push and pull.

The CLI is `jj`. Subcommands include `jj init`, `jj new`, `jj describe`, `jj commit` (a shortcut combining describe and new), `jj log`, `jj st` (status), `jj diff`, `jj abandon`, `jj squash`, `jj split`, `jj rebase`, `jj duplicate`, `jj edit`, `jj git push`, `jj git fetch`, `jj op log`, `jj op restore`, `jj resolve`. The grammar is consistent.

## Scenario walkthrough

### Operation 1 — Initial import

```
$ cd /path/to/initial/ledger
$ jj git init --colocate    # creates .jj/ and .git/ in same directory
$ jj describe -m "Initial import"
... working-copy commit (which contains all the imported files) is described ...
$ jj new
... starts a new working-copy commit on top of the import commit ...
```

The first commit's working-copy state has been described; a new working-copy commit has been opened for the next round of work. To publish to a Git remote:

```
$ jj git remote add origin git@github.com:cloudstreet-dev/ledger.git
$ jj bookmark create main -r @-    # name the parent commit
$ jj git push --bookmark main
```

### Operation 2 — Linear development

```
$ vi src/ledger/parser.py
$ jj describe -m "parser: handle blank lines and comments"
$ jj new
```

Or, more concisely:

```
$ vi src/ledger/parser.py
$ jj commit -m "parser: handle blank lines and comments"   # describe + new in one step
```

Each cycle: edit, commit, automatically have a fresh working-copy commit ready.

### Operation 3 — Branch

Jonas clones:

```
$ jj git clone git@github.com:cloudstreet-dev/ledger.git
```

For the `currency-conversion` work, no explicit "branch" command is needed. Jonas just commits:

```
$ vi src/ledger/parser.py
$ jj commit -m "parser: store amounts as (value, currency)"
$ vi src/ledger/reports.py
$ jj commit -m "reports: convert amounts to display currency"
```

The two commits form a stack on top of the `main` bookmark. To name the stack as a branch:

```
$ jj bookmark create currency-conversion
$ jj git push --bookmark currency-conversion
```

The bookmark is exposed to Git as a branch.

### Operation 4 — File rename

```
$ jj mv src/ledger/reports.py src/ledger/report_balance.py
$ # edit report_balance.py
$ # create report_register.py
$ jj describe -m "reports: split into balance and register modules"
$ jj new
```

JJ handles renames via Git's underlying rename detection (since the backend is Git) plus its own metadata. `jj log --follow` follows files across renames.

### Operation 5 — Binary file added

```
$ # add docs/logo.png to working tree
$ jj describe -m "docs: add project logo"
$ jj new
```

Storage is Git's blob store; JJ does not add a separate binary handling layer. For very large binaries, Git LFS works through JJ since the backend is Git.

### Operation 6 — Parallel edits

Aditi and Jonas both edit `README.md`. Aditi pushes:

```
$ jj describe -m "README: add Quick Start"
$ jj git push
```

Jonas, after fetching, has his commit and Aditi's both visible in the log. To rebase his work on top of Aditi's:

```
$ jj git fetch
$ jj rebase -d main@origin
... rebases his stack onto the new main ...
```

If conflicts arise during the rebase, the conflicted commits remain in the graph; he resolves them at his convenience using `jj resolve`.

### Operation 7 — Merge with conflict

Merging the `currency-conversion` stack:

```
$ jj rebase -s currency-conversion-base -d main
... attempts to rebase the stack onto main ...
... detects conflicts in parser.py ...
... rebase completes; conflicted commit is in the graph ...
$ jj log
... shows the rebased stack with one commit marked as conflicted ...
$ jj edit CONFLICT-COMMIT
$ jj resolve
... opens a merge tool, lets the user resolve, automatically updates the commit ...
```

The merge does not produce a separate merge commit; it produces a rebased stack on top of main. For users who want explicit merges, `jj new -m "merge" PARENT1 PARENT2` creates a merge commit.

### Operation 8 — Botched commit

This is where Jujutsu's model is most distinctive. Mireille catches her debug print and typoed message:

```
$ jj describe -m "register: add --csv flag"     # fixes the message
$ vi src/ledger/report_register.py    # remove debug line
... working-copy commit auto-updates to include the fix ...
$ jj squash    # folds the working-copy commit's changes into its parent
... the parent commit (the botched one) is updated to the corrected content ...
... since the change ID is stable, descendants (if any) auto-rebase ...
```

The botched commit's hash changes; the change ID is preserved; the operation log records the rewrite. If she changes her mind:

```
$ jj op log
... shows recent operations ...
$ jj op restore @-   # restore to before the squash
```

The botched commit is back, with the same content it had before the squash. The system supports unbounded undo through the operation log.

### Operation 9 — Tag

```
$ jj bookmark create v0.1.0
$ jj git push --bookmark v0.1.0
```

Jujutsu uses bookmarks for both branches and tags. For Git-side tags specifically, `jj git push` with appropriate flags creates Git tags.

### Operation 10 — Release

```
$ jj git fetch
$ git archive --format=tar.gz --prefix=ledger-0.1.0/ v0.1.0 -o ledger-0.1.0.tar.gz
```

Since the backend is Git, standard Git release workflows work. `jj` does not need its own archive command; Git's suffices.

## Model and mental load

What you have to hold:

- Working-copy-is-a-commit. Edits go into a real commit automatically.
- Change IDs vs commit hashes. Change IDs are stable; commit hashes change on rewrite.
- The operation log. Every operation is undoable.
- First-class conflicts. Conflicted commits are normal commits; resolve at your convenience.
- Auto-rebase of descendants. Amending a commit shifts its descendants automatically.
- Anonymous branches. Bookmarks are optional; commits exist as branch tips by default.

The mental load is real but the cognitive savings are also real. Users coming from Git find some operations strange (no `git add`?) and others delightful (undo any operation). The unfamiliarity is concentrated in the first day; after a week, the system is more comfortable than Git for most operations.

## Evolution and history rewriting

Jujutsu is the most permissive system in this book on history mutability, with the most rigorous tracking of what was rewritten. The operation log is a complete record; restoration is an operation, not a recovery. Change IDs preserve identity through rewrites; auto-rebase keeps descendants consistent.

This is the position the patch-theoretic systems wanted to occupy. Jujutsu reaches it on a snapshot substrate by treating change IDs as patches' identity-bearing analogues, by treating conflicts as first-class commit content, and by recording operations as structured objects. The result is a system that reads like patch theory in spirit while being implemented like Git in practice.

## Ecosystem reality

Jujutsu's ecosystem is small but active. The reference client is the only client. Editor and IDE plugins are minimal but growing. CI/CD systems work because the Git backend works; the JJ-specific operations live on the developer's local machine. The community is concentrated on Discord and GitHub; the documentation is good and improving.

Tools that explicitly support JJ as more than a Git wrapper are rare in 2026 but increasing. JJ-aware review tools, smartlog-style visualizations, and editor integrations are visibly under development. Adoption inside Google has grown to a meaningful internal user base. Adoption outside Google is concentrated in technically curious teams that want change-IDs and the operation log without leaving Git.

## When to reach for it; when not to

Reach for Jujutsu when:

- You are willing to invest a week in learning a new model and want change-ID rewriting, the operation log, and first-class conflicts.
- The team's hosting is Git-based and you want to keep it that way.
- You want to explore the workflow improvements without committing the team to a wholesale migration.

Do not reach for Jujutsu when:

- The team needs absolute stability and conservatism. JJ is young; releases sometimes change behavior; bugs exist. Production-critical workflows should still use Git directly.
- The hosting story requires non-Git protocols. JJ is Git-backed; non-Git use cases are not its primary target.
- The team is small, the workflow is straightforward, and the marginal benefit doesn't justify the learning investment.

## Epitaph

Jujutsu is the version control system that brings the patch theorists' insights — change identity, structural rewrites, conflicts as objects — onto the Git substrate the field already runs on, and is the best evidence in this book that the design space is still open in 2026.
