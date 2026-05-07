# 9. ClearCase

## Origin

ClearCase came out of Atria Software in 1992, designed by a team that had previously built DSEE — the Domain Software Engineering Environment — at Apollo Computer in the late 1980s. DSEE was the first commercial system to integrate version control, build management, and configuration management as a coherent whole, and ClearCase carried its lineage forward onto Unix workstations and, eventually, Windows. Atria merged with Pure Software in 1996 to form Pure Atria; Rational Software bought Pure Atria in 1997; IBM bought Rational in 2003; HCL Technologies bought IBM's Rational Software product portfolio in 2018. ClearCase is currently sold and maintained by HCL.

The system's animating idea was that configuration management is bigger than version control. A configuration is not "this commit"; it is *the set of file versions that, together, constitute a buildable system*. ClearCase's central abstraction is the *config spec*, a rules file that selects which version of which file appears in your *view*. The version control system is the substrate; the config spec is the language for expressing what the user wants to see and build.

This was a serious idea, and ClearCase was a serious product. Through the 1990s and into the 2000s it was the high-end choice for large-scale enterprise development: telecom, defense, banks, big-iron consulting projects. It was also expensive, complicated, and slow, and it required dedicated administrators. Its decline began when distributed VCS made its central-server-and-MVFS model look less essential, and its enterprise customer base aged out of greenfield projects. By 2020, ClearCase was a maintenance product. In 2026, it is one — HCL ships releases, customers run them, and few new projects adopt it.

## The system on its own terms

ClearCase has three concepts that distinguish it: the *VOB*, the *view*, and the *config spec*.

A **VOB** (Versioned Object Base) is a storage unit. It holds the versioned data — files, directories, branches, types, triggers, metadata — for a logical grouping of work. A project may span multiple VOBs (for code, for documentation, for assets), or fit into one. VOBs are heavyweight: creating one is an administrative act, not a developer one.

A **view** is what you work in. There are two types. *Dynamic views* present a virtual filesystem (the MVFS, Multi-Version FileSystem) where files appear on demand, selected by the config spec, with no local copy until the file is actually read. *Snapshot views* are conventional: the config spec is evaluated, the resulting files are copied to local disk, and you work against a real working copy that needs `cleartool update` to refresh.

The MVFS is the most genuinely strange thing in this book. From the user's perspective, the file system *is* version controlled. `ls /vobs/ledger/src/ledger/` shows whatever files the config spec selects; reading a file produces those bytes; writing requires checkout. The system intercepts file operations at the kernel level. On Unix, MVFS was a kernel module; on Windows, a filesystem driver. Builds run against the dynamic view directly, and `clearmake` records *which exact version of every file was read* during the build, producing a *configuration record* that lets the build be reproduced exactly later. This is configuration management that knows what it's doing.

A **config spec** is a rules file. A simple one looks like:

```
element * CHECKEDOUT
element * .../currency-conversion/LATEST
element * /main/LATEST
```

Read top to bottom: for each file element, prefer the version checked out by the current user; otherwise, the latest version on the `currency-conversion` branch if it exists; otherwise, the latest version on the `main` branch. Config specs can be arbitrarily complex: pinning specific revisions, selecting by label, mixing branches, applying time-based selection. A view's config spec defines what the user sees; changing the spec changes the view.

Branches are first-class typed objects. You declare a branch *type* on a VOB (`mkbrtype currency-conversion`) before you can branch onto it. Each file element on which the branch is applied gets its own branch off whatever the branch parent says (typically `LATEST` of `main` at the time the branch is taken). This per-element branch model, combined with config specs, gives the system extraordinary expressive power and a steep learning curve.

UCM (Unified Change Management) is the optional process layer that ClearCase added in the 2000s to constrain the flexibility into a sane workflow. UCM defines *projects* (containers for related streams), *streams* (development lines, similar to Perforce streams), *activities* (changesets — groups of related changes that move together), and *baselines* (named snapshots that streams advance against). Most large ClearCase deployments use UCM; the underlying base ClearCase model is still there, but UCM is the layer the developers see most.

The command surface is `cleartool`, with subcommands. The grammar is dense: `cleartool mkelem`, `cleartool checkout`, `cleartool checkin`, `cleartool mkbranch`, `cleartool merge`, `cleartool lsvtree` (list version tree), `cleartool describe`, `cleartool diff`, `cleartool find`, `cleartool setcs` (set config spec), `cleartool catcs` (cat config spec). A command-line cheat sheet was a near-universal artifact in any team using ClearCase.

## Scenario walkthrough

### Operation 1 — Initial import

The administrator first creates the VOB:

```
$ cleartool mkvob -tag /vobs/ledger -comment "Ledger project VOB" /var/vobs/ledger.vbs
```

Aditi (or the admin) creates a development view and mounts it:

```
$ cleartool mkview -tag aditi_ledger /views/aditi_ledger.vws
$ cleartool mount /vobs/ledger
$ cleartool setview aditi_ledger
```

In a dynamic view, `/vobs/ledger` is now visible. Aditi populates it:

```
$ cd /vobs/ledger
$ cleartool mkelem -nc -mkpath -ci src/ledger/parser.py
... and so on for every file ...
```

`mkelem` creates an *element* (the ClearCase term for a versioned file or directory); `-nc` means "no comment"; `-mkpath` creates intermediate directories as needed; `-ci` checks the new element in immediately. The first version of each element becomes `/main/0`; subsequent checkins on the main branch produce `/main/1`, `/main/2`, etc.

For a serious project, the import uses `clearfsimport`, which sweeps a source directory and creates VOB elements:

```
$ clearfsimport -recurse -nset /path/to/initial/ledger /vobs/ledger
```

The result is a populated VOB with each file element starting at `/main/0`.

There is no project-level commit. Each `mkelem -ci` and each subsequent checkin is per-element. An *activity* in UCM groups them; in base ClearCase, the team's convention does. The exemplar uses base ClearCase for clarity; the UCM equivalent is briefly noted at the end.

### Operation 2 — Linear development

The three subsequent commits are:

```
$ cleartool checkout src/ledger/parser.py
$ vi src/ledger/parser.py
$ cleartool checkin -c "parser: handle blank lines and comments" src/ledger/parser.py
```

Each commit is per-element. The version tree `/main/1`, `/main/2`, `/main/3` accumulates on `parser.py`. UCM would group the three changes into an activity called something like `parser-improvements`; base ClearCase has no such grouping by default.

### Operation 3 — Branch

Jonas joins. The administrator creates a view for him; he sets the view and works in it. For the `currency-conversion` work, the team first creates a *branch type*:

```
$ cleartool mkbrtype -nc currency-conversion
```

Then Jonas creates a config spec that selects the currency-conversion branch:

```
$ cat > /tmp/cc.cs <<EOF
element * CHECKEDOUT
element * .../currency-conversion/LATEST
element * /main/LATEST -mkbranch currency-conversion
EOF
$ cleartool setcs /tmp/cc.cs
```

The third rule is the magic: when a file is checked out for editing, if it does not yet have a `currency-conversion` branch, *create one*. This means Jonas can edit any file and his changes land on the `currency-conversion` branch automatically. To make a branch commit:

```
$ cleartool checkout src/ledger/parser.py
... new branch /main/currency-conversion/0 created ...
$ vi src/ledger/parser.py
$ cleartool checkin -c "parser: store amounts as (value,currency) pairs" src/ledger/parser.py
```

The version tree on `parser.py` now branches. `cleartool lsvtree -all -graphical src/ledger/parser.py` opens an X11 window showing the version graph — one of ClearCase's distinctive features.

### Operation 4 — File rename

ClearCase has rename via `cleartool mv`:

```
$ cleartool checkout -nc src/ledger
$ cleartool mv src/ledger/reports.py src/ledger/report_balance.py
$ # edit report_balance.py to remove register()
$ cleartool checkout -nc src/ledger/report_balance.py
$ vi src/ledger/report_balance.py
$ cleartool mkelem -nc -ci src/ledger/report_register.py
$ cleartool checkin -c "split into balance and register" \
    src/ledger src/ledger/report_balance.py
```

The directory containing the renamed files is itself a versioned element; modifying it (adding, removing, renaming children) requires checking out the directory. This is one of ClearCase's distinctive structural properties: directory structure is itself version-controlled.

The history of `report_balance.py` includes the rename. `cleartool lsvtree` follows the rename across the version graph; `cleartool annotate` follows it for blame.

### Operation 5 — Binary file added

ClearCase has *element types*. Common types include `text_file`, `binary_delta_file`, `compressed_file`, `file`. The element type determines storage and merge behavior. To add a binary properly:

```
$ cleartool mkelem -nc -mkpath -eltype binary_delta_file docs/logo.png
$ cp /path/to/logo.png docs/logo.png
$ cleartool checkin -c "docs: add project logo" docs/logo.png
```

`binary_delta_file` uses a binary diff algorithm for storage and disables text-style merge. ClearCase does not have an exclusive lock on binary types built in; locks are a separate concept (`cleartool lock`), used to lock entire branches or VOBs. For per-file exclusive editing, the team uses *reserved checkouts*:

```
$ cleartool checkout -reserved docs/logo.png
```

A reserved checkout is exclusive: nobody else can check the same element out reserved. This is the ClearCase analogue to Perforce's `+l` filetype, with the difference that it is per-checkout rather than per-element-type.

### Operation 6 — Parallel edits

For parallel edits to `README.md`, Aditi and Jonas both `cleartool checkout -unreserved` (the default), edit, and check in. The first to check in succeeds; the second's checkin succeeds *with a separate version on the same branch*, after which a merge produces a single converged version.

```
$ cleartool checkin -c "README: rewrite Overview" README.md
... succeeds, producing /main/12 ...
$ cleartool merge -to README.md /main/11    # Aditi's competing version
... auto-merges where possible ...
$ cleartool checkin -c "merge competing README changes" README.md
```

The parallel-edit handling is functional but heavier than CVS or Subversion's because of the explicit merge step.

### Operation 7 — Merge with conflict

To merge the `currency-conversion` branch back to `main`, Jonas (or Aditi) uses `cleartool merge` over the affected elements:

```
$ cleartool setview aditi_ledger    # view selecting /main/LATEST
$ cleartool checkout -nc src/ledger/parser.py
$ cleartool merge -to src/ledger/parser.py \
    /vobs/ledger/src/ledger/parser.py@@/main/currency-conversion/LATEST
... reports auto-merges and conflicts ...
$ vi src/ledger/parser.py    # resolve markers
$ cleartool checkin -c "Merge currency-conversion: parser changes" src/ledger/parser.py
```

The same operation on `report_balance.py`. ClearCase records the merge as a *hyperlink* between the source and target versions — first-class structural metadata, not just a comment. `cleartool lshistory -fmt` queries can find merges; `cleartool findmerge` finds elements that need merging.

UCM simplifies this: `cleartool deliver` ships the activities from a child stream to the parent, with the merge happening as part of the delivery.

### Operation 8 — Botched commit needing rewrite

ClearCase's stance is conservative. There is `cleartool rmver` (remove a version), but it is restricted: you cannot remove a version that has descendants, that is referenced by a label or branch, or that has metadata hyperlinks. Mireille's botched checkin, freshly made and not yet referenced by anyone, can be removed:

```
$ cleartool rmver -force src/ledger/report_register.py@@/main/4
... requests confirmation, then removes the version ...
```

After removal, she can re-checkout and re-checkin with the corrected content and message.

In practice, most teams disable `rmver` for non-administrators. The honest workflow is the same as Subversion's: commit a follow-up that fixes the problem, leave the original in history.

### Operation 9 — Tag

ClearCase tags are *labels*, applied to versions:

```
$ cleartool mklbtype -nc v0.1.0
$ cleartool mklabel -recurse v0.1.0 /vobs/ledger
```

The first command declares the label type; the second applies it to every element under the VOB at its current selected version. Like Perforce labels, ClearCase labels are first-class: they have history, can be modified with `mklabel -replace`, can be removed with `rmlabel`. To check out the labeled state, a config spec like:

```
element * v0.1.0
element -directory * /main/LATEST
```

The directory rule is necessary because directory versions need to be at LATEST (or labeled too) for the file selections to work; the practical detail is one of the rough edges that defined the ClearCase administrator's job.

### Operation 10 — Release

The release is conventionally produced by setting a snapshot view to the labeled config spec, exporting the resulting tree, and tarballing it:

```
$ cleartool mkview -snapshot -tag release_v0.1.0 ...
$ cleartool setcs -tag release_v0.1.0 /path/to/release.cs
$ cd /views/release_v0.1.0/vobs/ledger
$ tar czf /tmp/ledger-0.1.0.tar.gz --exclude='.@@' .
```

The exclude pattern keeps ClearCase metadata files out of the tarball. UCM has `mkbl` (make baseline) as a richer concept than labels, with promotion levels, full metadata, and stream relationships; the release workflow in UCM is to declare a baseline at the desired promotion level.

## Model and mental load

What you have to hold:

- VOBs vs views. A VOB is the data; a view is the user's perspective; the two are independent.
- Dynamic vs snapshot views. The MVFS gives you live access; snapshot views give you predictability.
- Config specs. The whole user-facing model passes through them; understanding them is non-negotiable.
- Per-element branches. The branch is per file, not per project; the *branch type* is per-VOB.
- The element-type taxonomy. Different types have different storage and merge behaviors.
- Reserved vs unreserved checkouts. The locking model is per-checkout, not per-file-type.
- UCM, if your project uses it. UCM adds projects, streams, activities, baselines, and a different command vocabulary on top of the base.

The mental load is genuinely high. ClearCase requires a class to teach. Most teams that adopted it had a dedicated administrator whose job included maintaining triggers, type definitions, and config-spec patterns; the developers got a curated subset. The system was powerful in proportion to that overhead.

## Evolution and history rewriting

Base ClearCase has `rmver` and `rmbranch` for surgical history edits, restricted by reference. UCM adds nothing to history rewriting; it adds process discipline that prevents most cases where rewriting would be wanted.

The system has not evolved structurally since the late 2000s. HCL maintains it: bug fixes, security updates, occasional integrations. The model is fixed.

## Ecosystem reality

ClearCase in 2026 is in run-out mode for most customers. The teams still on it have been on it for fifteen-plus years; they have invested in administrators, in trigger code, in build scripts that depend on `clearmake`, in configuration management practices that match its model. The cost of migrating is enormous; the benefit is not always clear. So they stay.

HCL's product team is small. The releases are slow. Documentation exists in IBM's old form and HCL's new form, and is not always synchronized. There is a community around it on the IBM and HCL forums, mostly long-time users helping each other through the same problems they were having in 2010.

Migration tools exist: scripts that walk VOBs and produce Git or Subversion histories, with various levels of fidelity. The merge hyperlinks and config-spec history are the parts that don't translate well; teams accept the loss as part of the migration cost.

## When to reach for it; when not to

You do not reach for ClearCase in 2026 unless you are already on it and the cost of moving exceeds the cost of staying. The system's strengths — config specs, MVFS, integrated build management, configuration records — were genuine, and several of them are unmatched in the modern landscape. But the operational cost is enormous and the workforce that knows the system is retiring.

If you are evaluating it from a clean slate — you are not.

If you are inheriting it — your job is to understand it well enough to maintain or migrate, not to extend it. Read the IBM/HCL documentation; talk to the administrator who has been there for ten years; prefer migration over operational deepening unless the regulatory environment forbids it.

## Epitaph

ClearCase was the most sophisticated configuration management system the field has ever shipped, and it was sophisticated enough that the field eventually decided sophistication wasn't what it wanted.
