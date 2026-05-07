# 21. Plastic SCM

## Origin

Plastic SCM was developed by Codice Software, a small company founded in Valencia, Spain by Pablo Santos and a small team in 2005. The first 1.0 release shipped in 2007. Unity Technologies acquired Codice Software in 2020, and the system has since been rebranded as *Unity Version Control* (UVCS), though the legacy name *Plastic SCM* and the `cm` command-line tool persist; the chapter uses the legacy name throughout, with the understanding that the current product is Unity Version Control.

The system was built to address a problem the founders saw clearly: game development teams in Spain and Europe were caught between Git's poor handling of binary content and Perforce's expensive, server-heavy model. Codice's pitch was a system that could do *both* — work like a DVCS for code where DVCS made sense, work like a centralized server for binary assets where centralization made sense, with first-class branching, smart locking, and a graphical workflow accessible to non-developers.

The product matured through the 2010s. Distinctive features arrived steadily: the *Branch Explorer* (a graphical view of the branch graph that became the system's signature), the *branch-per-task* workflow, *Gluon* (a simplified UI for artists who do not want to think in branches), *semantic merge* for some languages (structural rather than textual), *smart locking* on binary files, and tight integrations with Unity and Unreal Engine.

After Unity's acquisition, the system was tied more closely to Unity's tooling. Unity Version Control today is sold both as part of Unity Plus/Pro/Industry subscriptions and as a standalone product. The user base is concentrated in game development, in Unity-using teams especially, and in some film and visualization studios. It is also used by software teams that want a non-Git option that is more modern than Subversion and less expensive than Perforce.

## The system on its own terms

Plastic's data model is closer to Git's than to Perforce's: changesets in a DAG, content-addressed storage, distributed semantics. The differences are in workflow primitives and in the substantial UI investment.

A *workspace* is the developer's working tree. Workspaces are switchable between branches with `cm switch`, and the system tracks workspace state on the server (when in centralized mode) or locally (when distributed). A workspace is a working tree at a particular changeset on a particular branch.

A *branch* is a first-class object with a parent, a child set, and a label. Branches in Plastic are *the* unit of work: the dominant idiom is *branch-per-task*, where every feature, bug fix, or experiment lives on its own branch, and the branch is named after the task. Codice's branding around this — "task branches," "task-branching workflow" — became part of the product's pitch.

The *Branch Explorer* is a graphical view of the branch DAG. It is the tool users open most often. The view shows branches as horizontal swim-lanes, changesets as nodes, merges as connecting lines; it lets users select changesets, branches, or merges and operate on them through a context menu. The Branch Explorer is, by some accounts, the best graphical version control UI in any system in this book. It is the part Git has not quite produced an equivalent of after fifteen years of forge investment.

*Smart locking* is a per-file mechanism for binary content. The team configures a *lock policy* (typically by file extension: `.png`, `.psd`, `.fbx`, `.uasset`, etc.) and the server enforces locks on those files: a file matching the policy can only be edited by the user who has acquired the lock. This is the Perforce `+l` model recreated in Plastic, with the configuration on the server rather than per-file.

*Semantic merge* is a separate product in the Plastic family (and standalone — `semanticmerge` is sold as its own tool that integrates with Plastic, Git, and others). It performs language-aware merging for C#, Java, C++, Visual Basic, and a few others: where a textual three-way merge would produce a conflict because two developers reorganized the same file, semantic merge reasons about the structure (which functions moved where, which class changed) and produces a clean merge in many cases that line-based merge cannot.

*Gluon* is a separate UI mode targeting non-developer team members — artists, writers, designers — who want to use version control without thinking about branches and merges. Gluon presents a simpler model: pick a project, pick files to work on, lock them, edit, submit. The branches and merges happen behind the scenes; the user sees a workflow that resembles a shared file system with checkout-style coordination.

The CLI is `cm`. Subcommands include `cm setup`, `cm workspace`, `cm checkout`, `cm checkin`, `cm branch`, `cm switch`, `cm merge`, `cm find`, `cm log`, `cm diff`, `cm cat`. The grammar is verbose; most users prefer the GUI for daily work.

## Scenario walkthrough

### Operation 1 — Initial import

Aditi sets up a Plastic server (or uses the cloud-hosted option) and creates a repository:

```
$ cm repo create ledger@aditi-server
$ cm workspace create ledger /Users/aditi/work/ledger ledger@aditi-server
$ cd /Users/aditi/work/ledger
$ # populate working tree
$ cm add -R .
$ cm checkin -c "Initial import"
```

The first changeset is created on the default branch (`/main` in Plastic's branch namespace).

### Operation 2 — Linear development

For the branch-per-task idiom, *every* commit goes on its own branch. The exemplar's "linear development" maps onto three task branches:

```
$ cm branch /main/parser-blank-lines -c "parser: blank lines and comments"
$ cm switch /main/parser-blank-lines
$ vi src/ledger/parser.py
$ cm checkin -c "parser: handle blank lines and comments"
$ cm switch /main
$ cm merge /main/parser-blank-lines
$ cm checkin -c "Merge parser-blank-lines"
```

Each task branch is short-lived: branch off main, do the work, merge back, repeat. The Branch Explorer shows the project's history as a series of small branch-off-and-merge-back loops.

For projects that prefer linear main, the team can also commit directly to main; the branch-per-task workflow is idiomatic but not enforced.

### Operation 3 — Branch

For Jonas's `currency-conversion` branch:

```
$ cm switch /main
$ cm branch /main/currency-conversion -c "currency conversion feature"
$ cm switch /main/currency-conversion
$ # work happens here over multiple changesets
```

The branch is a first-class object. `cm branches` lists it; the Branch Explorer shows it as a parallel swim-lane.

### Operation 4 — File rename

```
$ cm checkout src/ledger
$ cm move src/ledger/reports.py src/ledger/report_balance.py
$ # edit report_balance.py
$ cm add src/ledger/report_register.py
$ cm checkin -c "reports: split into balance and register modules"
```

Renames are first-class. `cm history --follow src/ledger/report_balance.py` follows the rename.

### Operation 5 — Binary file added

The lock policy is configured server-side:

```
$ cm lockrules edit ledger@aditi-server
... edit the ruleset to add: lock(*.png, *.psd, *.fbx) ...
```

Mireille adds the logo:

```
$ cm add docs/logo.png
$ cm checkin -c "docs: add project logo"
```

The file is added; subsequent edits will require lock acquisition. Storage is the Plastic backend (SQL Server, MySQL, or Plastic's own jet-based engine), which handles binaries with deduplication and reasonable compression.

### Operation 6 — Parallel edits

Aditi and Jonas both edit `README.md`. With smart locking *off* for `.md` (the default for text files), they both `cm checkout`, edit, and `cm checkin`. The first to check in succeeds; the second receives a need-to-merge signal and runs `cm merge` to integrate.

For binaries, smart locking would prevent the parallel-edit case entirely: only one user can hold the lock at a time.

### Operation 7 — Merge with conflict

```
$ cm switch /main
$ cm merge /main/currency-conversion
... reports auto-merges and conflicts ...
$ # for languages with semanticmerge available, semantic merge runs first
$ # remaining conflicts open in the merge tool
$ cm checkin -c "Merge currency-conversion into main"
```

The merge changeset has two parents in the DAG. The Branch Explorer displays the merge graphically.

### Operation 8 — Botched commit

Plastic supports rewriting unreleased changesets:

```
$ cm undo --keepchanges /main:42    # undoes changeset 42, keeps changes in workspace
$ # fix the file
$ cm checkin -c "register: add --csv flag"
```

Or, more commonly, a follow-up commit corrects the issue. Plastic's stance on history is moderate: rewriting is supported with appropriate warnings, but the cultural default is forward correction.

### Operation 9 — Tag

Plastic uses *labels* applied to changesets:

```
$ cm label create v0.1.0 -c "First public release"
$ cm label apply v0.1.0 cs:42@ledger@aditi-server
```

Labels are first-class; they can be applied retroactively, moved, deleted (with audit logging).

### Operation 10 — Release

```
$ cm patch create label:v0.1.0 -d ledger-0.1.0.patch
$ # or, for a clean tarball:
$ cm getrevision label:v0.1.0 /tmp/release
$ tar czf ledger-0.1.0.tar.gz -C /tmp release
```

The release is conventionally tagged, the workspace switched to the label, the tarball produced. Unity Version Control integrates the release artifact handling with Unity Cloud Build for game projects.

## Model and mental load

What you have to hold:

- Workspaces with server-tracked state (in centralized mode) or local state.
- Branches as first-class objects, with branch-per-task as the idiomatic workflow.
- The Branch Explorer as the primary way to see project state.
- Smart locking via server-configured rules, not per-file.
- Gluon for artists; standard `cm` for developers.
- Labels as first-class.

The mental load is moderate. The system is more discoverable than Git for new users with the Branch Explorer; the CLI is verbose but consistent. Gluon makes the system accessible to artists who do not want to learn a CLI at all.

## Evolution and history rewriting

Plastic supports `cm undo` for unreleased changesets and various server-side operations for administrators. The branch-per-task workflow makes most rewriting unnecessary: bad work lives on a discardable branch and never reaches main.

## Ecosystem reality

Plastic SCM / Unity Version Control in 2026 is a current product. Unity bundles it; standalone licenses are sold; cloud hosting is offered. The user base is significant in game development, modest in other industries. Unity's marketing has integrated the product more tightly with the engine over the years; for non-Unity users, the standalone product remains available but the messaging is increasingly Unity-centric.

The system competes directly with Perforce for the game-development market and has gained share in the mid-market. For very large game studios with established Perforce deployments, Perforce typically still wins on installed-base inertia and tooling integration; for new and growing studios, Plastic / UVCS is often the better fit.

The Sourcegear/Unity ecosystem of tools — semanticmerge, the Plastic CLI, the GUI client, Gluon — is mature. CI/CD integrations exist; Unity Cloud Build is the canonical pipeline for game projects.

## When to reach for it; when not to

Reach for Plastic SCM / Unity Version Control when:

- You are building a game in Unity. The integration is best-of-class.
- You have a mixed team of developers and artists, and Gluon's UI is what your artists need.
- You want the locking-on-binaries model without Perforce's enterprise overhead.
- The Branch Explorer's visualization solves a real problem for your team.

Do not reach for it when:

- The team is fully developer, fully Git-fluent, and has no binary content.
- The hosting story matters more than the in-product experience: GitHub-style external collaboration is not Plastic's strength.
- The codebase is very text-heavy and the team is happy with Git's pull-request workflow.

## Epitaph

Plastic SCM is the version control system that took the design space's centralized/distributed and developer/non-developer axes seriously, built a product that handled both ends of each, and is the best evidence in this book that thoughtful UI investment can make the difference between a system that the field uses and a system the field merely respects.
