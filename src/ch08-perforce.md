# 8. Perforce

## Origin

Perforce was created by Christopher Seiwald in 1995. He had spent years working on Sun's NSE and Jaguar source-control efforts and at Imagen, and the experience left him with a clear thesis about what a version control system for serious commercial software needed: a fast central server, a stateful protocol that the server could reason about, atomic transactions, and a model that handled enormous trees with mostly-binary content. He founded Perforce Software (now branded Helix and called Helix Core for the version control product proper) and shipped what he had been planning. The product remains, three decades later, the dominant version control system in the parts of the software industry where Git did not win.

Those parts are not small. AAA game development is essentially universal Perforce. Film visual effects and animation — Industrial Light and Magic, Pixar, Weta, Framestore — run on Perforce. Hardware design teams use Perforce because the EDA tooling expects it. Embedded and automotive groups use Perforce because the audit semantics and centralized control are what their certification regimes require. Inside major software companies, large teams that work on shipping consumer products with art assets, localized content, and tight release management often choose Perforce despite the rest of the company being on Git.

The system has aged in the way good commercial software ages: continuous improvement without disruptive rewrites. Streams (a first-class branching model) arrived in 2011. Shelving (server-side stashes) arrived in 2009. P4DTG, P4Web, Swarm (the Perforce review tool), Helix4Git (Git-as-a-front-end), and various integrations were added over the years. The license is proprietary; small teams and open-source projects can use it free; large commercial users pay per seat.

## The system on its own terms

Perforce is *aggressively* centralized. There is one server (`p4d`); it holds the depot — the canonical store of all versioned files and history. Clients hold *workspaces* (also called *clients*; the terminology is overloaded), each with a unique name and a *view* mapping depot paths to local paths.

A workspace view looks like:

```
//depot/ledger/main/...    //aditi-laptop/ledger/...
//depot/art/textures/...   //aditi-laptop/textures/...
```

The double-slash prefix means "from the root of the depot." The trailing `...` is Perforce's wildcard for "this directory and all its children." A view can include and exclude paths (`-//depot/foo/bar/...` excludes), can map paths to different local locations, and is the primary mechanism for sparse checkouts. Workspaces are *server-side state*: when Aditi runs `p4 sync`, the server knows exactly which files her workspace claims to have at which revisions, and the protocol updates only what has changed.

Files have *file types*, set at add time and changeable later. The types are not just binary vs. text:

- `text` — text content, potentially keyword-expanded.
- `binary` — opaque bytes, no expansion or EOL conversion.
- `symlink`, `apple` (macOS resource fork), `unicode`, `utf8`, `utf16` — special cases.
- `+l` modifier — exclusive lock; only one user can have this file open for edit at a time.
- `+k` modifier — keyword expansion (`$Id$`, `$Author$`, `$DateTime$`, etc.).
- `+x` modifier — executable bit.
- `+S` modifier — store only the latest *N* revisions; older revisions are pruned.
- `+w` modifier — file is always writable on disk.
- `+m` modifier — modtime is preserved.

The combination matters. `binary+l` is the standard type for art assets in a game project: opaque storage, exclusive locking, no EOL or keyword games. `+S2` on build artifacts says "keep only the last two." File types are configured globally via the *typemap* table or per-file as needed.

The unit of work is the *changelist*. A changelist is a numbered, server-side object that groups files-being-edited with a description and an owner. There is always a *default* changelist for each workspace; `p4 edit foo.c` adds `foo.c` to the default changelist. `p4 change` creates a new numbered changelist (server-assigned) with a description; files can be moved between changelists. `p4 submit` ships a changelist atomically — every file commits or none do, and the changelist number becomes part of permanent history.

Submitted changelist numbers are monotonically increasing and globally unique within the depot. They are the closest analogue to Subversion's revision numbers, with the difference that *unsubmitted* changelists also have numbers (visible in `p4 changes -s pending`). The numbering is not contiguous in the submitted timeline; pending changelists hold numbers that may never be submitted.

The file model is unusually concrete. By default, every file in a Perforce workspace is *read-only on disk*. To edit a file, you run `p4 edit foo.c`; the server records that you've opened the file for edit and the local file becomes writable. When you submit, the server compares your local file to the depot and stores the change. If you didn't `p4 edit` first, you cannot save changes — the filesystem itself enforces the workflow.

This is alien if you've only used optimistic-concurrency systems. It is also the right model for art assets, where the alternative — two artists both modifying the same texture overnight — is genuinely worse than the alternative — one artist waiting for the other to finish.

## Scenario walkthrough

The exemplar plays cleanly in Perforce, with several operations being more concrete than they were in any earlier system.

### Operation 1 — Initial import

Aditi sets up a Perforce server (`p4d`) and creates a workspace mapping. She is working from an empty workspace and wants to import her existing project tree.

```
$ p4 client    # opens an editor on the workspace spec
... (set Root: to /Users/aditi/work/ledger and View: to map //depot/ledger/main/... )

$ cd /Users/aditi/work/ledger
$ cp -r /path/to/initial/ledger/* .
$ p4 reconcile -ad
... (lists every file as "needs add")
$ p4 submit -d "Initial import. CLI skeleton, parser stub, two failing tests."
Submitting change 1.
... files added and submitted ...
Change 1 submitted.
```

`p4 reconcile -ad` is a sweep that detects files in the workspace that are not in the depot and opens them for add. The submit creates changelist 1, the first "real" history entry. From here on, every commit advances the changelist counter.

### Operation 2 — Linear development

The three subsequent commits:

```
$ p4 edit src/ledger/parser.py
//depot/ledger/main/src/ledger/parser.py#1 - opened for edit
$ vi src/ledger/parser.py
$ p4 submit -d "parser: handle blank lines and comments"
Change 2 submitted.
```

Each commit opens the relevant files for edit, makes changes, and submits. The `default` changelist accumulates files; `p4 opened` shows what is currently open.

### Operation 3 — Branch

Perforce has two branching idioms. The older one, *branch specs*, is a named mapping of source paths to target paths used with `p4 integrate`. The newer one, introduced in 2011, is *streams*: first-class branch objects with declared parent relationships, types (mainline, development, release, virtual, task, sparsedev), and merge/copy rules.

Modern projects use streams. Aditi sets up streams from the start:

```
$ p4 stream -t mainline //depot/ledger/main
$ p4 stream -t development -P //depot/ledger/main //depot/ledger/currency-conversion
```

The first creates a mainline stream at `//depot/ledger/main`; the second creates a development stream parented to it. Jonas's workspace is reconfigured to point at the new stream:

```
$ p4 client     # change Stream: to //depot/ledger/currency-conversion
$ p4 sync
... files synced from the development stream ...
```

His subsequent edits and submits land on the development stream. `p4 changes //depot/ledger/currency-conversion/...` shows them; `p4 changes //depot/ledger/main/...` does not.

### Operation 4 — File rename

Perforce has first-class rename via `p4 move`:

```
$ p4 edit src/ledger/reports.py
$ p4 move src/ledger/reports.py src/ledger/report_balance.py
//depot/ledger/main/src/ledger/reports.py#5 - moved from
//depot/ledger/main/src/ledger/report_balance.py - moved to
$ # edit report_balance.py to remove register()
$ # create report_register.py with register() function
$ p4 add src/ledger/report_register.py
$ p4 submit -d "reports: split into balance and register modules"
Change 9 submitted.
```

`p4 filelog -i src/ledger/report_balance.py` follows the file's history through the rename; `p4 annotate` similarly handles the rename. The history is structurally preserved, not heuristically reconstructed.

### Operation 5 — Binary file added

For the logo PNG, the typemap configuration matters:

```
$ p4 typemap     # opens an editor; ensure entries like:
... .png    binary+l ...
... .psd    binary+l ...
```

With the typemap set, Mireille adds the logo and submits:

```
$ p4 add docs/logo.png
$ p4 submit -d "docs: add project logo"
Change 11 submitted.
```

The file is stored as `binary+l` — opaque bytes, exclusive lock. To edit it later, an artist must `p4 edit docs/logo.png`, which acquires the exclusive lock; until they submit or revert, no other user can `p4 edit` the file. This is the model that game and film studios stay on Perforce *for*.

Storage cost: Perforce stores binaries efficiently when they don't change much (full copies) and even more efficiently when they do (lazy diff via xdelta when applicable). Repositories with terabytes of art assets are routine; a Perforce server can be tuned to handle them without pathological behavior.

### Operation 6 — Parallel edits to the same file

Aditi and Jonas both want to edit `README.md`, which is `text` (no `+l`). They both run `p4 edit`, work, and the second to submit must integrate first.

```
# Aditi
$ p4 edit README.md
$ vi README.md
$ p4 submit -d "README: add Quick Start"
Change 12 submitted.

# Jonas, meanwhile
$ p4 edit README.md
$ vi README.md
$ p4 submit -d "README: rewrite Overview"
Out of date files must be resolved or reverted.
$ p4 sync
$ p4 resolve
... auto-merges where possible, prompts where not ...
$ p4 submit -d "README: rewrite Overview"
Change 13 submitted.
```

`p4 resolve` is the merge step. For text files it offers `am` (accept merge), `at` (accept theirs), `ay` (accept yours), `as` (accept safe — only auto-mergable changes), and `e` (edit). Conflicts that cannot auto-merge open in a three-way merge tool (P4Merge by default).

### Operation 7 — Merge with conflict

Jonas's stream-to-stream merge of `currency-conversion` back into `main` uses streams' integration model:

```
$ p4 client    # switch workspace back to //depot/ledger/main stream
$ p4 sync
$ p4 merge --from //depot/ledger/currency-conversion
... lists files to merge ...
$ p4 resolve
... auto-merges where possible, prompts on parser.py and report_balance.py ...
$ vi src/ledger/parser.py    # if needed
$ p4 submit -d "Merge currency-conversion into main; resolve parser conflict"
Change 15 submitted.
```

The merge produces a single changelist on `main` that records the integration. `p4 integrated -t //depot/ledger/main/... //depot/ledger/currency-conversion/...` shows the integration history. Streams track this automatically; the system knows which changes have been merged where.

### Operation 8 — Botched commit needing rewrite

Mireille catches her debug print and typoed message immediately. The submitted changelist is, in general, immutable. Her options:

1. **Edit the description.** With sufficient privilege:

```
$ p4 change -f -d "register: add --csv flag" 17
```

The `-f` flag forces the description change; the protect table determines whether Mireille has rights to do this. The change is logged but not versioned.

2. **Obliterate the changelist.** `p4 obliterate` (admin-only) can permanently remove a changelist's content, *if* nothing has been built on top of it. This is rare and audited.

3. **Submit a follow-up** that reverts the debug line. The standard, low-friction approach.

She takes (1) for the description (the server admin has granted her self-admin on her own changes, a common configuration) and (3) for the content:

```
$ p4 change -f -d "register: add --csv flag" 17
$ p4 edit src/ledger/report_register.py
$ vi src/ledger/report_register.py    # remove debug line
$ p4 submit -d "register: remove stray debug print from change 17"
```

The bad changelist's content remains in history; the description is now correct. A reviewer reading the log sees both changes and understands what happened.

### Operation 9 — Tag

Perforce uses *labels* for tagging. A label is a named collection of file revisions — it can be the entire depot at a moment, or any subset:

```
$ p4 label -t static v0.1.0
... opens an editor; saves the label spec ...
$ p4 labelsync -l v0.1.0 //depot/ledger/main/...
... captures current head revisions of every file under main ...
```

The `-t static` option fixes the label's revisions at labelsync time; the alternative, `automatic`, dynamically resolves to whatever revision matches a query. To check out the labeled state:

```
$ p4 sync //depot/ledger/main/...@v0.1.0
```

### Operation 10 — Release

The release tarball is generated from the labeled state. `p4 print` extracts file content; a script that walks the label and writes files to a clean directory produces the release tree:

```
$ p4 sync //depot/ledger/main/...@v0.1.0
$ rsync -a --exclude '.p4*' /Users/aditi/work/ledger/ /tmp/ledger-0.1.0/
$ tar czf ledger-0.1.0.tar.gz -C /tmp ledger-0.1.0
```

Mireille writes `CHANGELOG.md` as part of the release submit; the label is applied; the tarball is produced and published.

## Model and mental load

What you have to hold:

- The workspace concept. Each user has at least one workspace, server-side; switching machines means either a new workspace or syncing an existing one.
- Views. The mapping from depot to local is workspace state. Misconfigured views are the most common new-user error.
- File types. Choosing the right type at add time is important; getting it wrong (binary as text, especially) corrupts data.
- The read-only-by-default model. `p4 edit` before editing is a habit you develop in the first day.
- Changelists as a first-class object. Pending changelists hold work-in-progress; submitted changelists are permanent.
- Streams (or branch specs, in older deployments). The branching model has its own grammar.
- Resolve. Every parallel edit ends in a resolve step before submit; the workflow is built around this.

The mental load is not low, but it is *concrete*. Every concept has a clear object behind it on the server. The system is more bureaucratic than Git but also less surprising; recovery is easier because the server has authoritative state.

## Evolution and history rewriting

Submitted history is, by default, immutable. The administrator can `p4 obliterate` or rewrite descriptions. Developers cannot rewrite submitted history.

Pending changelists are fully editable: shelved (server-stashed), unshelved on a different workspace, reordered, recombined. Shelving since 2009 has filled the role Git's stash and rebase fill in modern Git workflows: a place to park work and recompose it before it becomes permanent.

The cultural stance is the audit-trail one. You shape work in pending changelists, then commit a clean history that is preserved as-is.

## Ecosystem reality

Perforce-the-company sells the server and offers a hosted service. The client is free to download. Tooling around it is mature and commercial: P4V (the GUI), Helix Swarm (review), P4Code (Visual Studio plugin), P4 Eclipse, integrations with Maya, Houdini, Unreal Engine, Unity, basically every DCC tool used by artists. The Unreal Engine shipped with Perforce integration as the default for years; it now also integrates with Plastic SCM, but the Perforce workflow is what most studios still use.

In game and film, Perforce is not contested. In other industries — embedded, hardware, regulated software — Perforce competes with vendor-specific tools and continues to win where the requirements are heavy. The community knowledge is concentrated in those industries. Stack Overflow's Perforce questions are unfashionable; the GDC talks and SIGGRAPH papers that touch version control assume Perforce.

Helix4Git is Perforce's Git-as-a-front-end product; it lets developers use Git locally against a Perforce server. Adoption is mixed. Most studios that want Git let their engine programmers use Git for engine code and keep artists on Perforce for assets, with no attempt to bridge.

## When to reach for it; when not to

Reach for Perforce when:

- You have substantial binary content (art, audio, video) that must be locked while edited.
- Your trees are very large (hundreds of gigabytes to terabytes) and growing.
- You need centralized authority and audit semantics that match a regulatory regime.
- Your tooling expects Perforce (game engines, EDA tools, certain DCC pipelines).
- You have a team that includes non-developers who benefit from the workspace/file model.

Do not reach for Perforce when:

- The team is small, the codebase is text-only, and the cost of running a server is the dominant concern. Git over a free GitHub or self-hosted forge is simpler.
- The team's culture is GitHub-pull-request-driven and changing it is expensive.
- You do not need the locking model, the binary handling, or the centralized authority. Most of what makes Perforce expensive is overhead in that case.

## Epitaph

Perforce is the version control system that won the parts of the industry where Git's blind spots — locking, binaries, scale, central authority — would have been malpractice, and it is the closest thing the field has to an honest centralized DVCS counterpoint.
