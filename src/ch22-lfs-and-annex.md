# 22. Git LFS and git-annex

## Origin

Git's storage model treats every file as a blob, content-addressed and replicated to every clone. The model works beautifully for source code, where files are small and the deduplication of unchanged content keeps repositories tractable. The model fails for binary content: large files do not deduplicate (each version is a fresh blob), they do not delta-compress meaningfully (PNGs and binary models are essentially incompressible), and they replicate to every clone whether anyone needs them or not. A repository with a few hundred megabytes of code and a few gigabytes of art assets balloons over time into something nobody can clone.

Two systems address this by layering over Git. *git-annex*, by Joey Hess, started in 2010 and approached the problem with maximum generality: track files' content separately from their existence in the tree, with multiple storage backends and selective replication. *Git LFS*, announced by GitHub in 2015, took a narrower position: replace large files in the repository with pointer files, store the actual content on a separate LFS server, and provide a transparent-enough experience that most users would not have to think about it.

Both are widely used. They solve overlapping but not identical problems. The chapter covers both, with an honest comparison of when each is the right choice.

## Git LFS

Git LFS — *Large File Storage* — is now standard infrastructure across GitHub, GitLab, Bitbucket, Gitea, Sourcehut, and most other hosted Git forges. The protocol is documented and openly implemented; the reference client is `git-lfs`, which installs as a Git filter and runs transparently on add, commit, push, pull, and checkout.

The model: when a file matches a configured pattern (`*.psd`, `*.bin`, `*.fbx`, etc.), Git LFS intercepts. On `git add`, the file's content is hashed (SHA-256), uploaded to the LFS server (on push), and replaced in the Git repository with a small *pointer file* containing the hash, size, and version metadata. The pointer file is what Git versions; the actual content lives on the LFS server, reachable by hash.

On checkout, the LFS filter intercepts, sees the pointer files, fetches the corresponding content from the LFS server, and writes the actual files to the working tree. The user sees the files. The Git repository sees only pointer files.

The configuration lives in `.gitattributes`:

```
*.psd filter=lfs diff=lfs merge=lfs -text
*.fbx filter=lfs diff=lfs merge=lfs -text
*.bin filter=lfs diff=lfs merge=lfs -text
```

Once configured, LFS is mostly transparent. The user adds, commits, pushes, pulls; Git LFS handles the routing.

The locking protocol (added in 2016) provides the missing piece for binary asset workflows: `git lfs lock path/to/file` claims a server-side lock on a file; other users see the lock and are prevented from making conflicting edits; `git lfs unlock` releases it. The locks are advisory by default — you can override with `--force` — but in practice teams treat them as exclusive.

LFS's costs:

- *Cost shifted to the host*. The LFS server stores all the binary content; storage costs scale with project size. GitHub charges for LFS bandwidth and storage past a free tier; self-hosted LFS servers (Gitea, GitLab, Sourcehut) inherit the storage cost.
- *Bandwidth amplification*. Pulling a repository with LFS-tracked files downloads the latest content for each. Sparse checkouts and `git lfs fetch --recent` can limit this, but the default fetches everything.
- *Filter overhead*. Every Git operation that touches LFS-tracked files goes through the filter. For working trees with many such files, the overhead is noticeable.
- *Migration is a one-way trip in practice*. Once a project has LFS-tracked content, removing it requires `git lfs migrate` and a force-push of rewritten history. Teams that accidentally LFS-track the wrong files have a cleanup task.

LFS is the standard answer for the "Git plus some big binaries" case. It is not the answer for the "Git plus terabytes of binaries" case; that case wants Perforce or Plastic.

### Scenario walkthrough — operations 5 and beyond

The exemplar's binary file is `docs/logo.png`, 24 KB. This is *too small* to need LFS by sensible thresholds; the chapter shows the operation anyway, because the workflow is the point.

```
$ git lfs install         # one-time setup
$ git lfs track "*.png"
$ git add .gitattributes
$ git add docs/logo.png
$ git commit -m "docs: add project logo"
$ git push
... LFS uploads logo.png separately, pushes pointer file in commit ...
```

For locking on binary assets:

```
$ git lfs lock docs/logo.png
... server records the lock ...
... other users running `git lfs locks` see Mireille's lock ...
$ # edit, commit, push
$ git lfs unlock docs/logo.png
```

The other operations from the exemplar play out as standard Git, with LFS transparently handling binary file content.

## git-annex

git-annex took a different approach to the same problem. Joey Hess, the system's author, was already a prominent Debian developer (and an author of `etckeeper`) when he started git-annex in 2010. The system is written in Haskell and is currently maintained by Hess and contributors at `git-annex.branchable.com`.

The premise: *separate the metadata about a file (its existence, name, hash) from the file's content*. The metadata lives in Git, replicated to every clone. The content lives in *some* clones, not necessarily all of them, with the system tracking which clones have which files. A clone might have everything, or nothing, or a curated subset; sync operations exchange both metadata and (selectively) content.

The implementation: tracked files in the working tree are *symlinks* to content in `.git/annex/objects/`. The symlink target is a name derived from the file's hash. The annexed content lives in `.git/annex/objects/`; clones that have not retrieved the content have the symlink without the target, manifesting as a broken symlink that users can request via `git annex get`.

The storage backends are diverse: local filesystem, rsync, SSH, WebDAV, S3, Glacier, IPFS, BitTorrent, and several specialized backends for archive material (the `bup` backend for incremental backups, the `web` backend for downloading from URLs). A single git-annex repository can have content in multiple backends; the system tracks which backend has each file.

The model is more flexible than LFS's and substantially more complex. The complexity is the cost of generality. git-annex repositories can serve as personal photo archives that selectively replicate to a phone, scientific data repositories where labs have read-only access to upstream content, distributed backup systems that span friends' servers, and conventional code repositories with large attachments.

### Scenario walkthrough — git-annex specifics

To bring the logo PNG under git-annex:

```
$ git annex init "aditi's laptop"
$ git annex add docs/logo.png
$ git commit -m "docs: add project logo"
$ git annex sync --content
... pushes both metadata and content to the configured remote ...
```

`git annex add` replaces the file in the working tree with a symlink and moves the content into `.git/annex/objects/`. The metadata is committed to Git; the content is stored separately. Other clones, on `git annex sync`, receive the metadata and can fetch the content with `git annex get docs/logo.png`.

The locking model is implicit: with `git annex` files in *direct* mode, files are read-only by default and `git annex unlock` makes them writable. The locking is closer to RCS's mode-bit-based approach than to LFS's server-coordinated locking.

git-annex's distinctive feature is `numcopies` — the system tracks how many copies of each file exist across all known remotes and refuses to delete the last copy of a file unless the user overrides. This is the kind of safety property that matters for archival material and that no other system in the book provides natively.

## When to use which

Use **Git LFS** when:

- You are on a hosted forge (GitHub, GitLab, etc.) that provides LFS as a service.
- The team is Git-fluent and wants the binary handling to be transparent.
- You need locking on binary assets and Git LFS's locking is sufficient.
- The total binary content is in the gigabytes, not the terabytes.

Use **git-annex** when:

- You need selective replication: clones with different subsets of content.
- You have data that lives in multiple storage backends (S3, archive tape, FTP, etc.) and want a single tracking layer.
- You need `numcopies`-style safety properties for archival content.
- The team is willing to invest in learning a more complex system in exchange for more flexibility.

Use **neither** when:

- The binary content is so large or central that the right answer is Perforce, Plastic SCM, or DVC (next chapter), not Git plus a binary layer.
- The team is small, the binaries few, and adding a binary layer is overkill.

## What both systems share

Both LFS and git-annex are *layers* on Git, not replacements. They preserve Git's history model, branching, merging, and most user idioms. They add a separate channel for content that does not fit Git's storage model and a coordination layer for the metadata about that content.

The honest read is that both exist because Git's blob storage was not designed for binary content, and the field built workarounds rather than redesigning the storage model. Perforce and Plastic, where binaries are first-class from the storage layer up, do not need LFS or annex. The systems with binary-aware storage handle the case more cleanly; the systems without it (Git, Mercurial) require add-on layers.

The lesson for the book: the snapshot model's deduplication-by-content-hash is excellent for source code and inadequate for binary content, and the field's response was not to fix the model but to layer something on top. The layering works. It is also a tax that gets paid every time a Git project grows binary content beyond the comfortable size, and the tax is one of the things on the inventory in Chapter 34.

## Ecosystem reality

Git LFS is universal. Every major hosted forge implements the protocol. The reference client is mature; bugs are rare; performance is acceptable. The hosting cost has been a perennial discussion point but is not a structural barrier.

git-annex is alive and has a dedicated user base. Hess continues to maintain it. The user base is concentrated in scientific data management, personal archival use, and a long tail of unusual workflows. New adoption is steady but small.

## Epitaph

Git LFS and git-annex are the answers Git gave to "what about big files?" — one narrow and uniform, one general and complex — and they are the visible evidence that Git's blob model was a code-shaped answer that the rest of the digital world had to be force-fit into.
