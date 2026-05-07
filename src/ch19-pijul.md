# 19. Pijul

## Origin

Pijul was started by Pierre-Étienne Meunier around 2014 as an attempt to build a version control system on a *sound* theoretical foundation for patches — one that would deliver Darcs's expressiveness without Darcs's pathological merge cases. Meunier and several collaborators (Florent Becker is named alongside him on much of the early work) drew on category theory: specifically, on the observation that the right way to think about a merge of two divergent histories is as a *pushout* in a category whose objects are repository states and whose morphisms are patches. The resulting model, formalized in several papers and informal write-ups, gives Pijul a property Darcs lacked: well-defined behavior on every merge case, with no exponential blowups, because the operation is grounded in a structure that mathematics has worked out.

Pijul is written in Rust. The first usable release was around 2017. The system has been pre-1.0 for years; current releases (as of 2026) are in the 1.0-beta range, with the project still moving toward a final 1.0. The implementation has matured substantially over the last few years. The hosted forge for Pijul, *the Nest* (`nest.pijul.com`), is small but functional. The user base is concentrated in functional programming and theoretical-computer-science adjacent communities.

The chapter covers Pijul as the working manifestation of patch theory in 2026. Where Darcs preserved the model in legacy form, Pijul is the active investigation of what the model actually wants to be when given modern engineering and a clear mathematical foundation.

## The system on its own terms

A Pijul repository is a directory containing a `.pijul/` subdirectory, which holds the patch store and metadata. The storage backend is *sanakirja*, Meunier's own copy-on-write key-value store, designed to give Pijul ACID semantics with append-only persistence (so the system can crash mid-operation without corrupting state).

A *change* in Pijul (the term Pijul uses where Darcs says "patch") is a named operation. A change has a hash, a name (the user's commit message), an author, a timestamp, and an explicit list of dependencies — other changes the current change depends on. The dependency list is *computed*, not declared: the system determines which earlier changes must be applied before this one can be coherent, by examining what the change touches.

The patch model is more refined than Darcs's. Where Darcs has hunk patches that can fail to commute on subtle textual interactions, Pijul represents file content as a graph of *line nodes* with edges representing structural relationships, and patches as graph operations. The graph representation lets Pijul reason about line identity across edits in a way pure-text-diff systems cannot.

Channels are Pijul's branches. A channel is a named view of a subset of the patch store; switching channels is fast because patches are not duplicated, only the channel's set of applied patches changes. Channels can be created, deleted, listed, switched between, and pushed/pulled selectively.

The CLI is `pijul`. Subcommands: `pijul init`, `pijul add`, `pijul record`, `pijul push`, `pijul pull`, `pijul fork` (create a channel), `pijul switch`, `pijul log`, `pijul reset`, `pijul rollback`, `pijul unrecord`, `pijul change` (show a change's content). The grammar is more conventional than Darcs's, with concessions to Git users coming from elsewhere.

## Scenario walkthrough

### Operation 1 — Initial import

```
$ cd /path/to/initial/ledger
$ pijul init
$ pijul add --recursive .
$ pijul record -m "Initial import"
```

The first change is recorded. To publish to the Nest:

```
$ pijul push aditi@nest.pijul.com:ledger
```

Or to a self-hosted Pijul server.

### Operation 2 — Linear development

```
$ vi src/ledger/parser.py
$ pijul record -m "parser: handle blank lines and comments"
```

Each `record` produces a change. The change's dependencies are computed from what it touches.

### Operation 3 — Branch

Jonas clones:

```
$ pijul clone aditi@nest.pijul.com:ledger
$ cd ledger
```

For the `currency-conversion` work, he forks a channel:

```
$ pijul fork currency-conversion
$ pijul switch currency-conversion
$ vi src/ledger/parser.py
$ pijul record -m "parser: store amounts as (value, currency)"
```

Channels are first-class. To push the channel:

```
$ pijul push --to-channel currency-conversion
```

### Operation 4 — File rename

```
$ pijul mv src/ledger/reports.py src/ledger/report_balance.py
$ # edit report_balance.py
$ pijul add src/ledger/report_register.py
$ pijul record -m "reports: split into balance and register modules"
```

Renames are first-class changes. The line-graph model preserves line identity across the rename, so subsequent edits commute correctly with the rename and blame works through it.

### Operation 5 — Binary file added

```
$ pijul add docs/logo.png
$ pijul record -m "docs: add project logo"
```

Pijul detects binary content and stores it as opaque add-and-replace operations. Storage uses sanakirja's facilities. As with most systems, binary files are not delta-compressed meaningfully.

### Operation 6 — Parallel edits

Aditi and Jonas both edit `README.md`. They `pijul record` independently. When pulling each other's changes, Pijul computes commutations:

```
$ pijul pull
... applies non-conflicting changes ...
... reports conflicts where commutation fails ...
```

Conflicts in Pijul are first-class objects, recorded in the repository state, and resolved by recording a *resolution* change that depends on the conflicting changes. The resolution is a normal change; it can be pushed and pulled like any other.

### Operation 7 — Merge with conflict

To merge `currency-conversion` into the main channel:

```
$ pijul switch main
$ pijul pull --from-channel currency-conversion
... applies the channel's changes, computing commutations and conflicts ...
$ # resolve conflict in parser.py
$ pijul record -m "Merge currency-conversion: resolve parser conflict"
```

There is no merge commit; the result is the union of both channels' patch sets, with resolution changes attached.

### Operation 8 — Botched commit

`pijul unrecord` removes a change locally, leaving its effect in the working tree:

```
$ pijul unrecord
... interactively prompts for which change to unrecord ...
$ # fix the working tree
$ pijul record -m "register: add --csv flag"
```

The botched change, if not yet pushed, is removed without a trace in the local repository (its changes return to the working tree). After push, removing a change requires the more aggressive `pijul rollback`, which produces a new change reversing the bad one — a forward correction with a structurally explicit relationship.

### Operation 9 — Tag

```
$ pijul tag create v0.1.0 -m "First public release"
```

Tags in Pijul are special changes that depend on the current state and have no further effect; they pin a particular moment.

### Operation 10 — Release

```
$ pijul archive --tag v0.1.0 -o ledger-0.1.0.tar.gz
```

`pijul archive` produces a tarball at the named state.

## Model and mental load

What you have to hold:

- Changes as identity-bearing objects with computed dependencies.
- The line-graph representation of file content.
- Channels as named subsets of the patch store.
- Conflicts as first-class objects, resolved by recorded resolutions.
- The push/pull model with channel selection.

The mental load is moderate. The model is internally consistent and rewards understanding the theory, but does not require it for daily use. Users coming from Git can pick up the basic workflow in a day; the advantage of the patch-theory model becomes visible later, when they encounter rename-plus-edit interactions or want to cherry-pick across branches.

## Evolution and history rewriting

`pijul unrecord` for local removal; `pijul rollback` for forward correction; `pijul amend` for editing the most recent change's content or metadata. The system tries to give the user the tools they need without making mutability the default.

## Ecosystem reality

Pijul is small and active. The Nest hosts projects; several are working repositories used by their authors. The Pijul project itself is hosted on the Nest. Documentation is good in the Pijul Manual and increasingly good in tutorials posted by users. The Rust ecosystem provides Pijul with a small but appreciable contributor base.

Tooling integration is sparse compared to Git. Pijul has its own forge, a CLI, no major IDE plugins, no CI/CD ecosystem assuming Pijul. Migration to and from Git exists.

## When to reach for it; when not to

Reach for Pijul when:

- You are starting a new project and find the patch-theory model attractive on its own merits.
- The team is small, the contributors are sympathetic, the hosting requirements are modest.
- You want to invest in a system whose underlying ideas may have a long future, even if the present-day ecosystem is small.

Do not reach for Pijul when:

- You need the GitHub/GitLab feature set: external contributors, CI/CD integrations, project-management hooks.
- The project is large or has many casual contributors who will resist learning a new system.
- You need stability and breadth of tooling more than you need theoretical elegance.

## Epitaph

Pijul is the version control system that figured out what Darcs's theory wanted to be — and is the active proof that the patch-theoretic line of thinking is not a historical curiosity but a position the field could still occupy if it chose.
