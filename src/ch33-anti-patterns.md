# 33. Anti-Patterns and Migration Paths

This chapter is operational. It covers the version control anti-patterns the book has seen often enough to name, and the migration paths between systems with notes on what does and does not transfer cleanly. The chapter is for teams about to make decisions with consequences; the goal is to fail less often by knowing where the cliffs are.

## Anti-patterns

### Committing secrets

The most expensive anti-pattern in absolute terms, every year. A developer commits an AWS access key, an API token, a database password, a TLS private key. The credential ends up in a public Git history; bots scrape public Git history continuously; the credential is exploited within minutes.

The fix on detection is *immediate rotation* — assume the credential is compromised — followed by *history rewriting* with `git filter-repo` or BFG Repo-Cleaner, followed by force-push and notification to every contributor that they need to re-clone. Even after rewriting, the credential lived briefly on every host that mirrored the repository, which usually includes the platform's distributed cache. *Rotate, do not just rewrite.*

The prevention is `pre-commit` hooks (`gitleaks`, `truffleHog`, `detect-secrets`), platform-side secret scanning (GitHub Secret Scanning, GitLab Secret Detection), and a culture that treats committing a credential as an incident, not a mistake.

### Committing build artifacts

A `node_modules/` directory, a `target/` directory, a `dist/` directory, a `build/` directory committed to version control. The result is a repository inflated with content that should never have been versioned: the build artifacts dominate the storage, and every change to them produces an enormous diff that obscures the real change.

The fix is `.gitignore` that excludes the artifact directories, plus a one-time history cleanup if the artifacts are large enough to matter. The prevention is `.gitignore` from the start.

### Mixing source and generated code in the same path

Generated code (compiled outputs, generated parsers, generated documentation, generated language bindings) committed to the same paths as hand-written source. The generated content has to be regenerated on every build; it conflicts trivially in merges; reviewers cannot tell whether a diff is a real change or a generation artifact.

The fix is to either *generate at build time* (and not commit the generated files) or *generate to a clearly separated path* (`generated/`) with the source-of-truth in a different path. Tools like Bazel, Nix, and language-specific code generators handle this.

### Force-pushing to shared branches

`git push --force` to `main` or to a PR's branch that other people have pulled. The force-push silently rewrites history; pulls by other developers will produce confusing merge attempts; CI systems chase ghost commits.

The fix on detection is `git reflog` to find the original commit and re-push it, then a careful coordination with anyone who pulled the bad version. The prevention is *branch protection* rules at the platform level (block force-pushes to `main` and protected branches), and `git push --force-with-lease` instead of `--force` for the cases where rewriting your own branch is genuinely intended.

### Long-lived feature branches

A feature branch that lives for months, accumulating commits, diverging from `main`. Eventually it has to be integrated; the integration is enormous; the conflicts are pervasive; the review is impossible.

The fix is *integration before completion*: feature flags for in-flight work on `main`, regular merges from `main` into the branch (or rebases), and a discipline that no branch lives more than a week or two without integration. The prevention is the same discipline, instituted at the team's launch rather than after the disaster.

### Squashing all commits down to one on every merge

A team policy that every PR is squashed into a single commit on merge. The history of `main` is clean, but the *progression of thought* in the original commits is lost forever. A reviewer six months later cannot see how the contributor arrived at the final answer; bisecting through the PR is impossible because there is only one commit.

The fix, if the team values the original commits, is to allow merge commits or fast-forward merges instead. The trade-off is real: a clean linear history vs. a faithful record. Teams should pick deliberately.

The opposite anti-pattern — never squashing, leaving every "WIP" and "fix typo" commit in `main` — produces noisy history. The middle ground is interactive rebase before merging: the contributor cleans up their commits to a series of meaningful units, the maintainer merges them as-is.

### Monorepo when polyrepo is right (and vice versa)

A team with three independent products in three independent repositories that decides "we should have a monorepo" and merges them, despite the products having different release cadences, different team ownership, and different deployment targets. The result is a repository where most contributors don't care about most of the content, where CI builds everything on every change, and where ownership boundaries blur.

The opposite: a team with one logical product across thirty microservice repositories, where any meaningful change touches three of them, and the cross-repo PR coordination is an ongoing tax.

The right answer is to look at the team's *actual* boundaries: what builds together, what ships together, what is owned together. Repositories should align with those boundaries. Monorepo is the right answer when the team has one product and the build-ship-own boundary is one. Polyrepo is the right answer when those boundaries are multiple and stable. Either is wrong when forced against the actual structure.

### Submodules used as the wrong tool

Git submodules embed one repository in another by reference. The model has specific use cases (vendoring with explicit upgrade decisions, sharing code between projects with infrequent updates) and many cases where it is the wrong tool: when the embedded code is changing alongside the parent, when contributors need to make changes in both at once, when the build dependency is complex.

The fix is to use a *subtree merge* (which copies the embedded code into the parent's history) or a *monorepo-like* arrangement instead. The prevention is to ask, before adding a submodule, "do contributors need to make atomic changes across this and the parent?" If yes, submodules are not the right tool.

### Initial commits of enormous size

A new repository whose first commit is "imported the existing project," with hundreds of megabytes of accumulated content. The initial commit is so large that subsequent operations (clones, archives, blame) pay the cost forever.

The prevention is to *prune the import*: remove build artifacts, generated files, vendored dependencies that should be tracked separately, before the first commit. If the project genuinely has hundreds of megabytes of source-of-truth content, evaluate whether Git is the right tool or whether the binary content belongs in LFS, in Perforce, or somewhere else.

### Vendoring everything (or nothing)

Vendoring third-party dependencies into the repository — copying their source into a `vendor/` or `third_party/` directory — solves reproducibility (the build does not depend on external packages being available) at the cost of repository size and update overhead. Some projects vendor everything; some vendor nothing.

The right answer is *vendor what cannot be reproducibly fetched*. Modern package managers with lock files (npm, pip, cargo, go modules) handle reproducibility without vendoring most of the time. Vendoring should be reserved for cases where the package manager is insufficient, where the dependency is genuinely customized, or where the build environment requires it.

### Branch naming that does not scale

A team starts with `feature/widget`, `feature/gadget`, `feature/thingy`. After two years, there are 400 feature branches, half abandoned, no consistent owner attribution. The branch list is unusable.

The fix is *prefix-by-author* (`alice/widget`, `bob/gadget`) and *automatic deletion of merged branches*. Most platforms can delete branches automatically on PR merge; the option should be on by default for the team.

### Tagging without annotation

Lightweight tags (just a ref pointing at a commit) carry no metadata. Tagging a release with `git tag v1.0` and pushing it produces a tag without an author, message, or signature. Six months later, no one knows who tagged it or why.

The fix is *annotated tags*: `git tag -a v1.0 -m "First public release"`. Signed annotated tags (`git tag -as`) provide cryptographic authentication of who tagged what.

### Ignoring the reflog

A developer accidentally `git reset --hard`s away their work. They believe it is gone; they panic; they redo the work from scratch.

The fix is `git reflog show`, which shows every recent change to refs, with the orphan commit's hash recoverable. The prevention is knowing the reflog exists.

### Failing to delete branches after merge

Stale branches accumulate in the platform UI; the branch list grows unmanageable; new contributors cannot tell which branches are active.

The fix is automatic deletion on merge (a platform option) and periodic cleanup of branches that have not been pushed to in N months.

### Treating CI as a check, not a contract

A team that uses CI as a "is this passing right now?" check, with unstable tests that flake intermittently, with the merge button greying out and lighting up depending on the test mood. The CI's signal is unreliable; the team's trust in it erodes; flaky tests become the norm.

The fix is *treating CI as a contract*: a failing test is a defect, either in the code or in the test; flaky tests are bugs that need fixing or quarantining (in a separate suite that does not gate merges); the green-CI requirement is non-negotiable. The discipline is hard to establish but worth establishing.

## Migration paths

### CVS to Subversion

The canonical migration of the early 2000s. `cvs2svn` (now part of `cvs2git`) reads a CVS repository and produces a Subversion dump file with full history. The migration is mature; quirks (vendor branches, file mode bits, missing commit boundaries) are well-documented.

The migration is essentially complete in 2026; few CVS repositories remain to migrate, and most that do are read-only archives.

### CVS to Git

`cvs2git` (the same project) produces a Git repository directly. The history quality is good; commits are reconstructed from per-file CVS revisions with timestamp-based heuristics (CVS has no atomic commits, so the migration tool has to infer commit boundaries).

The cleanup after migration involves consolidating committers' email addresses (CVS used Unix usernames; Git wants email addresses), normalizing line endings, and configuring `.gitattributes` and `.gitignore`. The result is a Git repository that has the CVS history; it does not have CVS-era issues, mailing lists, or release archives.

### Subversion to Git

`git svn clone` produces a Git repository from a Subversion repository, preserving history. For a single-branch Subversion repository, the migration is straightforward. For a repository with the standard `trunk/branches/tags` layout, `git svn clone --stdlayout` translates the structure: `trunk` becomes `main`, branches become Git branches, tags become Git tags.

Quirks: Subversion's user-visible features (svn:externals, svn:eol-style, svn:executable) need translation to Git equivalents (.gitmodules for externals, .gitattributes for eol-style, file mode for executable). Subversion's user-friendly revision numbers do not translate; Git's hashes are the new identity.

The migration is mature. The challenge is usually not the technical migration but the cultural one: teams used to Subversion's mental model take time to learn Git's.

### Mercurial to Git

`fast-export` (from the `fast-export` project), `hg-git` plugin, or `git-cinnabar` (Mozilla's tool, optimized for very large Mercurial repositories). All produce reasonable Git histories from Mercurial repositories.

For Mercurial repositories with extensions (mq, evolve, largefiles), the migration is more involved; some structural information is lost. For repositories without extensions, the migration is clean.

The cultural migration is small in this case: Mercurial users find Git mostly familiar, with names changed.

### Perforce to Git

`git p4` is part of Git itself, supporting bidirectional sync between a Git repository and a Perforce depot. Full one-way migration uses `git p4 clone <depot>`, which produces a Git repository with Perforce changelists translated to commits.

Caveats are larger than Subversion's migration. Perforce's branch model (streams or branch specs) does not translate cleanly; binary asset locking is lost; the depot's typemap and other server-side configuration has no Git equivalent. Many teams that migrate from Perforce to Git also migrate from a binary-and-text repository to a text-only repository, with the binaries moving to LFS, to a separate Perforce instance, or to dedicated asset storage.

The migration is sometimes wise, sometimes a mistake. The framework in Chapter 32 should be applied to the post-migration target, not just the pre-migration source.

### ClearCase to Git

Several tools — `clearcase2git`, `git-clearcase`, custom scripts — extract ClearCase history into Git. The migration is laborious. The config-spec model does not translate; the merge-hyperlink history does not translate; the per-element branch nuances flatten. Teams typically accept some loss of fidelity in exchange for the ability to leave.

### Bazaar to Git

`bzr-git` plugin (or, for Breezy, `brz-git`) handles bidirectional operation between Bazaar and Git. Full one-way migration is well-supported. The history transfers cleanly; rename information (Bazaar's file IDs) is preserved through Git's commit metadata for the move operation.

### Visual SourceSafe to Git

`vss2git` and similar tools extract VSS history into Git with various levels of fidelity. The fidelity is, on average, lower than other migrations because VSS's storage was less rigorous and many real-world VSS databases have corruption that the migration tool must work around. Some VSS history simply cannot be cleanly extracted; teams accept this.

### Git to Sapling, or Git to Jujutsu

These are not migrations; they are *adoptions* with the substrate unchanged. The Git repository remains; the new tool reads and writes the same `.git/` directory; the team's existing tooling continues to work. Adoption is per-developer; some developers can use Sapling or Jujutsu while others stay on Git, against the same repository.

### Migration data that does not transfer

In any platform migration (GitHub to GitLab, GitLab to Gitea, GitLab to Forgejo, GitHub to Bitbucket, anywhere to self-hosted), the *Git history* transfers cleanly. The platform-specific data — issues, pull requests, comments, CI configurations, packages, releases, security advisories — transfers in varying degrees.

Most platforms provide *export tools* that produce JSON-like dumps of issues and pull requests. Most platforms provide *import tools* that ingest JSON-like dumps. The two ends rarely match cleanly. References between issues and PRs typically break (issue #42 on the source platform points at issue #42 on the target only if the migration preserves IDs, which it usually does not). User accounts on the source typically do not map to user accounts on the target; references to authors become text labels.

The realistic plan for a platform migration:

1. Migrate the Git history first; verify it.
2. Export issues, PRs, and other platform data.
3. Import to the target with awareness of the fidelity gaps.
4. Update inbound links (Stack Overflow, blog posts, documentation) to the new URLs.
5. Maintain a redirect from the old to the new for as long as is practical.
6. Accept that some data will be effectively orphaned; document what.

### When *not* to migrate

If the current system is working and the team is productive, the migration cost may exceed the benefit. The framework in Chapter 32 should evaluate the *post-migration state* against the *current state plus migration cost*. Migrations driven by perceived modernity rather than concrete pain are usually mistakes.

The pain that justifies migration is concrete: the current system has security holes that are not being patched; performance is unworkable; the contributor base cannot find the project because of platform restrictions; a hard-deadline-bound vendor is exiting; regulators are mandating change. Each of these is a real reason. "Everyone uses Git now" is, by itself, not.

## Epitaph

Anti-patterns are version control's accumulated debt; migrations are version control's unpaid promises. Both are tractable when named, both expensive when ignored.
