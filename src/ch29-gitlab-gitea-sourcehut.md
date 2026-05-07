# 29. GitLab, Gitea, Sourcehut

The forge market in 2026 is concentrated but not monopolized. GitHub holds the largest share by every measure, but several real alternatives serve real audiences with real differences in stance. This chapter covers the three most consequential: GitLab (the GitHub-shaped competitor with strong self-hosting), Gitea and its fork Forgejo (the lightweight self-hostable forge), and Sourcehut (the deliberately minimal anti-platform). Each represents a position about what a forge should and should not be.

## GitLab

GitLab was founded in 2011 by Dmitriy Zaporozhets, a Ukrainian developer who began the project after using GitHub privately and wanting a self-hostable alternative. Sid Sijbrandij joined as co-founder and CEO; the company incorporated in 2014 in San Francisco and grew through several venture rounds. GitLab went public on NASDAQ in October 2021. The company is fully remote and has been since founding, with employees in roughly seventy countries.

The product is open-core. *GitLab Community Edition* is MIT-licensed, free to download, free to self-host, with a generous feature set that covers Git hosting, issue tracking, CI/CD pipelines, container registry, package registry, wiki, snippets, and user management. *GitLab Enterprise Edition* is the paid tier, with additional features (advanced authentication, audit logs, security scanning, geo-replication, role-based access controls) layered on. The two share a common code base; the Enterprise features are gated by license check.

GitLab's positioning has been *DevOps platform*: the pitch is that an organization can run its entire software development lifecycle on GitLab — code, issues, CI, deploy, monitor — without integrating multiple tools. The pitch resonates with regulated industries (finance, healthcare, defense), self-hosting shops (where GitHub Enterprise is harder to deploy than a self-hosted GitLab), and large companies that want vendor independence.

The CI/CD layer is GitLab's strongest distinguishing feature. GitLab CI was the first CI/CD system tightly integrated into a forge, predating GitHub Actions by several years. The YAML pipeline language is mature, the runner system is flexible, and the integration with merge requests, deploys, and environments is deep. Many teams that use GitHub for hosting nevertheless use GitLab for CI through hybrid setups.

The pull-request analog is the *merge request*. The mechanics are similar to GitHub's: branch, propose, review, merge. The semantic differences are mostly cosmetic; the workflow the user lives in is recognizable from GitHub.

The exemplar plays out on GitLab as a series of operations against a project page that mirrors GitHub's. The merge request UI is the focal point for collaboration. CI runs from `.gitlab-ci.yml` at the repository root.

In 2026, GitLab is the most credible alternative to GitHub for organizations that want a comparable feature surface with self-hosting available. The cultural footprint is smaller; the technical footprint is mature. The two systems compete in the enterprise market on price, features, and policy.

## Gitea and Forgejo

*Gitea* was forked from *Gogs* in 2016 by community members who felt Gogs's development was too tightly controlled by its original author. Gitea is written in Go, lightweight (a single binary, runnable on a Raspberry Pi), and self-contained. The user surface is modeled on GitHub's, with the major features (repositories, issues, pull requests, wikis, releases) covered.

In 2022, the Gitea project was incorporated as Gitea Ltd., with a commercial entity controlling the trademark and a portion of the development. Community members concerned about the corporate direction forked the project in October 2022 as *Forgejo*, governed by Codeberg e.V., a German non-profit. Forgejo and Gitea remain technically similar but socially distinct: Forgejo is community-governed, Gitea is corporate-led.

Both projects are alive in 2026. Gitea continues with its commercial backing and a steady release cadence. Forgejo has its own release cadence, with Codeberg as the flagship hosted instance. The technical differences are small enough that for most use cases the two are interchangeable; the social differences matter to projects that care about governance.

*Codeberg* — `codeberg.org` — is the visible Forgejo instance. It hosts a substantial number of free-software projects, particularly European projects and projects that want to host outside the Microsoft / GitHub ecosystem. Codeberg's resources are modest compared to GitHub's, but the service is real, free, and growing.

Gitea / Forgejo's positioning is *the lightweight self-hostable forge*. A small team or an individual can run an instance on a $5/month VPS; an organization can run an instance on a single mid-sized server for tens of thousands of repositories. The administrative model is simple; backups are file copies; migration is a directory move. The contrast with GitLab — which is heavyweight to deploy, with a Ruby on Rails core, a PostgreSQL database, a Redis cache, a Sidekiq job runner, an Elasticsearch service for search, and a Container Registry — is the point.

The exemplar plays out on Gitea / Forgejo as a stripped-down GitHub: pull requests, issues, releases, basic Actions-compatible CI (Gitea Actions, with a different runner but similar YAML), wiki, packages. Some advanced functionality (advanced security scanning, large-team SAML integration, complex CI workflows) is not present or is rougher than on GitHub.

For projects that want to be *outside* the GitHub orbit but inside something familiar, Gitea / Forgejo is the dominant answer.

## Sourcehut

Sourcehut — `sr.ht`, pronounced "sir hat" — was started by Drew DeVault in 2017. The project is a deliberate inversion of the GitHub model. Where GitHub is a single integrated platform with a tight feature loop, Sourcehut is a federation of *small, independent services* that each do one thing: `git.sr.ht` for repositories, `lists.sr.ht` for mailing lists, `todo.sr.ht` for tickets, `builds.sr.ht` for CI, `paste.sr.ht` for pastes, `man.sr.ht` for documentation, `pages.sr.ht` for static hosting. Each service has its own subdomain, its own UI, its own API; they are interoperable but not unified.

The contribution workflow is *patches by email*. A contributor runs `git format-patch` and `git send-email` to submit a patch to the project's mailing list. The maintainer reviews, possibly comments by replying, and applies the patch with `git am`. There is no "fork" button; there is no "open pull request" button; there is no platform-side branch on the project's repository for the contributor's work.

The model is genuinely different from GitHub's, and DeVault has been articulate about why: the email-patch workflow predates the pull-request workflow by decades, scales differently (mailing lists with thousands of subscribers handle review without UI infrastructure), composes with non-Sourcehut tools (any email client, any project not on Sourcehut), and avoids the lock-in concerns the platform model creates. The position has critics; it has also retained a small but committed user base that includes high-profile maintainers (the Wayland project, several BSD-adjacent projects, much of the suckless community, parts of the SourceHut software stack itself).

Sourcehut's commercial model is also distinctive: there is no free tier for hosted accounts. To use `sr.ht`, you pay (currently somewhere around $20–$50/year for a personal account; commercial pricing for companies). The position is that hosting is not free, that ad-supported and venture-backed models compromise users, and that paying directly aligns the user's interests with the platform's.

Sourcehut also offers all of its software open-source under a permissive license; self-hosting is supported and documented. Several teams and organizations run their own Sourcehut instances.

The exemplar on Sourcehut plays out very differently. *Operation 1 (initial import)* is `git push` to `git@git.sr.ht:~aditi/ledger`. *Operation 2 (linear development)* is plain Git. *Operation 3 (branch)* is plain Git; there is no platform-side affordance for "this is a branch waiting for review." When Jonas wants Aditi to integrate his work, he runs:

```
$ git format-patch -3
$ git send-email --to=~aditi/ledger-devel@lists.sr.ht 0001*.patch 0002*.patch 0003*.patch
```

The patches arrive at the mailing list. Aditi (and any subscribers) read them. Discussion happens on the list. When Aditi is ready, she applies the patches:

```
$ git am < /path/to/saved/patches.mbox
```

Or pipes the mailing-list archive's patch URL directly through `b4` (a tool that simplifies the workflow). The result is committed to the upstream branch; the contribution has flowed through email rather than through platform UI.

*Operation 7 (merge with conflict)* is plain Git. *Operation 9 (tag)* is plain Git. *Operation 10 (release)* is `git push --tags`; the tag appears on `git.sr.ht`'s release page; a release announcement is, conventionally, an email to the project's mailing list with release notes and a link to the artifact.

The Sourcehut workflow is what kernel development has used for thirty years. It is also what most working programmers in 2026 have never tried.

## What the alternatives represent

Three positions on what a forge should be:

*GitLab*: a GitHub-shaped competitor with self-hosting and a strong DevOps story. The position is that the platform model is correct and that the right competitive response is to do the platform model better in some respects (CI, self-hosting, pricing transparency). GitLab's success is evidence that there is a market for this position.

*Gitea / Forgejo*: a lightweight self-hostable forge that does the basics well. The position is that most teams do not need the full GitHub feature surface and that a small, simple, self-contained tool is the right answer for many use cases. The success of Codeberg and the proliferation of self-hosted Gitea instances is evidence that this position has a real audience.

*Sourcehut*: a deliberately-minimal anti-platform that bets on workflow rather than features. The position is that the platform model has costs and that the email-patch workflow, with small focused services, is closer to what version control has historically been and should remain. The audience is small but committed; the existence of the position is itself the contribution.

There is also a fourth position the chapter has not covered: the *self-hosted Git server with no forge UI at all*. `git` runs over SSH to a bare repository on any server with the `git` command installed; that is the minimal viable hosting. Several large projects run essentially this way (the Linux kernel itself, hosted on `git.kernel.org`, with `cgit` as the read-only web UI and `lkml` as the contribution channel), and the model deserves recognition: it is the model the rest of the platform discussion is layering on top of.

## Choosing among them

Choose GitLab when:

- You need self-hosting with a full feature surface comparable to GitHub Enterprise.
- The team's CI/CD requirements are heavy and GitLab CI's maturity is the deciding factor.
- The organization has DevOps integration concerns that GitLab's "everything in one app" message resonates with.

Choose Gitea / Forgejo when:

- You want self-hosting with a small operational footprint.
- The project's needs are core forge features (repos, issues, PRs, basic CI) and not the full GitHub surface.
- The team values community governance (Forgejo) over corporate roadmap stability (Gitea).

Choose Sourcehut when:

- The project's contributors prefer the mailing-list workflow.
- You want to be deliberately outside the platform model.
- The team's collaborators include people who already use email-patch workflows and would prefer not to migrate to pull requests.

Choose plain `git` over SSH when:

- The project is small, the contributor pool is small, and the operational simplicity is the dominant value.
- You want the absolute minimum dependency on platform tooling.

## Epitaph

The forge alternatives are the visible counter-evidence to the claim that GitHub is the only way to host source code, and they collectively define the boundaries within which version control's future, in the platform era, is being negotiated.
