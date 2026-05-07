# Appendix D. Further Reading

Annotated references for the systems and topics covered. Where a primary source exists (an author's paper, a project's official documentation), the primary source is preferred. Secondary sources are included where they offer something the primary does not — context, comparison, or correction. URLs are omitted for sources whose canonical addresses are stable enough to find through search; addresses change, search engines do not.

## General references

**Eric Sink, *Version Control by Example*.** A free online book, originally written to accompany Vault, but with discussion of multiple systems. A friendly introduction to the field for readers who are starting from CVS or no version control at all.

**Scott Chacon and Ben Straub, *Pro Git*.** The canonical Git reference, free online, multiple editions. The internals chapters are unusually good for understanding what Git is doing under the porcelain.

**Bryan O'Sullivan, *Mercurial: The Definitive Guide*.** The canonical Mercurial reference, also free online. Out of date in a few specifics but accurate on the model.

**Karl Fogel, *Producing Open Source Software*.** Free online, regularly updated. The chapter on version control is brief; the book's broader treatment of open-source development culture is what makes it useful here.

## Chapter 4 — SCCS

**Marc Rochkind, "The Source Code Control System."** *IEEE Transactions on Software Engineering*, 1975. The original paper. Available through IEEE; older copies float around.

**The SCCS man pages on AIX, Solaris, and the GNU CSSC project's documentation.** For working with surviving SCCS repositories.

## Chapter 5 — RCS

**Walter F. Tichy, "Design, Implementation, and Evaluation of a Revision Control System."** *Proceedings of the 6th International Conference on Software Engineering*, 1982. The paper that defined RCS.

**The RCS man pages.** `rcsfile(5)` documents the file format clearly and is worth reading for the storage-format discipline alone.

## Chapter 6 — CVS

**Per Cederqvist et al., *Version Management with CVS*.** The CVS manual, distributed with the source. Comprehensive.

**Karl Fogel, *Open Source Development with CVS*.** The book that taught a generation of open-source developers what CVS was. Out of print but findable; the introductory chapters remain useful for understanding the era.

## Chapter 7 — Subversion

**Ben Collins-Sussman, Brian W. Fitzpatrick, C. Michael Pilato, *Version Control with Subversion*.** The SVN Book. Free online, Creative Commons licensed, multiple editions tracking Subversion versions. The canonical reference.

**Apache Subversion's documentation.** Current release notes, FAQs, and migration guides.

## Chapter 8 — Perforce

**Perforce documentation at perforce.com.** The Helix Core administrator's guide and command reference are comprehensive. The streams documentation and the file types reference are essential for understanding the model.

**Larry Wall's classic perl-related Perforce posts**, mostly for historical context on the system's early years and culture.

## Chapter 9 — ClearCase

**IBM Rational ClearCase documentation** (legacy, but largely preserved on IBM and HCL sites).

**The "ClearCase Wisdom" community archives** — independent collections of administrator notes, scripts, and patterns that together constitute much of the practical knowledge of running large ClearCase deployments. Findable but scattered.

## Chapter 10 — Vault, VSS, and the corporate backwater

**Joel Spolsky's mid-2000s posts on Visual SourceSafe.** Polemical but mostly accurate; the *Joel on Software* archives are the easiest place to find them.

**Eric Sink's writings at SourceGear.** The Vault rationale and various reflections on commercial version control. Sink's blog is worth reading independently of the Vault material.

**Vendor documentation** for StarTeam (Micro Focus / OpenText), PVCS / OpenText Version Manager, PTC Codebeamer, OpenText Synergy, AccuRev. Each is the primary source for its system.

## Chapter 11 — BitKeeper

**Larry McVoy's blog (`mcvoy.com`) and BitMover's documentation.** Primary source for the technical design.

**The `linux-kernel` mailing list archives, February 2002 through April 2005.** The decision to adopt BK, the years of use, and the revocation are visible there in primary form.

**Andrew Tridgell's posts** on the `sourcepuller` work, available through linux-kernel and his personal site.

**The 2005-04-06 lkml thread "Kernel SCM saga.."** (subject line approximate) where Linus announces the situation and opens the discussion that becomes Git.

## Chapter 12 — Monotone

**Graydon Hoare's *The Monotone Tutorial* and *Monotone Whitepaper*.** Both available through the Monotone project's site (`www.monotone.ca` historically; mirrors elsewhere).

**Hoare's writings on cryptographic identity in version control.** The ideas reappear in his later work on Rust's release process.

## Chapter 13 — GNU Arch

**The GNU Arch and tla manuals.** Largely of historical interest; the project's documentation lives on FSF mirrors.

**Tom Lord's writings**, scattered across mailing list archives. Findable; rewarding to read for the ambition of the original design.

## Chapter 14 — Bazaar

**The Bazaar documentation** at `bazaar.canonical.com` (now mirrored elsewhere as Canonical reduced its hosting).

**The Breezy project's documentation** for the current fork, at `breezy-vcs.org`.

**Martin Pool's blog and writings** on the project's history.

## Chapter 15 — Mercurial

**Bryan O'Sullivan, *Mercurial: The Definitive Guide*.** Free online.

**Matt Mackall's `hgbook` and the project's official documentation.**

**Pierre-Yves David's writings on `evolve`** are the best primary source on mutable history with provenance.

## Chapter 16 — Git

**Scott Chacon and Ben Straub, *Pro Git*.** Free online; current.

**The `git` source tree's `Documentation/technical/` directory.** Contains the most accurate descriptions of Git's internals: the object format, the pack format, the protocol.

**Linus Torvalds's `git` initial commit messages** for the original design rationale, in the project's own history (`git log --reverse --no-merges` will get you there).

**Junio C. Hamano's "What's cooking"** posts on the git mailing list, ongoing for 20 years, are the primary source for Git's continuing evolution.

## Chapter 17 — Fossil

**The Fossil documentation at `fossil-scm.org`.** Comprehensive, written by D. Richard Hipp. Includes the design rationale and the comparison-to-Git material.

**Hipp's writings on SQLite development and the role Fossil plays.** Available via the SQLite documentation and SQLite-adjacent talks.

## Chapter 18 — Darcs

**David Roundy's writings on patch theory.** The `darcs` documentation includes a *Theory of Patches* section that summarizes the original ideas. Roundy's original papers and posts are scattered but findable.

**Jason Dagit's PhD work on Darcs's merge problems and the move to Darcs 2** is the academic primary source for the exponential-merge issue and its resolution.

## Chapter 19 — Pijul

**Pierre-Étienne Meunier's papers, particularly on the categorical foundation.** The Pijul project's site (`pijul.org`) hosts the manual, papers, and links to the formal model.

**The Pijul Manual.** Aimed at users; covers both the model and the operations.

## Chapter 20 — Snapshots vs patches

**Roundy's *Theory of Patches* paper, Meunier's papers**, and the Sapling and Jujutsu design documents.

**Linus Torvalds's design notes for Git in early 2005.** The clearest articulation of why the snapshot model was chosen for the kernel's needs.

## Chapter 21 — Plastic SCM

**Pablo Santos's writings** at the Codice / Plastic / Unity blogs, archived under various names over the years. The Branch Explorer and branch-per-task workflow rationale is laid out there.

**Unity Version Control documentation** at Unity's site.

## Chapter 22 — Git LFS and git-annex

**The Git LFS specification** at `github.com/git-lfs`. The protocol is documented and stable.

**Joey Hess's writings on git-annex** at `git-annex.branchable.com`. Includes the design notes, the use cases, and the philosophical reasoning.

## Chapter 23 — DVC

**The DVC documentation at `dvc.org`.** Tutorials, model description, and integration guides.

**Iterative.ai's blog** for the broader ML-versioning context.

## Chapter 24 — Helix Core in art pipelines

**Perforce's "Helix Core for Game Development" guides.**

**GDC and SIGGRAPH talks on version control in games and film.** Talks by speakers from Naughty Dog, Insomniac, Ubisoft, Industrial Light and Magic, and Pixar at various years cover real-world pipeline configurations. Most are recorded; the GDC Vault is the central archive.

## Chapter 25 — Sapling

**The Sapling documentation at `sapling-scm.com` and the source repository at `github.com/facebook/sapling`.** The internal-design documents include the rationale for change-IDs, the Mononoke architecture, and the EdenFS design.

## Chapter 26 — Jujutsu

**The Jujutsu documentation at `jj-vcs.github.io`** (and the source repository at `github.com/jj-vcs/jj`).

**Martin von Zweigbergk's design documents and conference talks** on Jujutsu's model. The "first-class conflicts" and "operation log" rationales are well-explained.

**Steve Klabnik's tutorial *Jujutsu for Git users*** is widely cited as the easiest entry point for adopters coming from Git.

## Chapter 27 — Tool to platform

**The SourceForge Wikipedia entry, archived contemporaneous press, and the *Founders at Work* interview with Tom Preston-Werner** for early GitHub history.

**Microsoft's announcement of the GitHub acquisition** and Nat Friedman's subsequent posts on the platform's direction.

## Chapter 28 — GitHub

**GitHub's documentation at `docs.github.com`** is comprehensive on the platform's features.

**Nadia Eghbal's *Working in Public*** for the broader argument about open-source maintenance in the platform era.

**The GitHub Archive (`gharchive.org`)** for empirical data on platform activity over time.

## Chapter 29 — GitLab, Gitea, Sourcehut

**The GitLab Handbook** is unusually transparent about the company's internal workings; available at `about.gitlab.com/handbook/`.

**Drew DeVault's writings at `drewdevault.com`** for the Sourcehut philosophy.

**The Forgejo and Codeberg communities' documentation** for the community-governance perspective.

## Chapter 30 — Gerrit

**Gerrit's documentation at `gerritcodereview.com`.** Comprehensive on the model and the operational specifics.

**The Android Open Source Project's contribution guide** is a working example of Gerrit-driven open-source.

**Rietveld's archived documentation** for the history of code-review tooling.

## Chapter 31 — Platform-as-VCS

**Bradley Kuhn's writings on free-software infrastructure** at `sfconservancy.org`. Critical of the platform-absorption pattern from a free-software-freedom angle.

**Multiple essays on the *Octoverse* report and on the dependency-graph implications of GitHub's market share.** Annual reports from GitHub itself, plus independent analyses.

## Chapter 32 — Decision frameworks

**Atlassian's "Comparing Workflows" documentation** for some of the workflow patterns.

**The various "moving from X to Git" guides** that vendors and consultants have produced over the years; they implicitly encode decision frameworks.

## Chapter 33 — Anti-patterns and migration

**The `cvs2svn` / `cvs2git` documentation** at `cvs2svn.tigris.org` (now mirrored).

**The `git p4` man page and the `git svn` man page** for the canonical Git-side migration tools.

**`git filter-repo`** and the BFG Repo-Cleaner documentation for history rewriting.

**`gitleaks`, `truffleHog`, and `pre-commit`** documentation for secret-scanning and pre-commit hooks.

## Chapter 34 — What we lost

**This book.** Plus the further reading for each chapter; the inventory in Chapter 34 is a synthesis, and the original sources are where the ideas came from.

## Forums and communities

**The `git@vger.kernel.org` mailing list.** The Git project's primary development venue. Reading a few weeks of traffic is more illuminating than most secondary sources.

**The Mercurial mailing lists** at `selenic.com/mailman/listinfo`.

**The Sapling Discord, the Jujutsu Discord, and the Pijul forum** are where the active development of those systems is discussed.

**`git-scm.com/community`** lists current community resources.

## Closing note

This list is a starting point. The version control field rewards reading widely: a Mercurial user who has read the Git internals understands both better; a Git user who tries Pijul or Jujutsu for a week thinks about commits differently. The reading is not large in absolute terms; the entire core literature of the field, primary sources for every chapter in this book, is something a curious reader could work through in a few weekends of focused effort. The investment compounds: every system in this book has ideas worth carrying into whatever system the reader uses tomorrow.
