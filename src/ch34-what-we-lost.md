# 34. What We Lost When Git Won

This chapter is an inventory. It is not a complaint about Git; Git's chapter said its piece, and the system has earned its place. The inventory is of ideas — design positions, workflow primitives, mental models, properties of older systems — that have been buried or made invisible by the dominance of one position in the design space, and that working programmers in 2026 mostly have not encountered.

The chapter's tone is intended to be honest, not nostalgic. Some of these ideas were better than what the field replaced them with; some were worse; some were merely different and could profitably exist alongside what we have. The point is to make them visible, so that anyone choosing what to build, what to use, or what to teach has the option of reaching for them.

## 1. Patches as identity-bearing objects

In the patch model — Darcs, Pijul, and to a different degree Sapling and Jujutsu through change-IDs — a *change* is a thing that exists, separate from the bytes it currently produces. Cherry-picking is applying the same patch in a different repository; the patch keeps its identity. Reordering is checking commutation; the system knows when two patches are independent. Conflicts are the algebraic failure of two patches to commute, recorded as objects.

This is not what Git does, and it is not what the field's mental model includes. Most programmers have never encountered the idea that a *change* could have an identity that survives rebasing without explicit reflog tracking. Sapling and Jujutsu have begun to bring change-IDs into the snapshot world; they are not yet widespread.

What survives: Pijul (small but alive), Darcs (small but alive), Sapling and Jujutsu's change-IDs (growing), the `Change-Id:` trailer convention in Gerrit (widespread in Gerrit-using projects but not elsewhere). What is lost: the default mental model that a change is a thing, not a delta against a parent.

## 2. Mercurial's UX clarity

Mercurial's command line was, by intent and by execution, easier to use than Git's. Errors suggested fixes. Commands were consistent. The basic operations — commit, push, pull, merge, branch — worked the way users expected most of the time, with fewer footguns. New users were productive faster.

This is the loss the broadest set of working programmers would notice if they tried the alternative. Most Git users have absorbed the system's quirks (the difference between `checkout` and `switch`, the three modes of `reset`, the `commit --amend` corner cases, the ambient confusion about what `pull` does by default) as the cost of doing business. They are not, in fact, intrinsic costs of version control. Mercurial demonstrated that they were not, and Mercurial lost.

What survives: the Mercurial project continues; Sapling inherits much of the UX sensibility; Jujutsu is consciously friendlier than Git. What is lost: the cultural default that version control should not be a hazing ritual.

## 3. Fossil's bundled simplicity

Fossil packages version control, bug tracking, wiki, forum, web UI, and HTTP server into a single binary that runs on any reasonable platform with no external dependencies. A small project's complete infrastructure is one file plus one SQLite database. Backups are file copies. Migration is a file move. Hosting is `fossil server`.

This is the unification the platform era promised and delivered in a different way: GitHub gives you the same set of features, *bundled with the platform*, with the platform's lock-in. Fossil gives you the same set of features, *bundled with the binary*, with no platform.

What survives: Fossil itself, used by SQLite and a small set of other projects. The idea is preserved; the practice is rare.

## 4. Perforce's honest centralization

Perforce's model — one server, one canonical truth, workspaces with mappings, file types with explicit semantics, exclusive locks on binaries, monotonic changelist numbers, audit-trail by default — is a coherent expression of the position that *centralized authority is a feature, not a defect*. For game studios, film studios, hardware design teams, and anyone with substantial binary content, the centralized model is structurally better than the distributed-but-effectively-centralized model that Git plus a forge produces.

The DVCS revolution presented centralization as a problem to be solved. For the use cases Perforce serves, it was not a problem; the distributed-with-platform-as-server arrangement Git settled into is, for those use cases, a worse version of what Perforce already offered.

What survives: Perforce / Helix Core, dominant in games and film. Plastic SCM / Unity Version Control, smaller but growing. Subversion, in slow decline. The position remains occupied; the position is rarely chosen by teams who do not already know they need it.

## 5. The "files have versions" mental model

In RCS and CVS, the unit of history was a file. A file had revisions; a revision had a number; you could ask "what was this file at revision 1.4?" without thinking about commits, trees, or graphs. The model was simple, intuitive, and in many cases correct enough.

The unit-is-the-project model that Git and Subversion settled on is more powerful and is the right model for most projects. It is also less direct: ask a junior programmer "what was the version of `parser.py` six months ago?" and they reach for `git log`, find the commit, look up the file in that commit. In RCS, the same operation was `co -r1.4 parser.py`. The simpler operation is gone.

What survives: RCS still ships in most Unix distributions; sysadmins still use it for one-file configurations. The mental model is mostly invisible to programmers under thirty.

## 6. SCCS's interleaved delta storage

The weave format — every revision of a file stored in a single text file, with each line marked by which deltas added or removed it — is the most elegant storage format any version control system has shipped. Retrieval of any version is linear in file size; the format is human-readable; the storage is compact. In fifty years, no major version control system has copied it.

What survives: SCCS still exists, AIX still ships it, CSSC reimplements it. The format is academically interesting; the engineering practice is dead.

## 7. The Mercurial evolve / Sapling / Jujutsu lineage on history mutability

Mutable history with full provenance — the system tracks that this commit replaced that one, who did the rewrite, when, and why — is qualitatively different from Git's "rewrite and rely on the reflog" approach. Pulling a remote with rewritten history that I have a local copy of: in evolve / Sapling / Jujutsu, the system knows that the new commit *is* the old one's successor; in Git, I have to figure that out manually or with `git rebase --autosquash` heuristics.

What survives: Mercurial evolve (extension, alive but not default); Sapling (Meta's, open-source, growing); Jujutsu (the most actively recommended). The default Git ecosystem has not adopted this. It could; the work has been done; the path is clear.

## 8. ClearCase's config specs

A *config spec* is a declarative rule for what version of every file appears in your view. "Prefer my checkouts; otherwise use the latest on `currency-conversion`; otherwise use the latest on `main`" is one config spec. "Use the version labeled `release-1.0` on these files; latest on `main` for everything else" is another. Switching context means changing the spec.

The expressive power of this is large. Builds with non-trivial release engineering needs — "this build is a special hotfix variant, with these specific changes layered on the v1.0 line" — are routine in a config-spec system and elaborate in Git.

What survives: ClearCase, in legacy maintenance. The idea has not been picked up elsewhere. Sparse checkout in Git approaches one corner of it; nothing reaches the full expressivity.

## 9. Bazaar's flexibility of workflow

Bazaar treated centralized vs. distributed as a per-branch choice, not a system-level commitment. A bound branch behaved like Subversion (every commit pushes to a master); an unbound branch behaved like Git (commits are local until pushed). The same client served either model, with the choice made per branch.

The position — that the centralization axis is a *workflow* axis, configurable per use, not a *system* axis decided once — is one nobody else has occupied seriously.

What survives: Breezy (the Bazaar fork) implements the model; it is used in some Debian-adjacent infrastructure. The idea is unfashionable.

## 10. The email-patch workflow as a default

`git format-patch` and `git send-email` still work in 2026. The Linux kernel still uses them. So do FreeBSD, OpenBSD, NetBSD, parts of the Wayland project, and a handful of others. The workflow is portable, archive-friendly, asynchronous in a way pull requests are not, and tool-agnostic.

For most working programmers, the workflow is invisible. They have never sent a patch by email; they have never received one; the tooling on their machine for the workflow is unconfigured. The workflow has been quietly preserved by projects that never abandoned it; it is not visibly present in the field's idea of how contribution happens.

What survives: kernel.org, the BSD projects, Sourcehut. The cultural form is alive; the cultural awareness of it is shrinking.

## 11. Monotone's certs and trust filtering

Monotone's idea — that *anyone* can sign assertions about a commit (this commit is on this branch, this commit has been reviewed, this commit is approved for release), and that *the user* decides which signers to trust — is a different model of authority than what the field settled on. The forge model has *the platform* as the authority; everyone trusts the platform. Monotone's model lets trust be distributed and filtered.

What survives: Monotone is dormant; the idea lives on partially in Sigstore-style transparency logs and supply-chain provenance work. The full vision — version control with first-class certificate-based trust — is not implemented anywhere active.

## 12. Per-file pessimistic locking as a default option

The systems of Chapter 10 — VSS, Vault, StarTeam, PVCS — offered pessimistic locking on files as a default-available option that worked well for binary content. Git LFS bolted locking on after the fact, with weaker guarantees. Perforce kept it; Plastic kept it; the rest of the field forgot.

For teams whose work involves files that cannot be merged, locking-as-default is not a curiosity; it is what the version control system should provide. Git's defaults make it harder for those teams than it needs to be.

What survives: Perforce, Plastic, Subversion (with `svn:needs-lock`), Git LFS (with caveats). The idea is preserved in specialized systems; the cultural default is "everyone can edit, merge later," even when that is wrong.

## 13. Integrated bug tracking inside the version control system

Fossil ships with a bug tracker as part of the tool. Plastic has integrated review and tracking. ClearQuest paired with ClearCase; Mercurial paired with Bugzilla through plugins; Trac paired with whatever you wanted. The forge era moved this *out* of the tool and *into* the platform — and made it the platform's concern, with the platform's lock-in.

Fossil's small-project completeness — code, bugs, wiki, forum, all in one binary, all in one SQLite file — is preserved as a position the field could re-occupy. The integration is real; the lock-in does not have to come with it.

## 14. The composability of pre-platform infrastructure

The pre-platform world had specialized tools for each function: Bugzilla for bugs, Mailman for discussion, Trac or a custom site for the project page, Jenkins for CI, Apache for hosting. The integration was loose. The cost was assembly; the benefit was independent evolution.

The platform world has unified those into one product. The cost is lock-in; the benefit is integration.

The composability is *recoverable* but *expensive*: a project that wants Linear for issues, Jenkins for CI, and GitHub for source can have it, with bots and integrations to tie things together. The composability is no longer the default; it is a deliberate posture against the default.

## 15. The kernel's "no version control system, just patches and people" model

For ten years, the Linux kernel's version control was patches and a small number of trusted lieutenants reading and forwarding them. The system worked. It scaled to thousands of contributors. It did not have GitHub; it did not have Gerrit; it did not have any of what 2026 considers infrastructure.

The model is preserved in the kernel's BitKeeper-then-Git years: lkml is still where review happens; patches still arrive by email; maintainers still curate. But the model as the *primary* version control story — that a version control system, in some sense, is *not the most important piece of the contribution flow* — is largely invisible to a generation of programmers who have only worked through pull requests.

What survives: the kernel itself, BSD projects, a handful of others. The model's existence is preserved; its plausibility as a starting point for new projects is not.

## 16. The understanding that version control is plural

The most subtle loss, and the one this book exists to address. Programmers who came of age after 2010 rarely encounter the idea that *version control is a category with multiple intelligible answers*. They learn Git. They use Git. They argue about Git workflows. The category collapses into the product; the design space becomes invisible.

The loss is not Git's fault; it is the byproduct of dominance. The recovery is the act of looking at the design space again — at the systems before Git, at the systems alongside Git, at the systems that came after Git and learned from it. The recovery is what the book hopes to be a small contribution to.

## What this is not

This is not an argument for going back. The world before Git had its own problems, and most of them were genuine. CVS was bad in specific ways. SourceSafe was bad in many ways. The patch-by-email workflow scales but requires discipline that not every team has. The mailing list culture that produced excellent kernel development also produced toxic flame wars. ClearCase's expressive power came with operational pain that hurt real organizations. Nostalgia is the wrong frame.

This is not an argument that Git should not have won. Git won by fitting a moment, and the moment was real. The kernel needed something that worked at its scale, in 2005, with the constraints of the time. Git was the right answer for that situation. GitHub was the right amplifier for the answer. The combination is the dominant arrangement of 2026 because the combination has continued to fit a large enough share of the field's actual work.

This is not even an argument that any team should leave Git. Most of the readers of this book will continue to use Git for the foreseeable future, and that is fine. The book's claim is the modest one: knowing that Git is one position in a design space, with specific costs that other positions don't share, makes the position chosen rather than inherited. The team's understanding of *why* they use Git is more durable than the team's habit of using it.

## What might still happen

The design space is not closed. The book's evidence:

*Jujutsu* is gathering momentum, and its workflow primitives — change-IDs, the operation log, first-class conflicts, working-copy-as-commit — are being absorbed by users who are then articulate about wanting them in Git. Some of those primitives may eventually land in Git proper.

*Sapling* has demonstrated that change-IDs can work at very large scale, and the experience is shaping how Meta-style monorepos can operate. The lessons are public; they are available for adoption.

*Pijul* is the one serious attempt to bring patch theory back into the field's working vocabulary. Its growth is slow but visible. If a moment arrives where free cherry-picking, structural renames, and conflicts-as-objects matter enough to a sufficiently large project, Pijul (or something like it) is positioned to receive the attention.

*The forge alternatives* — Codeberg, Sourcehut, self-hosted Forgejo — continue to grow and continue to demonstrate that the platform model is not the only model. Their growth is not threatening GitHub's dominance, but they are sustaining the position that the dominance is conditional, not natural.

*Supply-chain security* is forcing the field to take provenance and signing seriously. Sigstore, SLSA, and the work around them touch territory Monotone explored decades ago. The certs-and-trust model may yet make a return through the supply-chain door.

*The AI-assisted code-generation moment* — Copilot and its successors — is changing what a commit *is*, in ways the field has not finished thinking about. Some of the change may push toward the snapshot model harder; some may push toward patch-theoretic ideas (a generated change has identity that survives regeneration). The space is being re-explored.

The version control conversation is not closed. The book's contribution, if it has one, is the modest claim that the conversation never *should* close — that a fifty-year-old field with this much working diversity is one whose participants benefit from knowing the breadth of the conversation, not just the loudest speaker.

## Epitaph

What we lost when Git won was the visibility of the design space, the knowledge that there were other answers, the cultural memory of what those answers were good for. The systems are still here. The ideas are still here. They are recovered by being read, by being tried, by being treated as live options rather than historical curiosities. This book is a small contribution to that recovery. The design space is open. The conversation continues.
