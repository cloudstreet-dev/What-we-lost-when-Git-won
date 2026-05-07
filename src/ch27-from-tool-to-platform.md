# 27. From Tool to Platform

This chapter is the hinge into Part VII. The version control systems of Parts II through VI are tools — programs that run on a developer's laptop, that talk to servers when they have to, and that leave most adjacent concerns (issue tracking, code review, release management, project pages, contributor onboarding) to whatever the team chooses. The systems of Part VII are *platforms*: hosted services that wrap a version control system and absorb the adjacent functions, sometimes more than the version control function itself. The shift from tool to platform is the largest cultural change in the field over the period this book covers, and it has had effects the field is still working through in 2026.

The chapter does not have a scenario walkthrough. It has a history, an argument, and a setup for the four chapters that follow.

## The pre-forge era

In 1998, a free-software project's infrastructure typically looked like this. *Source code* lived in CVS, on a server somewhere — often donated by a sympathetic university, sometimes hosted by Cygnus or VA Research, sometimes on a maintainer's home Linux box. *Bugs* lived in a separate database — the GNU Bug Tracking System, Bugzilla (created by Mozilla in 1998), or a homegrown PHP application. *Discussion* happened on mailing lists, with archives at `mail-archive.com` or the project's own server. *Releases* were tarballs, hosted on FTP, announced on a mailing list and possibly Freshmeat. *Documentation* was either in the source tree (`README`, `INSTALL`, `man/`) or on a separate web server, with PHP or Perl behind it if it was dynamic, plain HTML if it was not.

Each function had its own tool, often its own server, often its own administrator. A project's *infrastructure* was something maintainers had to assemble from parts, with the parts donated or paid for separately. Some projects had elaborate infrastructure (the Linux kernel had the lkml archive, ChangeLog, kernel.org, Bugzilla, and several other moving pieces); some projects had nothing but a tarball and a mailing list.

The arrangement had advantages. Each function could be the best-of-breed for its job. The project owned its data; if the bug tracker became unmaintainable, the project moved to a different one without losing the rest of the infrastructure. Tools were composable; new functions (a wiki, a continuous integration system, a vulnerability tracker) could be added without convincing a platform to support them.

The arrangement also had real costs. Setting up a project's infrastructure took weeks of a maintainer's time. Migrating an established project between tools was painful (CVS to Subversion took the open-source world years). The tools did not know about each other; cross-references between issues, commits, and patches were ad-hoc text in commit messages. Discoverability was poor; finding the right project required searching across mailing list archives, Freshmeat, and word of mouth.

## The proto-forge

*SourceForge*, launched in 1999 by VA Linux, was the first attempt to package the infrastructure as a service. A SourceForge project got CVS hosting (later Subversion), a Bugzilla-style tracker, mailing list management, file release downloads, a project home page, and a forum. SourceForge ran tens of thousands of projects at its peak. It worked because it absorbed the assembly cost: a maintainer who wanted version control plus issue tracking plus a mailing list got all three from one form.

SourceForge's later trajectory — sale, decline, advertising aggression, security incidents in the 2010s — is a separate story. What matters for this chapter is the model. SourceForge demonstrated that a hosted, integrated infrastructure was a market: free-software maintainers wanted what it provided, in volumes that justified the engineering. The model was attractive enough that imitators followed. *Savannah* (FSF), *Gna!*, *RubyForge*, *Gforge* (open-source SourceForge clone, eventually FusionForge), and several others appeared in the early 2000s.

*Trac*, written in 2003, took a different approach: a single tool that combined a wiki, a ticket tracker, a Subversion (later Git, Mercurial) browser, and a roadmap, packaged for self-hosting. Trac was genuinely useful and ran for years on many projects' infrastructure. The integration was richer than SourceForge's; the deployment was the project's responsibility.

*Redmine*, started in 2006, was the Ruby-on-Rails analog of Trac: integrated tracker, wiki, repository browser, deployable on a project's own infrastructure.

These tools showed that integration of version control with adjacent functions was both desirable and tractable. None of them, however, became the field's center of gravity. They remained one infrastructure choice among several.

## GitHub

*GitHub*, launched in April 2008 by Tom Preston-Werner, Chris Wanstrath, and PJ Hyett, did three things differently from SourceForge and Trac.

First, it built around Git. Git's distributed model meant a fork was cheap; GitHub made forking a single click. The act of forking became a *social* operation: you could see who had forked the project, you could see what they were doing in their fork, and you could send a *pull request* to the upstream maintainer asking for your changes to be integrated. The pull request — a piece of UI rather than a piece of Git — became the central object of GitHub's culture.

Second, it made the project's *page* the primary artifact. A GitHub project had a URL, a description, a `README` rendered as the page's main content, a list of issues, a list of pull requests, a list of contributors, an activity timeline, and an issue tracker, all on one URL. The cost of *finding* a project went to zero. The cost of *contributing* to a project went to a single click (fork) plus a handful more clicks (edit, commit, pull request).

Third, it monetized through private repositories. Free hosting for open-source projects was the marketing surface; paying users got private repositories for their commercial work. The model worked: by 2014, GitHub was profitable, growing fast, and the default for new open-source projects.

The field changed underneath the model. By 2012, "I have a project on GitHub" was the canonical form of "I have a project at all." By 2015, "fork me on GitHub" badges were on the homepages of language standards bodies, government agencies, and academic publications. Microsoft's $7.5 billion acquisition of GitHub in 2018 was the field's official acknowledgement that the platform had become infrastructure.

GitHub, in 2026, is the version control platform for the open-source world and a substantial fraction of the closed-source world. It has Issues, Discussions, Pull Requests, Actions (CI/CD), Pages (static hosting), Packages (artifact registry), Codespaces (cloud development environments), Copilot (AI assistance), Advanced Security, dependency management, and an API surface that lets a developer treat the platform as a programming target. Each of these is a function some VCS users used to handle outside the version control system. Most are, for most projects on GitHub, only available *inside* the platform.

## What the platform absorbs

The forge era's defining property is *absorption*. Functions that used to live outside the version control system have been absorbed into the platform: issue tracking, code review, releases, CI/CD, package distribution, security advisories, sponsorship, documentation hosting, discussion forums, AI-assisted code generation, supply-chain provenance.

For most users, most of the time, the absorption is a benefit. The integration is real; the cross-references work; the surface area is uniform. A developer who learns one workflow learns it for everything they do. The platform's investment in each function is far larger than any single project could afford.

For a smaller set of users, the absorption is a cost. The cost is felt by:

- *Maintainers of long-lived projects* who built infrastructure on the assumption of composable tools and now find their data partially in their version control system, partially in a hosted issue tracker, partially in a hosted CI/CD configuration, partially in a hosted discussion forum, with the platform owning the integration logic.
- *Projects that need to migrate* between platforms (GitHub to GitLab, GitLab to a self-hosted Forgejo, anywhere to anywhere). The git history travels; the issues, pull requests, CI configurations, and discussions usually do not, or migrate with substantial fidelity loss.
- *Projects in jurisdictions with platform restrictions*. GitHub has been blocked or restricted in several countries at various times. A project whose infrastructure depends on the platform inherits the platform's geopolitical exposure.
- *Projects that want functions the platform does not provide*. The platform's roadmap is the platform's; if a project needs a function the platform deems out of scope, the project has to layer something on or move.

The cost is also felt at the level of the *version control system*. Functions that GitHub absorbs are functions that the version control system itself used to evolve to provide. Mercurial's `hg serve` and `hg evolve`, Fossil's bundled bug tracker and forum, Subversion's `svnadmin` integration, Bazaar's Launchpad-tied workflow — all are evidence of version control systems trying to be platforms in their own right. They mostly lost. The function moved up the stack to the hosted forge, and the version control system stopped competing in those areas.

This is the structural shift the chapter is naming. Version control, as a tool category, has been hollowed out by the forge as a category. What remains in the version control system itself is the storage and the synchronization. Everything else has moved up.

## The four chapters that follow

Chapter 28 covers GitHub on its own terms: the pull request as a social object, the platform's affordances, the network effects, the costs to projects of being on GitHub.

Chapter 29 covers the alternatives: GitLab (which followed GitHub's model and succeeded as a self-hostable competitor), Gitea (a smaller, self-hostable forge), Forgejo (the community fork of Gitea), and Sourcehut (a deliberately minimal forge that takes the opposite stance). The chapter is about what platform diversity looks like in 2026.

Chapter 30 covers Gerrit and the *code review* lineage that runs alongside the forges: the position that code review is a serious enough function to deserve its own first-class system, separate from the platform.

Chapter 31 is the analytical chapter, parallel to Chapter 20 in its role: an argument that the platform absorbing the version control system has costs the field has not fully accounted for, and an inventory of what a project commits to when it commits to a platform.

Read Part VII as a continuation of the design space exploration. The forges are version control systems too, in the sense that what users *experience* as version control is increasingly the forge's UI. A book on the design space that ignored the forge era would be a book on something else.
