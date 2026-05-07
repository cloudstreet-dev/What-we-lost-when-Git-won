# 12. Monotone

## Origin

Monotone was started by Graydon Hoare in 2003. Hoare — later widely known as the original creator of Rust — designed Monotone with a specific set of priorities that made it different from every other version control system contemporary with it. The most important were *cryptographic identity as the basis for everything* and *content-addressed storage*. Both ideas predated Monotone in the abstract; Monotone was the first version control system to make them load-bearing.

The system shipped under GPL. It was used by a handful of projects — Pidgin (then GAIM) tried it; the OpenEmbedded project used it for years; Hoare himself used it for some of his own work — but it never reached widespread adoption. Its performance was poor at scale, particularly on the SQLite-based storage backend it used, and it arrived in the same window as Git, Mercurial, and Bazaar, which together defined a different model the field embraced more readily. Active development tapered off in the 2010s. The project is technically alive — releases occurred as late as 2022 — but practically dormant.

Monotone matters because it influenced everything that came after. Git's use of SHA-1 content addressing for blobs, trees, and commits is recognizably the Monotone idea. Pijul's (later) emphasis on a precisely defined storage model and operations over it has Monotone's discipline. The notion that revisions can have certificates attached — assertions made by signed parties about the revision's status — was a serious idea that no mainstream system has equaled since.

The chapter is short relative to the system's intellectual influence. Monotone is, in 2026, primarily a thing to read about. The reading is rewarding.

## The system on its own terms

Monotone's storage is content-addressed. A *file* is named by the hash of its bytes. A *manifest* is a list of (path, file-hash) pairs — what Git calls a tree. The hash of the manifest names the entire state of a working tree. A *revision* is a small structure containing the manifest hash, the parent revision hash(es), and the changes from each parent. The hash of the revision names the entire history up to that point; changing any byte of any ancestor changes the revision's hash.

This is the same model Git later used. Monotone got there first, in code, as a working system, two years before Git existed. The model is described in *The Monotone Tutorial* and in academic work by Hoare; reading it after Git is deeply familiar.

What Monotone added beyond content-addressed storage was *cryptographic identity*. Every user has an RSA key pair. Every commit is signed; without a signature, the commit is not authoritative. A revision's existence is established by its hash being computable; a revision's *meaning* — that this revision is a commit by Aditi, that it is on the `main` branch, that it has been reviewed and approved — is established by *certificates* attached to the revision.

A certificate is a signed assertion: "I, holder of this key, assert that revision X has property Y with value Z." Standard certs include `branch:main`, `author:aditi@example.org`, `date:2026-04-01T08:00:00`, and `changelog:reports: split into balance and register modules`. Custom certs can be defined: `tested:passing`, `reviewed-by:jonas`, `approved-for-release:v0.1.0`. The system treats all of these uniformly. The branch a commit is "on" is whatever branch certs say it is on; the author is whatever author cert says it is.

The trust model is the leverage. When Aditi pulls from Jonas, she imports revisions and certs. Her local view of history is filtered by which signers she trusts. By default, signing keys are not trusted; she runs `mtn trust` to mark Jonas's key as trusted, after which his certs count toward her view. This is a different shape of trust than a centralized server's "the server is authoritative" or Git's "whoever pushes can write." It is, in particular, immune to compromise of any single party: if Jonas's key is stolen and used to forge a `branch:main` cert on a malicious revision, Aditi has the option to revoke trust in his key (or to trust only certs co-signed by another reviewer) without affecting the rest of the history.

The network protocol is *netsync*, a custom binary protocol with mutual authentication via the same RSA keys. Two Monotone instances connect, exchange the certs they have, and reconcile differences. There is no central server in the protocol; any instance can sync with any other.

Branches in Monotone are values of the `branch:` cert. Two heads on the same branch (a divergence) are routine and visible; the system supports working with multiple heads simultaneously, with `mtn merge` to bring them together.

The CLI is `mtn`. Subcommands include `mtn init` (create a database), `mtn add`, `mtn commit`, `mtn pull`, `mtn push`, `mtn sync`, `mtn checkout`, `mtn update`, `mtn log`, `mtn merge`, `mtn approve`, `mtn cert`, `mtn trust`, `mtn pubkey`, `mtn privkey`.

## Scenario walkthrough

### Operation 1 — Initial import

Aditi initializes a Monotone database and generates her key:

```
$ mtn db init --db=ledger.mtn
$ mtn genkey aditi@example.org
... prompts for passphrase ...
$ cd /path/to/initial/ledger
$ mtn --db=../ledger.mtn setup --branch=ledger
$ mtn add --recursive .
$ mtn commit --branch=ledger -m "Initial import"
mtn: beginning commit on branch 'ledger'
mtn: ... new revision a3f5...
```

The first commit creates a revision with no parent, signed by Aditi's key. Certs are attached: `branch:ledger`, `author:aditi@example.org`, `date:...`, `changelog:Initial import`. The revision hash names the entire repository state.

### Operation 2 — Linear development

```
$ vi src/ledger/parser.py
$ mtn commit -m "parser: handle blank lines and comments"
... new revision ...
```

Each commit is a new revision with the previous as parent. The branch cert keeps each commit on `ledger`. The hash chain is unbroken; tampering is detectable.

### Operation 3 — Branch

Jonas joins. Aditi gives him access; he generates his own key, pulls the database from her server, and trusts Aditi's key:

```
$ mtn db init --db=jonas-ledger.mtn
$ mtn genkey jonas@example.org
$ mtn --db=jonas-ledger.mtn pull mtn://aditi-host/?branch=ledger
$ mtn --db=jonas-ledger.mtn checkout --branch=ledger /tmp/work
$ mtn trust aditi@example.org
```

For the `currency-conversion` work, Jonas commits with a different branch cert:

```
$ vi src/ledger/parser.py
$ mtn commit --branch=ledger.currency-conversion -m "parser: store amounts as (value, currency)"
```

The revision is the next in the chain (parent is the latest `ledger` revision he had pulled), but its `branch:` cert is `ledger.currency-conversion`. To Jonas, this is now the head of a separate branch. The hierarchical naming convention (`ledger.currency-conversion`) is idiomatic Monotone.

### Operation 4 — File rename

```
$ mtn rename src/ledger/reports.py src/ledger/report_balance.py
$ # edit report_balance.py to remove register()
$ mtn add src/ledger/report_register.py
$ mtn commit -m "reports: split into balance and register modules"
```

Renames are recorded in the revision's change set. `mtn log src/ledger/report_balance.py` follows the rename through history.

### Operation 5 — Binary file added

Monotone has no special handling for binaries beyond suppressing line-ending normalization for files with binary content. The PNG is added with `mtn add docs/logo.png` and committed normally. Storage is content-addressed; if the file changes, both the old and new content are stored. Compression is applied where it helps.

### Operation 6 — Parallel edits

Aditi and Jonas both edit `README.md`, each in their own database. When they sync, the system has *two heads* on the `ledger` branch — one with Aditi's edit, one with Jonas's. This is normal in Monotone. The system reports it; the team responds with a merge.

```
$ mtn merge --branch=ledger
mtn: 2 heads on branch 'ledger'
... runs three-way merge tool ...
mtn: merging revisions a3f5... and b7c2...
mtn: created revision c9d1... as merge
```

The merge revision has two parents and is a normal commit; certs identify which user produced the merge.

### Operation 7 — Merge with conflict

Jonas merges his `currency-conversion` branch back. The mechanic is `mtn propagate`, which moves changes between branches:

```
$ mtn propagate ledger.currency-conversion ledger
... merges with conflicts on parser.py ...
$ # resolve conflicts in interactive merge tool
$ mtn commit -m "Merge currency-conversion into main"
```

`mtn propagate` is the idiom for "merge from branch X to branch Y" and is one of Monotone's more pleasant ergonomics.

### Operation 8 — Botched commit

Mireille's botched commit cannot be removed from history; the content-addressed model means the revision exists wherever it has been propagated, and a hash chain cannot be silently rewritten. Her options are:

1. **Disapprove the revision.** Attach a cert saying `disapproved-of:true` (or a similar custom cert by team convention) and commit a follow-up that backs out the bad change. Tools that filter the view by trust and certs hide the disapproved revision from normal output.
2. **Suspend the bad cert and re-cert.** Attach a `branch:suspended` cert to the bad revision (or whatever convention the team uses); the revision still exists but is no longer on the branch.
3. **Commit a follow-up.** Standard immutable-history approach.

The exemplar takes (3). Monotone's stance is the audit-trail one, with certs offering a structured way to annotate retrospectively without rewriting.

### Operation 9 — Tag

Tags in Monotone are certs:

```
$ mtn cert HEAD tag v0.1.0
```

Or, equivalently, attach a `tag:v0.1.0` cert to the revision. The cert is signed by the user; trust filtering applies.

### Operation 10 — Release

`mtn checkout --revision=tag:v0.1.0` produces the release tree. A tarball is generated from it. The release notes are committed and similarly cert-tagged.

## Model and mental load

What you have to hold:

- Content-addressed everything. A revision's hash names the entire history up to it.
- Certs as the metadata mechanism. Branch, author, date, changelog, tag, custom — all are certs.
- Trust as the filter. Your view of history depends on which keys you trust.
- The split between *the database* (which holds revisions, certs, keys) and *the working copy* (which is a checkout from the database).
- The netsync protocol. Pulling a remote means exchanging certs and revisions, with mutual authentication.

The mental load is moderate-to-high. The model is internally clean but unfamiliar; users coming from CVS or Subversion needed time to absorb the trust-and-cert idea. New developers in 2026 who came up on Git find the content-addressed parts familiar and the cert-based metadata strange.

## Evolution and history rewriting

History is structurally immutable. Certs can be added; certs cannot be removed except through a database-rebuild operation. The system is the strongest "history is forever" position in the book.

## Ecosystem reality

Monotone in 2026 is dormant. The reference implementation builds and runs; the project's mailing list has occasional traffic; the GitHub presence is sparse. There are no major projects on it. Its ideas live on in Git, in Pijul, in some of the discussions around Sigstore and the supply-chain provenance work.

## When to reach for it; when not to

Do not reach for Monotone for new work. The model is genuinely interesting and worth studying; the implementation is not maintained at the level needed for production use. If you want signed commits in 2026, use Git with Sigstore-backed signing; if you want trust-filtered history, build a layer on top of Git or wait for the supply-chain tooling to mature.

If you encounter a Monotone repository, install the system, read the tutorial, and migrate to Git. Migration tools are crude but functional.

## Epitaph

Monotone was the system that worked out content-addressed history before Git did, treated cryptographic identity as the foundation rather than as an afterthought, and is therefore the system whose ideas Git inherited without acknowledgement and whose practice the field forgot.
