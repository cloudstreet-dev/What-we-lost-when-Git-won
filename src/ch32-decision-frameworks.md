# 32. Decision Frameworks

This chapter is a tool, not an argument. It is a structured way to choose a version control system from the design space the book has covered. The chapter assumes a team facing a real choice — a new project, a migration, a green-field decision — and walks through the questions that should determine the answer.

The framework has six axes, each producing a position. Most teams will land on one of three or four common combinations, with Git plus a forge being one of them. The point of the framework is to make the choice visible: a team should know which of their constraints pushed them toward Git, which toward Perforce, which toward Sapling, and which toward something else.

## Axis 1: Dominant content

The first question. *What is the team versioning?*

**Predominantly text** (source code, configuration, documentation, prose). Text-only projects fit the snapshot or patch models well; merging is tractable; storage is small; the design space is wide.

**Predominantly binary** (game art, film assets, schematics, PSDs, FBX, 3D models, audio, video). Binary content does not merge; it does not delta-compress; storing every revision is expensive. Locking is essential; integration with binary tools matters; the design space narrows to systems that handle this case natively.

**Mixed** (most game projects, many design-and-code projects, ML workflows). Mixed content is the hardest case. Pure text systems handle the binary side awkwardly; pure binary systems handle the text side without enthusiasm. The systems that handle both — Plastic SCM, Perforce, Helix4Git bridges — are the candidates.

**Data-and-code** (ML projects, data-science projects, projects where the artifact is a function of code plus data). The design space includes DVC, lakeFS, MLflow's lineage tools. Pure version control systems alone are not the right answer; the question is which augmentation fits.

## Axis 2: Team shape

*Who is contributing, and what tools do they use?*

**Developers only.** Programmers using IDEs and CLIs. Standard Git and the platform model fit cleanly.

**Developers plus non-developers.** Programmers, designers, technical writers, possibly product managers. Some contributors will not learn `git rebase -i`. Plastic SCM's Gluon, Subversion's simplicity, or platform UIs (GitHub web editor, GitLab Web IDE) come into consideration.

**Artists (DCC tool users).** 3D modelers, texture artists, animators using Maya, ZBrush, Substance, Houdini. Perforce or Plastic with the tool's own integration is the answer; Git LFS is the layered alternative if Git is mandated.

**Regulated industry contributors.** People who need audit trails, sign-offs, traceability between requirements and code changes. PTC Codebeamer, GitLab Premium with audit features, Helix Core in compliance configurations.

**Globally distributed contributors with email-only collaboration.** A project where some contributors cannot or will not use a platform's web UI for review. Sourcehut's email-patch workflow, or Git plus a mailing list with `git send-email`.

## Axis 3: Workflow needs

*How does the team get changes from idea to integrated?*

**Pull-request workflow** (the GitHub default). Branches, pull requests, line comments, approve-and-merge. Best served by GitHub, GitLab, Gitea / Forgejo, or self-hosted Git plus a forge.

**Stacked-diff workflow.** Sequences of small commits, each reviewed independently, all updated together when the bottom changes. Best served by Sapling, Jujutsu, Phorge, or layered tools (Graphite, ReviewStack).

**Email-patch workflow.** Patches sent by mail, reviewed in mail, applied with `git am`. Best served by Sourcehut, plain Git plus a mailing list, kernel-style infrastructure.

**Heavyweight code review** (every change reviewed, multi-step approval, audit). Best served by Gerrit, Phorge, Helix Swarm, GitLab Premium, GitHub with strict branch protection.

**Trunk-based development with continuous integration.** Frequent small commits to a single main branch, with feature flags for in-flight work. Best served by any system that supports fast operations on a shared main branch — Git with a fast forge, Perforce with continuous integration to mainline.

**Long-lived release branches with backports.** Several major versions in maintenance, fixes flowing between them. Best served by systems with strong cherry-pick and merge-tracking — Subversion (mergeinfo), Mercurial with bookmarks, Perforce streams, Git with disciplined rebase practice.

## Axis 4: Scale

*How big is this going to get?*

**Small** (one to ten contributors, one repository, code-only). Almost any system works. The dominant cost is the team's existing knowledge; pick what they know or what is easiest to learn.

**Medium** (tens of contributors, dozens of repositories, mostly code). Git plus a forge is the dominant choice. Perforce is overkill unless there are binaries; Subversion is fine but losing maintenance.

**Large** (hundreds of contributors, hundreds of repositories or one large monorepo, mixed content). The trade-offs become visible. Git monorepos hit performance walls; Perforce monorepos need careful tuning; Sapling and Jujutsu are designed for this scale; Plastic SCM scales well; ClearCase scaled but is in legacy.

**Very large** (Google or Meta scale: monorepos with millions of commits, hundreds of thousands of files, terabytes of source). The market is small; the answers are specialized: Sapling against Mononoke (Meta's path), Piper or Mononoke-fronted Git (Google's path), bespoke systems built on top of Git or Mercurial. Most teams should not need this category.

## Axis 5: Deployment

*Where does the version control system live?*

**Cloud-hosted by a vendor.** GitHub.com, GitLab.com, Bitbucket, Atlassian Cloud, Azure DevOps Cloud. Lowest operational cost; highest dependency on the vendor.

**Self-hosted on the team's infrastructure.** GitHub Enterprise Server, GitLab CE/EE, Gitea, Forgejo, Sourcehut, self-hosted Perforce, self-hosted Plastic. Higher operational cost; full control over data and policy.

**Hybrid** (production code in cloud, classified or sensitive code self-hosted). Common in regulated industries. Requires the team to operate two systems, with discipline about what goes where.

**Air-gapped.** Government, defense, classified work. Requires fully self-contained systems; cloud platforms are not options. Self-hosted Gitea, GitLab, Subversion, Perforce, Fossil.

## Axis 6: Sovereignty

*How much does the team need to own its data and workflow?*

**Platform-OK.** The team is comfortable with the platform owning the issues, the CI configuration, the release pages, the discussions. The platform's terms are acceptable; the platform's roadmap is acceptable; lock-in is not a primary concern. GitHub or GitLab.com fit.

**Mixed.** The team wants to be on a platform but wants the option to leave. Self-hosted Gitea / Forgejo, self-hosted GitLab CE, with regular exports of issues and CI configuration. Or platform plus duplicate hosting (mirror to a self-hosted instance).

**Self-hosting required.** The team's policy or compliance regime requires the data to live on the team's servers. Self-hosted everything: Git server, issue tracker, CI, release infrastructure.

**Workflow sovereignty.** The team rejects platform-prescribed workflows and wants control over how review, contribution, and integration happen. Sourcehut, plain Git plus mailing lists, Gerrit deployed independently.

## Common combinations

Most teams land in one of these combinations.

**The default (text + developers + PR workflow + medium scale + cloud + platform-OK).** Git on GitHub, GitLab.com, or Bitbucket Cloud. The vast majority of new projects in 2026 fit here, and the choice is well-served.

**The self-hosted developer team (text + developers + PR workflow + medium scale + self-hosted + sovereignty).** Self-hosted Gitea, Forgejo, or GitLab CE. The infrastructure cost is moderate; the sovereignty is real.

**The game studio (mixed + artists + branch-per-task + large + self-hosted + air-gapped-friendly).** Perforce / Helix Core, with Helix Swarm for review and DCC tool integration. Plastic SCM (Unity Version Control) is the alternative, especially for Unity-engine projects.

**The regulated enterprise (text + developers + heavyweight review + large + self-hosted + sovereignty + audit).** GitLab Premium, Gerrit, or in legacy cases ClearCase. PTC Codebeamer for the integrated requirements-and-code use case.

**The kernel-style project (text + developers + email-patch + medium + plain-Git + maximum sovereignty).** Plain Git on a project-controlled server, with a mailing list, with `git format-patch` and `git send-email`. The Linux kernel, FreeBSD, OpenBSD operate this way.

**The ML / data-science team (data-and-code + developers + experiment-tracking + medium + cloud + platform-OK).** Git plus DVC, on GitHub or GitLab, with the DVC remote on cloud storage. Possibly augmented with a hosted experiment tracker (Iterative Studio, MLflow, Wandb).

**The Sapling / Jujutsu adopter (text + developers + stacked-diff + medium-to-large + cloud-or-self-hosted + sovereignty-flexible).** Sapling against Git or Mononoke; Jujutsu against Git on any host. The investment is in the workflow tooling; the substrate stays Git for compatibility.

**The Fossil project (text + small team + integrated-everything + small + self-hosted + maximum simplicity).** Fossil. Single binary, single SQLite file, code-plus-bugs-plus-wiki-plus-forum in one artifact.

## Anti-patterns

Some combinations look reasonable on paper and fail in practice.

**Git for an art-asset-heavy game.** Git + LFS works for projects with modest binary content; for AAA-game-scale art pipelines, the LFS storage and bandwidth costs grow unmanageable, and the locking is advisory rather than enforced. The team that tried this in 2023 is on Perforce in 2025.

**Self-hosted GitLab for a five-person startup.** GitLab CE is a heavy stack: Postgres, Redis, Sidekiq, GitLab itself, container registry. For a small team, the operational overhead exceeds the value over GitHub.com.

**Sapling against a Git host without Mononoke.** Sapling can operate against any Git remote, but several of its distinctive features (server-side change-ID tracking, restacking semantics through pull) require Mononoke. Teams that adopt Sapling against GitHub get a partial experience; they should know what they are giving up.

**Perforce for a small open-source project.** Perforce's licensing for small open-source teams is favorable, but the operational and tooling overhead — the workspace model, the typemap, the CI integration that assumes Git — is enough that the project's contributor friction grows. Most small open-source projects fit Git better.

**Trying to use Subversion in 2026 for a new project.** Subversion is alive and works; the ecosystem is in retreat. Tooling, hosted forges, IDE integration, and contributor familiarity all favor Git. New projects should choose Git unless there is a specific reason — heavy binary content with a free-software preference, or interop with an existing Subversion deployment — that points elsewhere.

**Unconsidered defaults.** A team that picks "Git on GitHub" without thinking through the six axes will probably end up with a workable choice; they may also end up with a choice that does not fit their actual constraints. The framework's value is in the *thinking through*, not in the answer's surprise.

## How to use the framework

For a real decision, work through the axes in order:

1. List the dominant content. Note any non-text content and its volume.
2. List the team shape. Note any non-developer contributors and their tools.
3. List the workflow needs. Note any specific patterns (stacked diffs, email patches, heavyweight review).
4. Estimate scale, present and projected.
5. Determine deployment constraints.
6. Determine sovereignty needs.

Then map the combination to systems. Most combinations will narrow to two or three candidates; among them, the team's existing knowledge, hiring profile, and operational capacity decide the rest.

The framework does not eliminate disagreement; reasonable teams will weight axes differently and reach different conclusions. The framework's contribution is making the disagreement *about the weights*, not about the existence of the alternatives.

## A note on changing systems

Migrations are expensive. The framework's recommendation, even when it suggests a system the team is not currently on, is to consider the migration cost in the cost column. A team on a workable-but-suboptimal system often does better staying than migrating, especially if the migration disrupts ongoing work.

Migrations that are unambiguously good in 2026 include:

- Subversion to Git, for projects whose ecosystem and contributors have moved to Git.
- ClearCase to Git, for projects whose configuration management requirements are not what they once were.
- VSS to anything, on any timeline.
- Bazaar to Git or Breezy, for projects where Bazaar is on borrowed time.

Migrations that are often *not* worth it:

- Perforce to Git, for projects with substantial binary content.
- Mercurial to Git, for projects on Mercurial that are working fine, especially Meta-shaped projects that benefit from staying close to Sapling's lineage.
- Plastic SCM to Git, for game projects on Unity.

The migration cost is paid in engineering time, in workflow disruption, in retraining, and in the loss of platform-bound data (issues, CI, releases) when the platform changes too. The benefit is the new system's better fit for the constraints. Both sides should be sized honestly; the framework's role is to inform the sizing.

## Epitaph for the framework

The version control choice is a decision the field has, in 2026, treated as already made. The framework's value is reopening the decision long enough for a team to make it deliberately, with awareness of the design space and the costs of the defaults.
