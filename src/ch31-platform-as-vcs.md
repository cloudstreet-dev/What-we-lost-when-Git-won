# 31. What Gets Lost When the Platform Becomes the Version Control

This chapter is the parallel of Chapter 20: an analytical chapter that names something the field has accepted without having argued for it. Chapter 20 argued that the snapshot model was a choice, not a discovery. This chapter argues that the platform model — the GitHub-style fusion of version control, issue tracking, code review, CI/CD, package distribution, and project pages into one hosted service — has costs the field has not fully accounted for, and that the costs are larger than they appear because the platform now defines what "version control" means in everyday usage.

The chapter is not an argument for leaving GitHub. It is an inventory of what the platform absorbed, what its absorption has displaced, and what a serious team should think about when choosing how to host its work.

## What "version control" means in 2026

Ask a working programmer in 2026 what version control is. The answer, in most cases, will mix Git operations with GitHub operations indistinguishably. *I open a pull request*. *I review the diff and approve*. *I merge it*. *I check the CI status*. *I tag a release*. *I close the issue*. Each of those is partly Git, partly platform; the platform parts are inseparable from the rest of the workflow in the speaker's mental model.

This is the absorption Chapter 27 named, made concrete. The platform has expanded "version control" colloquially to mean "what we do with our code, end to end." When the field talks about "moving from CVS to Git," it usually means "moving from CVS-plus-Bugzilla-plus-mailing-list-plus-FTP-releases to Git-plus-GitHub." The version control change is one piece; the rest is a change of platform, of workflow, of culture.

The merger has costs. The chapter inventories them.

## Loss 1: Composability

In the pre-platform era, a project could mix and match tools. A project might use Subversion for version control, Bugzilla for issues, Mailman for discussion, Trac for the wiki and the project home page, custom scripts for releases. Each tool was independent; replacing one did not require replacing the others; the project's data lived in the project's chosen mix.

In the platform era, the tools are bundled. The project's issues are GitHub Issues; the project's discussion is GitHub Discussions; the project's wiki is GitHub Wiki; the project's CI is GitHub Actions; the project's releases are GitHub Releases. Each of these is a function the project once chose independently; on the platform, they come as a set.

The bundle has benefits — integration, discovery, ease of adoption. It has costs the field underweights. *A project on the platform cannot easily replace one tool without replacing them all.* If GitHub Actions becomes too expensive, moving to another CI system means decoupling the CI from issues, releases, and pull requests, where the integrations were free. If GitHub Issues' data model does not fit the project's needs, switching to a separate tracker means losing the cross-references the platform provides.

This is not a hypothetical. Many projects in 2026 use external tools (Linear, Jira, ClickUp) for issue tracking *despite* the integration with GitHub Issues, because the external tool fits their workflow better. The cost they pay is that GitHub PRs that reference Linear issues do not auto-close; the cross-references require bots; the integration is glued together rather than built in. The composability is recoverable, at a cost. The cost is the price of escape.

## Loss 2: Project ownership of project data

A project's *data* — its issues, its review comments, its CI configurations, its releases — is part of the project. In the pre-platform era, the project owned this data: it lived on the project's servers, in the project's chosen formats, exportable on the project's terms.

In the platform era, the data lives on the platform's servers, in the platform's formats, exportable on the platform's terms. The Git history is portable; everything else is partially portable at best. GitHub provides export tools, but the export is approximate: GitHub Issues exported as a JSON dump are not the same as GitHub Issues running natively on a different platform; the references between issues, PRs, commits, and releases get mangled in the transfer.

This is felt sharply by projects that try to migrate. Migrating a Git history is straightforward; migrating ten years of issue history with intact cross-references is a months-long engineering project. Many projects that consider leaving GitHub conclude that the migration cost is too high and stay; the staying is, in part, an artifact of the data ownership problem.

Self-hosting helps somewhat. A team running Gitea or Forgejo or self-hosted GitLab owns the data in a way that a team on GitHub.com does not. Migration *between* self-hosted instances is also more tractable. The cost of self-hosting is operational; the benefit is sovereignty.

## Loss 3: The pull request narrowing what review can be

Chapter 30 covered Gerrit, Phorge, and the code review lineage: tools that take review seriously as a discipline, with structured comments, multi-patch-set revisions, vote semantics, and resolution tracking.

The pull request, as it exists on GitHub and its imitators, is a flattening. The "approve / request changes / comment" trichotomy collapses Gerrit's vote levels. The single-thread-of-work model loses the patch-set structure of revisions in response to review. The "merge" button at the bottom of the PR encourages the reviewer to be the merger, eliding the separation Gerrit enforces between review and submit.

Most teams in 2026 that take review seriously have learned to live with the pull request's limitations. Some have built layered tooling — Reviewable, Graphite, ReviewStack — that puts richer UX on top of GitHub's PR substrate. The work is real and useful. It is also a workaround for a primitive that, by being the dominant convention, has set the upper bound on what review tooling can naturally be.

The cost is one of *imagination*: programmers who have only used pull requests have a hard time articulating what they would want from review tooling, because the pull request is the shape they think review takes. Engineers who have used Gerrit or Phabricator can articulate the gap; those who have not, often cannot.

## Loss 4: Mailing lists as a collaboration substrate

The mailing list — a public archive of threaded email — is a collaboration substrate that predates everything else in this book. The Linux kernel is developed on lkml; PostgreSQL, on its own lists; OpenBSD, FreeBSD, NetBSD, on theirs; the Python language standard, partly on python-dev. The list is the shared memory of the project, the thing new contributors read to understand the project's history, the place where decisions are made and recorded.

The platform's discussions feature attempts to replace this. GitHub Discussions, GitLab Discussions, Sourcehut's mailing-list service. Of these, only Sourcehut's is genuinely a mailing list (the others are forum-style threads). The forum-style replacement loses the property that matters most: *email*. Mailing lists work because every message arrives in the participant's inbox; every participant can see every message immediately, in their existing email client, integrated with their existing workflow. Forums require the participant to visit. Forums are *places to go*; mailing lists are *things that arrive*.

The cultural cost: the kernel's development pace, the BSD projects' deliberation, the Python community's PEP discussions, the W3C's working-group memos — these all function in part because of the asynchronous, archival, broadcast properties of the mailing list. New contributors who have only used GitHub Discussions or Discord do not have the model in their hands. They are not less capable; they have been trained on a different shape of asynchronous communication, and the difference shows up in unexpected ways.

## Loss 5: The patch as a portable artifact

A patch — the output of `git format-patch` or `diff -u`, with its `--- a/file` and `+++ b/file` headers and its hunks — is a portable artifact. It can be emailed; it can be saved to a tarball; it can be applied to any Git, Mercurial, or even some non-DVCS repositories with appropriate tools; it can be archived for permanence.

The pull request is a *thing on GitHub*. It has a URL on GitHub. It has comments on GitHub. Its review history lives on GitHub. To preserve a pull request, you preserve a snapshot of the GitHub UI; to apply the work in a non-GitHub context, you extract the patches from the PR and use them as patches.

In an important sense, the pull request is *not* a portable artifact. The Git side of it (the commits) is portable; the rest is not. Projects that build cross-project tooling — bots that propagate fixes across forks, scripts that mirror PRs across forges — are doing the work to make the platform-bound artifact portable, against the grain of the platform.

The kernel's email-patch workflow, by contrast, has been portable for thirty years. A patch sent to lkml in 2005 can still be applied to today's kernel (modulo conflicts, but the artifact is still a patch). A pull request from 2015 lives at a GitHub URL that may or may not exist next year.

## Loss 6: The CI configuration as a project artifact

CI/CD configurations in 2026 are typically YAML files (`.github/workflows/*.yml` for GitHub Actions, `.gitlab-ci.yml` for GitLab CI). On the surface, the configuration is portable: the YAML is in the repository, version-controlled, transferable.

In practice, the configurations are *platform-specific*. GitHub Actions YAML uses GitHub's `actions/*` library; GitLab CI YAML uses GitLab's runner concepts. Migrating CI between platforms means rewriting the configuration. Multi-platform CI requires duplicating the configuration; tools like `act` (which runs GitHub Actions locally) and `dagger` (which abstracts CI across platforms) help, but the underlying portability gap is real.

The pre-platform CI tools (Jenkins, Travis CI, CircleCI before Travis declined) had their own portability problems but were *separate from the version control system*. A team could change CI tools without changing where their code lived. The platform's CI is bound to the platform; a CI change is a platform change is a hosting change.

## Loss 7: The project page as a separable thing

A project's *home page* — the URL where humans read about the project, find the documentation, and get to the source — used to be a separate concern from the version control. The page might be on the project's own domain, on a personal home page, on SourceForge, on Savannah; the source might be elsewhere.

In the platform era, the project page is *the platform's project page*. `https://github.com/owner/repo` is the de facto homepage. The README is rendered there; the issue tracker is linked there; the releases are listed there; the discussion is linked there. A project's identity becomes the URL.

This is convenient. It is also a binding. The project's URL changes when the project moves. The project's link economy — the inbound links accumulated over years from blog posts, Stack Overflow answers, books, documentation — is bound to the platform URL. Moving means rewriting the inbound links; old links break or need redirects.

A project that maintains its own home page (linking to wherever the source happens to be hosted) preserves the link economy. Most projects in 2026 do not. The cost of moving is, in part, the cost of the broken inbound links the move would create.

## Loss 8: Decentralization as a social property

Git is structurally decentralized: every clone is a full repository, peer-to-peer synchronization is a normal operation, the system has no required central authority. Distributed version control is, in its bones, a tool for coordination without a central server.

The platform-as-VCS makes the central server *required again*, socially if not technically. A project's identity is its GitHub URL; its issues are on GitHub; its CI runs on GitHub; its releases are on GitHub. The Git substrate would let the project move; the social substrate makes moving expensive.

The structural decentralization that DVCS provides is recoverable: a project can mirror to multiple platforms, can self-host alongside platform hosting, can publish bundles for offline access. Most projects do not. The decentralization is, in 2026, more of an option than a default.

## What the platform got *right* that should not be lost

The chapter is not an argument that the pre-platform era was better in every way. Several things the platform delivers genuinely improved the field, and any honest accounting has to keep them:

*Discoverability.* Finding a project, finding a maintainer, finding contributors — these are tractable in a way they were not in the SourceForge era. The platform's investment in search, in profile pages, in cross-references is real and valuable.

*Onboarding friction.* New contributors can submit their first contribution to a project in five minutes on the platform. In the pre-platform era, the same operation took an hour of researching the project's specific submission process. The reduction is a benefit.

*Integration.* The cross-references between issues, PRs, commits, and releases work. The integration is real; the value is real; the convenience compounds.

*Investment.* The platform's engineering team is large and competent. Functions are maintained, security issues are addressed, the API is stable. A project on the platform inherits the platform's maintenance for free.

These are real benefits. A serious analysis cannot ignore them. The chapter's claim is the milder one: the benefits come with costs the field underweights, and the costs are real enough to deserve naming.

## What this means for choosing

Choosing to be on a platform is choosing more than a Git host. It is choosing:

- An issue tracker with that platform's data model.
- A code review tool with that platform's UI.
- A CI/CD system with that platform's runners and YAML.
- A discussion forum with that platform's affordances.
- A release page with that platform's URL structure.
- A community discovery surface with that platform's network effects.
- A contributor onboarding pattern with that platform's friction profile.
- A relationship to that platform's commercial trajectory and ownership.

Each of these is a position. Most projects in 2026 take all of GitHub's positions by default; the position that is the result is rarely articulated. The chapter's hope is that articulating it makes the choice more visible.

Some teams will, after articulating the choice, decide GitHub is right for them. Many will. The choice having been made consciously is the point. The pre-articulation default — GitHub because GitHub is what version control is — is the part the chapter argues against.

## What the platform-era inventory looks like

To summarize the losses, with each one's recovery option:

| Loss | Recovery |
|---|---|
| Composability | Self-host, or mix platform + external tools |
| Data ownership | Self-host, or maintain regular exports |
| Review beyond pull requests | Use Gerrit, Phorge, or layered tools (Graphite, etc.) |
| Mailing-list collaboration | Sourcehut or self-hosted lists alongside the platform |
| Portable patches | `git format-patch` workflow continues to work |
| Portable CI | Self-host CI, or use multi-platform abstraction tools |
| Project page sovereignty | Maintain a separate project home page |
| Decentralization | Mirror to multiple hosts, self-host alongside |

Most projects will take some of these recoveries and not others. The recoveries are not free; each adds operational complexity. Whether the complexity is worth the recovery is the team's call, made with full awareness of what is being recovered.

## Epitaph

The platform absorbed version control whole, and the field has been doing version control through the platform's hands for fifteen years — long enough to forget that there was a difference, and long enough that the difference has cost things the field has not yet finished paying for.
