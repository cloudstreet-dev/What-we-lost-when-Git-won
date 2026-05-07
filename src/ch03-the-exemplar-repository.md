# 3. The Exemplar Repository

This chapter specifies a single fictional project and a sequence of operations that every later format chapter renders. The point is to fix the scenario once, with enough precision that no later chapter has to invent details. When Chapter 8 says "Mireille adds the logo and Aditi reverts a botched commit," every reader should know, without flipping back, exactly what is being added, what was botched, and which commit is being reverted.

Read this chapter once. The format chapters will assume you have.

## The project

The project is a small command-line tool called **`ledger`** — a plain-text double-entry bookkeeping utility for personal finance, vaguely in the spirit of John Wiegley's `ledger` and Martin Blais's `beancount`, though the specifics differ. It is implemented in a single language (Python 3.11). It reads plain-text journal files, parses transaction entries, and produces account balances, account histories, and a few simple reports.

The project is small. Throughout the scenario, the working tree never exceeds twenty source files, never includes more than two binary assets, and never grows past about fifteen hundred lines of code. The point is not to model a realistic codebase; the point is to exercise every axis of every version control system the book covers, in a tractable form.

The repository, at the moment Aditi first creates it, has the following layout:

```
ledger/
├── README.md
├── LICENSE
├── pyproject.toml
├── src/
│   └── ledger/
│       ├── __init__.py
│       ├── cli.py
│       ├── parser.py
│       └── reports.py
├── tests/
│   ├── test_parser.py
│   └── fixtures/
│       └── small.journal
└── docs/
    └── format.md
```

The journal file format is a subset of the Beancount syntax: each transaction is a date, a narration, and two or more account/amount lines, separated by blank lines. A small example, which appears in `tests/fixtures/small.journal`, looks like this:

```
2026-01-04 "Coffee"
  Expenses:Food:Coffee   4.25 USD
  Assets:Cash:Wallet    -4.25 USD

2026-01-05 "Rent for January"
  Expenses:Housing:Rent  1875.00 USD
  Assets:Bank:Checking  -1875.00 USD
```

The tool is invoked as `ledger balance journal.txt` or `ledger register Expenses:Food journal.txt`, and prints the result to stdout. The implementation details do not matter for the version control story; only the file shapes and the timing of changes do.

## The contributors

Three people contribute to `ledger` over the course of the scenario.

**Aditi Rao** (`aditi@example.org`) creates the project. She is in Bangalore, eight hours from the project's notional UTC clock. She is the original author and remains the project's primary maintainer throughout the scenario. Her commits are identifiable by her email address.

**Jonas Berg** (`jonas@example.org`) joins second. He is in Stockholm, on or near UTC. He works asynchronously to Aditi and reviews her changes and contributes his own. He is the contributor responsible for the parallel edit that produces the merge conflict.

**Mireille Okafor** (`mireille@example.org`) joins third. She is in Lagos, also near UTC, and works in shifted hours from Jonas. She contributes the binary asset and is the contributor who makes the botched commit later corrected by amend or rewrite. She also cuts the release.

The three contributors never sit in the same room. They communicate through commit messages, the issue tracker (when one exists in the system being demonstrated), and email. The version control system is the substrate through which their work meets.

## The sequence of operations

The scenario is a sequence of ten ordered operations. Each format chapter shows how the system in question performs each, in that system's native idioms. The wording below is deliberately system-agnostic; a Subversion chapter will say "commit revision 1," a Git chapter will say "commit," a Darcs chapter will say "record," and so on.

### Operation 1 — Initial import

Aditi creates the repository on her laptop, populated with the layout above. The initial state contains exactly the files listed; every Python file has a meaningful skeleton (`cli.py` has an argument parser stub, `parser.py` has a function `parse_journal()` that returns a list of transactions, `reports.py` has `balance()` and `register()` stubs, `test_parser.py` has two failing tests). She makes one commit recording all of this. Her message, verbatim, is:

> Initial import. CLI skeleton, parser stub, two failing tests.

If the system in question requires a separate "initialize repository" step before the first commit, that step is folded into Operation 1. If the system requires a remote/server to be configured first, the chapter notes how that works.

### Operation 2 — Linear development

Aditi makes three more commits over the course of the next two days, fleshing out the parser. The commits, in order, are:

1. `parser: handle blank lines and comments`
2. `parser: tokenize amounts with currency`
3. `tests: add fixture for two-currency journal`

These are unremarkable linear commits. They serve to give the repository a small but real history before any branching or sharing happens. The third commit adds a new fixture file, `tests/fixtures/multicurrency.journal`, and updates `test_parser.py` to read it.

### Operation 3 — Branch

Aditi shares the repository with Jonas. The mechanism is the system's native one: a clone, a pull, a checkout against a server, an exchange of patches by email — whichever the system uses. Whatever that mechanism is, after Operation 3 begins Jonas has a working copy with the four commits from Operations 1 and 2.

Jonas creates a branch named `currency-conversion` to develop multi-currency support. On this branch he makes two commits:

1. `parser: store amounts as (value, currency) pairs`
2. `reports: convert amounts to display currency in balance`

The first commit is a non-trivial change: it modifies the data structure used throughout the parser. The second is shorter. The branch exists for several days while Jonas iterates; it does not yet merge back to the main line.

A chapter for a system without first-class branches (CVS, in some renderings; Subversion's branches-as-directories) shows the system's idiom for the same effect.

### Operation 4 — File rename

Meanwhile, on the main line, Aditi decides that `reports.py` should be split. She renames it to `report_balance.py` and adds a new file, `report_register.py`, moving the `register()` function. The file `reports.py` no longer exists after this operation; `report_balance.py` contains the `balance()` function, and `report_register.py` contains `register()`. She also updates `cli.py` to import from the two new modules.

This is a deliberate rename-with-split. It is *the* operation that distinguishes systems that handle rename history well from systems that do not. The chapter for each system shows what `log` or `blame` on `report_balance.py` returns afterward.

She makes this as a single commit:

> reports: split into balance and register modules

### Operation 5 — Binary file added

Mireille joins the project. She designs a small logo for the project and adds it. The file is `docs/logo.png`, a 24 KB PNG image. She also updates the README to reference the logo with a Markdown image link.

This is one commit:

> docs: add project logo

For systems with explicit binary handling (Perforce filetypes, Plastic SCM's smart-locking on binary patterns, Git LFS, git-annex, ClearCase's element types), the chapter shows the configuration step required. For systems without binary handling (the default Git case, Mercurial without largefiles, Darcs), the chapter notes the consequence — the binary lives in history at full size — without drama.

### Operation 6 — Parallel edits to the same file

Aditi and Jonas, working independently in their own time zones, both edit `README.md`. Aditi's edit adds a section called *Quick Start* with three sample commands. Jonas's edit, made on his `currency-conversion` branch and then synced back, rewrites the *Overview* paragraph to mention multi-currency support. The two edits do not touch the same lines, but they are close enough — separated by only a blank line — that a naïve diff and patch will fail to merge cleanly in some systems and merge cleanly in others.

For the chapter's purposes, the *intended* outcome is that both edits land. Whether they merge automatically or require human resolution is a property of the system being demonstrated.

### Operation 7 — Merge with conflict

Jonas merges his `currency-conversion` branch back to the main line. The merge involves the parallel `README.md` edits from Operation 6 *and* a real conflict in `parser.py`: Aditi's main-line work in Operation 4 (the split into two report modules) updated `parser.py` to expose a function `tokenize_amount(s)` returning a single number; Jonas's branch work in Operation 3 changed the same function to return a `(value, currency)` tuple. The two changes overlap on the function's return statement.

The conflict is resolved in favor of Jonas's tuple-returning version, with Aditi's caller code in `report_balance.py` updated to unpack the tuple. The resolution is committed as a merge commit (or the system's equivalent — patch-system chapters describe the resolution differently).

The merge commit's message is:

> Merge currency-conversion into main; resolve parser/reports conflict

### Operation 8 — Botched commit needing rewrite or amend

Mireille adds a new feature: a `--csv` flag to the `register` report. While testing, she commits a version that includes a stray `print("DEBUG:", row)` line and a commit message with a typo: `"register: add --cvs flag"`. She notices both immediately after committing and before sharing the commit with anyone else.

The chapter for each system shows how Mireille fixes this. Some systems (Git, Mercurial with `--amend`, Sapling, Jujutsu, Darcs's `unrecord`) make this trivial. Some systems (CVS, Subversion, classic Perforce until recent shelving features) make it impossible to amend, and the correction is a follow-up commit reverting the debug line and fixing the message in the next commit's body.

The chapter notes the difference, and notes whether the botched commit is *erased* from history or *added to* by the correction. This distinction is the chapter's first opportunity to discuss the *history mutability* axis from Chapter 2 in concrete terms.

### Operation 9 — Tag

Aditi tags version `0.1.0`. The tag is annotated where the system supports it: tag name `v0.1.0`, message `First public release`, author Aditi.

The format chapter shows the system's tagging idiom — `git tag -a`, `hg tag`, `svn cp` to a tags directory, `bk tag`, `fossil tag add`, and so on — and notes whether tags are first-class objects or merely conventional names attached to revisions.

### Operation 10 — Release

The release is the act of producing a versioned artifact and publishing it somewhere. For `ledger`, the artifact is a source tarball: `ledger-0.1.0.tar.gz`, generated from the tagged commit, accompanied by release notes in a `CHANGELOG.md` that Mireille updates as part of the same set of operations.

The release is *not* a commit per se; it is a workflow on top of the version control system. The format chapter describes the conventional release workflow for each system: how the tarball is generated (`git archive`, `hg archive`, `svn export`, `bk export`, etc.), where it gets published (the project's hosted release page, an FTP server, an email announcement to a list, the system's own bundle distribution mechanism in the case of Fossil), and what the relationship is between the release and the tag.

For systems with first-class release support (Fossil's tarballs, GitHub Releases as a layer over Git tags), the chapter notes that. For systems without it, the chapter describes the manual procedure and stops there.

## What the scenario exercises

The ten operations exercise every axis named in Chapter 2.

Operation 1 exercises *initial setup* and the system's *working-copy model*. Operation 2 exercises basic linear *commit* and *atomicity*. Operation 3 exercises *distribution*, *sharing*, and *branching cost*. Operation 4 exercises *rename handling* and *unit-of-history* (snapshot vs. delta vs. patch). Operation 5 exercises *binary handling*, *scaling*, and any explicit *file type* mechanism. Operation 6 exercises *concurrent edits* and the system's automatic-merge logic. Operation 7 exercises *conflict semantics* and *merge*. Operation 8 exercises *history mutability* and the system's *amend/rewrite* mechanism. Operation 9 exercises *identity* and *tagging*. Operation 10 exercises the *release workflow* and the boundary between version control and the rest of the toolchain.

Some operations are stranger in some systems than in others. A SCCS chapter cannot really show Operation 3 — there is no native distribution model — and the chapter discusses what teams actually did (shared NFS, file copies, a designated sccs server). A Darcs chapter renders Operation 7 as a patch-conflict resolution rather than a merge commit, because Darcs does not have merge commits per se. A Fossil chapter folds Operation 9 and Operation 10 into the same workflow because Fossil's bundle format is the release.

Where a system cannot perform an operation as described, the chapter says so and shows the system's nearest equivalent. Chapter 32 returns to these gaps as inputs to the decision framework.

## What the scenario does not include

The scenario does not include a security incident, a forced reset, a corrupted repository, a long-running fork, an external contributor without commit rights, a backport from a stable branch, or a multi-team monorepo. These are real situations and several systems handle them very differently, but they would multiply the scenario beyond tractability. Where a system is particularly distinctive in handling one of these (Sapling and stacked diffs; Gerrit and external contributions; Plastic and gluon for non-developers), the chapter notes it as an aside.

The scenario also assumes the project is small. Scaling behavior — what happens at a million files, ten million commits, a hundred-gigabyte working tree — is discussed where relevant in the chapters covering systems whose answers to those questions are distinctive. The exemplar is not the place to test scaling.

With the exemplar fixed, Part II begins, with the system that started everything: SCCS, in 1972, the year before C had a name.
