# Appendix B. Timeline (1972–present)

A condensed chronology of the systems and events covered in the book. Dates are approximate where the event spans a range; primary sources for any specific event are in Appendix D's references for the relevant chapter.

## 1970s — The first systems

**1972** — Marc Rochkind writes SCCS at Bell Labs. Initial implementation in SNOBOL4 on OS/MVT; the C version on Unix follows shortly. The first published version control system that worked.

**1975** — Rochkind publishes *The Source Code Control System* in *IEEE Transactions on Software Engineering*. SCCS becomes the documented model.

## 1980s — RCS, CVS

**1982** — Walter Tichy publishes the first version of RCS at Purdue. The 1982 ICSE paper *Design, Implementation, and Evaluation of a Revision Control System* describes it. Reverse deltas, free software licensing, GNU adoption.

**1986** — Dick Grune begins CVS as shell scripts wrapping RCS at the Vrije Universiteit Amsterdam.

**1989** — Brian Berliner rewrites CVS in C at Prisma. The C implementation becomes the version most users encounter.

**1989** — PVCS Version Manager begins shipping (Polytron Corporation, later Serena, later Micro Focus, later OpenText).

## 1990s — The corporate era

**1992** — Atria Software ships ClearCase, descendant of Apollo's DSEE. The MVFS, config specs, and per-element branches define the high-end enterprise position.

**1994** — Microsoft acquires One Tree Software; rebrands the product as Visual SourceSafe.

**1995** — Christopher Seiwald founds Perforce Software. Perforce 1.0 ships.

**1995** — Linux Standards Base discussions begin. Linux kernel development still patch-based.

**1998** — Bugzilla released (Mozilla). The bug tracker that defined a generation of bug trackers.

**1998** — Larry McVoy's *The SourceManagement Problem* circulates on linux-kernel. The thesis that becomes BitKeeper.

**1999** — VA Linux launches SourceForge. The first proto-forge, hosting CVS for thousands of open-source projects.

## Early 2000s — DVCS emerges

**2000** — BitKeeper 1.0 ships. The first commercial distributed version control system.

**2000** — CollabNet announces Subversion. Karl Fogel, Jim Blandy, and team begin development.

**2001** — Tom Lord begins GNU Arch (`tla`). The first serious free-software DVCS, ultimately undone by its UX.

**2002** — Linus Torvalds adopts BitKeeper for Linux kernel development. The decision is controversial; the workflow is productive.

**2002** — Trac released. The first widely-adopted integrated wiki/tracker/repository browser.

**2003** — David Roundy releases Darcs. Patch theory enters the field as a working system.

**2003** — Vault released by SourceGear. The "VSS done correctly" alternative.

**2003** — Graydon Hoare begins Monotone. Content-addressed storage and cryptographic identity, two years before Git.

**2004** — Subversion 1.0 released (February). The CVS replacement begins its rise.

**2005, April 3** — Larry McVoy revokes the BitKeeper free-OSS license following Andrew Tridgell's `sourcepuller` work. The kernel project loses its version control system.

**2005, April 7** — Linus Torvalds makes the first Git commit.

**2005, April 19** — Matt Mackall makes the first Mercurial release.

**2005** — Canonical begins Bazaar (originally as `baz`, GNU Arch reimplementation; rewritten in Python soon after).

**2007** — Plastic SCM 1.0 ships (Codice Software, Spain).

**2007** — D. Richard Hipp releases Fossil for SQLite development.

**2007** — Mozilla migrates `mozilla-central` from CVS to Mercurial.

## Late 2000s — GitHub and the rise of Git

**2008, April** — GitHub launches as a paid service. Tom Preston-Werner, Chris Wanstrath, PJ Hyett.

**2008** — Subversion 1.5 adds merge tracking via `svn:mergeinfo`.

**2008** — Bazaar 1.0. The flexibility-of-workflow position fully realized.

**2008** — Shawn Pearce begins Gerrit at Google for Android development.

**2009** — Subversion donated to Apache Software Foundation; becomes Apache Subversion.

**2009** — Mercurial 1.0 ships. The system's first major release.

**2010** — Joey Hess releases git-annex.

**2011** — Subversion 1.7 rewrites the working copy format, centralizing `.svn/`.

**2011** — Perforce introduces streams (first-class branching).

**2011** — Phabricator open-sourced by Facebook.

**2011** — Dmitriy Zaporozhets founds GitLab.

## Mid-2010s — Maturity

**2014** — Facebook adopts Mercurial for what becomes the largest Mercurial deployment in the world.

**2014** — Pierre-Étienne Meunier begins Pijul work.

**2015** — GitHub announces Git LFS.

**2016** — BitMover open-sources BitKeeper under Apache 2. Larry McVoy retires the commercial product.

**2016** — Gitea forked from Gogs.

**2016** — Git LFS adds locking protocol.

**2017** — DVC first release. Iterative.ai founded.

**2017** — Drew DeVault launches Sourcehut.

**2018** — Microsoft acquires GitHub for $7.5 billion.

**2019** — GitHub Actions launches (CI/CD).

## 2020s — Platform consolidation, new systems

**2020** — Bitbucket drops Mercurial support.

**2020** — Unity acquires Codice Software; Plastic SCM rebranded as Unity Version Control.

**2020** — Martin von Zweigbergk's Jujutsu becomes publicly visible at Google.

**2021** — Phabricator deprecated by Meta. Phorge community fork begins.

**2021** — GitLab IPOs on NASDAQ.

**2022, October** — Forgejo forks from Gitea.

**2022, November** — Meta open-sources Sapling.

**2023** — Mozilla announces migration from Mercurial to Git.

**2024** — Mozilla completes the Mercurial-to-Git migration. The largest open-source Mercurial deployment ends.

**2025** — Jujutsu adoption grows visibly outside Google. The system becomes the most actively recommended for new users curious about post-Git workflows.

**2026** — This book is written. Git is dominant; the design space remains open.

## Reading the timeline

Two patterns are visible.

*The DVCS revolution was a window.* Between 2005 (Git, Mercurial, BitKeeper revoked) and 2010 (Bazaar peak, GitHub established), the field went through a generational change in tooling. After 2010, the variety in active development narrows; the remaining systems are either entrenched (Perforce, ClearCase, Subversion in legacy use) or in dialogue with Git (Sapling, Jujutsu, Pijul, Fossil's small audience).

*Platform absorption began in 2008 and is ongoing.* GitHub launched in April 2008. The platform-as-VCS arrangement that Chapter 31 critiqued is roughly fifteen years old. The acceleration of features (Actions, Codespaces, Copilot) on the GitHub side, and the entrenchment of GitLab as the alternative, are the ongoing chapters of that absorption. The story is not finished.

The kernel/BitKeeper episode of April 2005, taking up a single row of the timeline, was the largest single event in version control history. Its effects are still being worked out twenty-one years later.
