# 25. Sapling

## Origin

Sapling is Meta's version control system, open-sourced in November 2022 after roughly a decade of internal development. Inside Meta the system was simply called *Mercurial*, but it had diverged from upstream Mercurial enough by the late 2010s that the same name no longer described the same thing. The 2022 open-source release brought the system out under the name *Sapling* (the command-line tool is `sl`) and made the implementation, design choices, and internal documentation public.

The story behind Sapling is the story of a particular scaling problem. Facebook's monorepo, by 2014, was the largest known production Mercurial deployment in the world. The repository contained source for the Facebook web product, the iOS and Android apps, internal infrastructure, machine learning code, and much else; commit volume was thousands per day; clone size, even with aggressive optimizations, was approaching the limits of what Mercurial's revlog format could handle. Facebook's options were to migrate to Git (which had its own scaling problems and would have been a multi-year company-wide reorganization), to scale Mercurial harder, or to build a new system. They chose to scale Mercurial harder, and the result was Sapling.

The substantial pieces:

- *Mononoke* — the server, a Rust reimplementation that replaced Mercurial's Python server. Mononoke handles repositories far larger than Python Mercurial could; it serves both Mercurial and (via translation) Git protocols.
- *EdenFS* — a virtual filesystem, originally for macOS and Linux, that pages files on demand. A working copy of the Facebook monorepo, with EdenFS, has the appearance of a fully checked-out tree but pages files only when accessed. Cloning is essentially constant time; the cost is paid lazily, on file read.
- *Sapling proper* — the client-side tool, with a UI that looks like Mercurial's at first glance but with significant changes: change-IDs (stable across rewrites), smartlog (the default visualization), absorb (an unusual rewriting tool), stacked-diff workflow as a primary mode.

The open-source release of 2022 made Sapling available outside Meta. The system can run against a Git repository (using a Git-compatible backend), against a Mononoke server, or against a Sapling-native file backend. Adoption outside Meta has been steady but modest. The community is concentrated in teams interested in the workflow improvements (stacked diffs, smartlog) more than in the scale story.

## The system on its own terms

Sapling is, on the surface, a Mercurial-derived DVCS. The core model is changesets in a DAG, with familiar commands (`sl commit`, `sl pull`, `sl push`, `sl log`). The differences become apparent quickly.

*Change identity is stable across rewrites.* Every commit has both a *commit hash* (Git/Mercurial style: hash of the commit's bytes) and a *change ID*: a stable identifier that survives amend, rebase, and other rewrites. When you `sl commit --amend`, the commit hash changes but the change ID does not. Sapling tracks the lineage: this change ID has had three versions, and they are these commits. Tools that operate on change IDs — review tools, stacked-diff workflows, the smartlog — can show "this change in its current form" without needing the user to manage hash transitions.

*Smartlog is the default view.* `sl smartlog` (or `sl sl`) shows a compact, ASCII-art rendering of the changes that matter to you: your local commits, the remote main branch, the relationships between them, with everything else hidden. The view is automatic; the algorithm is opinionated; the output is, for most users, the right answer for "show me what's going on" most of the time. Many Sapling users invoke `sl sl` more often than any other command.

*Stacked diffs are first-class.* The "stacked diff" workflow — a sequence of small, ordered commits, each reviewed independently, all updated together when the bottom of the stack changes — is the dominant code-review workflow at Meta and at several other companies that have adopted it. Sapling's commands assume this workflow: `sl absorb` folds work-in-progress changes into the matching commits in the stack; `sl rebase --restack` updates a stack when its base moves; `sl prev` and `sl next` navigate up and down a stack; the view of "where in the stack am I?" is built into the smartlog.

*Pull is unusually conservative.* `sl pull` updates remote-tracking refs but does not move your working state. The user explicitly opts into rebasing onto the new remote head. The default avoids the surprise where `git pull` might silently merge or rebase depending on configuration.

*Mutable history is acknowledged.* Sapling treats commit-amend, rebase, and other rewrites as normal operations and tracks them as such. The *obsolescence markers* (a concept inherited from Mercurial's `evolve` extension) record that a commit has been replaced by another, with the lineage preserved. Pulling a remote whose stacks have been updated automatically resolves obsolescence; the tool does the bookkeeping.

The CLI is `sl`. Subcommands include `sl commit`, `sl status`, `sl diff`, `sl log`, `sl smartlog`, `sl pull`, `sl push`, `sl rebase`, `sl absorb`, `sl prev`, `sl next`, `sl uncommit`, `sl unamend`, `sl amend`, `sl fold`, `sl split`, `sl undo`. The grammar is closer to Mercurial than to Git.

## Scenario walkthrough

### Operation 1 — Initial import

```
$ cd /path/to/initial/ledger
$ sl init
$ sl add .
$ sl commit -m "Initial import"
```

To publish, configure a remote and push:

```
$ sl push --to remote/main
```

Sapling's relationship to remotes is more explicit than Git's. There is no "push origin"; you push to a named remote bookmark.

### Operation 2 — Linear development

```
$ vi src/ledger/parser.py
$ sl commit -m "parser: handle blank lines and comments"
```

`sl smartlog` after several commits shows the chain compactly:

```
@  4f3a (15 minutes ago) aditi
│  parser: handle blank lines and comments
│
o  3c1e (2 hours ago) aditi
│  Initial import
```

### Operation 3 — Branch

Jonas clones:

```
$ sl clone http://aditi-host/ledger
```

For the `currency-conversion` work, the stacked-diff workflow is more idiomatic than long-running branches. Jonas commits directly:

```
$ vi src/ledger/parser.py
$ sl commit -m "parser: store amounts as (value, currency)"
$ vi src/ledger/reports.py
$ sl commit -m "reports: convert amounts to display currency"
```

This is a stack of two commits sitting on top of `main`. To submit for review, Sapling integrates with Phabricator, ReviewStack, GitHub Pull Requests, or other review systems via plugins; each commit becomes a separate review.

If long-running branches are preferred, `sl bookmark` creates one. The system handles either workflow.

### Operation 4 — File rename

```
$ sl mv src/ledger/reports.py src/ledger/report_balance.py
$ # edit report_balance.py
$ sl add src/ledger/report_register.py
$ sl commit -m "reports: split into balance and register modules"
```

Renames are recorded in commit metadata. `sl log --follow` follows the rename.

### Operation 5 — Binary file added

```
$ sl add docs/logo.png
$ sl commit -m "docs: add project logo"
```

For very large binaries, Sapling's LFS protocol works similarly to Git LFS. With EdenFS as the working copy, large files page on demand and never occupy local disk if the user never reads them.

### Operation 6 — Parallel edits

Aditi and Jonas both edit `README.md`. After Aditi's push, Jonas's push is rejected because the remote head has moved. He runs:

```
$ sl pull
$ sl rebase -d remote/main
... auto-rebases his stack onto Aditi's commit ...
$ sl push --to remote/main
```

The rebase is opinionated: stacks restack automatically; merge commits are unusual.

### Operation 7 — Merge with conflict

For merging the equivalent of `currency-conversion`:

```
$ sl rebase -d remote/main -s currency-conversion-base
... rebases the stack of commits onto main, conflicts on parser.py ...
$ # resolve conflicts in working tree
$ sl resolve --mark src/ledger/parser.py
$ sl continue
```

Sapling's culture (and Meta's) prefers rebase and stacked diffs over merge commits; merge commits are rare in Sapling repositories.

### Operation 8 — Botched commit

This is where Sapling is distinctive. Mireille catches her debug print and typoed message after `sl commit`. Several options:

*Amend in place.* The change ID is preserved; the commit hash changes:

```
$ vi src/ledger/report_register.py
$ sl amend -m "register: add --csv flag"
```

The amended commit replaces the original; the obsolescence marker records the lineage; descendant commits in the stack are automatically restacked.

*Absorb.* If she has already made several follow-up commits and wants to fold a fix into the right one:

```
$ vi src/ledger/report_register.py    # remove debug print
$ sl absorb
... absorbs the change into the commit that introduced the debug print ...
```

`sl absorb` is one of Sapling's distinctive tools. It examines the working tree's uncommitted changes, identifies which commit in the stack each hunk should belong to (based on the line's history), and folds the changes into the matching commits without intervention. Usually it does the right thing; when it doesn't, the system explains and lets the user adjust.

After amend or absorb, `sl push --force-no-replication-needed` or the system's push-with-stack-update updates the remote.

### Operation 9 — Tag

Tags in Sapling are similar to Mercurial's: a versioned tags file and `sl tag`. Tags travel with pulls and pushes.

```
$ sl tag v0.1.0
```

### Operation 10 — Release

```
$ sl archive -r v0.1.0 ledger-0.1.0.tar.gz
```

Standard archive workflow. For Meta-internal release machinery, the release artifact is produced by separate build infrastructure off the tagged commit.

## Model and mental load

What you have to hold:

- Change IDs vs commit hashes. Change IDs are stable; commit hashes are not.
- Smartlog as the dominant view.
- Obsolescence markers, recording rewrites without destruction.
- The stacked-diff workflow, if you adopt it.
- EdenFS as a virtual filesystem, if you use it.

The mental load is real but tractable. Users coming from Git find some commands familiar and some new; the change-ID concept takes a session to internalize and is worth it. Users coming from Mercurial find the system mostly familiar with several improvements.

## Evolution and history rewriting

Sapling is the most permissive system in this book on history mutability, *with* the most rigorous tracking of what was rewritten. Obsolescence markers preserve the lineage; pulling someone else's restacked stack updates your local view of the obsolete-then-replaced commits without losing the connection. Rewriting is normal; the system handles it.

This is, in some ways, the position the patch-theoretic systems were trying to occupy from a different angle. Pijul and Darcs treat patches as identity-bearing objects so that rewrites preserve identity; Sapling and Jujutsu use change-IDs on a snapshot substrate to achieve a similar property. The two paths converge on the same insight: *a change is a thing that exists*, separate from its current bytes, and the version control system should track that.

## Ecosystem reality

Sapling is open-source and actively maintained at Meta. The repository (`facebook/sapling` on GitHub) is the canonical source. External adoption is growing slowly; the documentation is good; the system can drive Git repositories or Sapling repositories with similar UX.

ReviewStack is Meta's open-source review tool that integrates with Sapling stacks. Phabricator, the older tool, is no longer actively developed (Meta deprecated it in 2021), but Phorge — a community-maintained fork — continues. Sapling's GitHub PR integration is functional.

The Mononoke server is open-source but more involved to deploy than client-side Sapling. Most external Sapling users run Sapling against a Git repository on a conventional Git host; the full Sapling experience (Mononoke + EdenFS + Sapling client) is mostly internal to Meta and a few enterprise users with the engineering capacity to deploy it.

## When to reach for it; when not to

Reach for Sapling when:

- You are doing stacked-diff style code review and want first-class tooling for it.
- You value change-ID-stable rewriting and are coming from Git's commit-hash-only model.
- You are working at scale that approaches Mercurial's limits and want a practical migration path that preserves Mercurial-like UX.

Do not reach for Sapling when:

- The team is small, the workflow is straightforward, and Git is sufficient. The investment in learning Sapling is real, and the payoff comes from features the team may not need.
- The hosting story is critical and you do not want to operate a Mononoke server. Running Sapling against a Git host works but loses some of the system's distinctive properties.

## Epitaph

Sapling is the version control system that took everything Meta learned from running Mercurial at impossible scale and turned it into a system that brings the change-ID and stacked-diff workflow to the rest of the field — and is the proof that the snapshot model can grow up to do what the patch theorists wanted, given enough engineering.
