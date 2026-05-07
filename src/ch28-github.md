# 28. GitHub

## Origin

GitHub was founded by Tom Preston-Werner, Chris Wanstrath, and PJ Hyett in 2008, with Scott Chacon joining shortly after. The site launched as a beta in February 2008 and as a paid service in April. The founders had been part of the Ruby and Rails community in San Francisco; the initial product was scratch-an-itch software for hosting Git repositories with a usable web UI, designed by people who used Git daily and did not enjoy the existing options (mostly cgit, gitweb, and `git daemon` plus a tarball server).

The ten years from 2008 to 2018 transformed the company and the field. Microsoft acquired GitHub in 2018 for $7.5 billion. Nat Friedman ran it from 2018 to 2021; Thomas Dohmke has run it since. Under Microsoft, the platform has continued to expand: Actions (CI/CD) shipped in 2019, Codespaces (cloud development environments) shipped in 2020, Copilot shipped in 2021 (and has since become an enormous business in its own right), Advanced Security and various enterprise features have layered on for the corporate market.

In 2026, GitHub hosts more than 100 million repositories and serves a substantial fraction of the world's open-source development. It is, by most measures, the largest single concentration of source code in human history. The platform's defaults are what working developers think of as how software development happens.

## The pull request as a social object

The pull request is GitHub's signature contribution to programming culture, and it is genuinely a *new* thing — not a Git operation, not a version control concept, but a piece of UI that turned out to be a unit of social interaction.

The mechanics: a contributor *forks* a repository (creates their own copy, writable, on the platform), commits to a branch in the fork, and opens a *pull request* against the upstream — a request that the upstream maintainer integrate the contributor's branch into the upstream's branch. The pull request has a title, a description, a list of commits, a diff view, an inline comment system, a review system (approve, request changes, comment), CI status, conflict status, and a merge button. Once approved, the merge button creates a merge commit (or a squash, or a rebase, depending on policy) on the upstream.

The pull request became the unit of *contribution* in the open-source world. A new contributor's first interaction with a project is increasingly *open a pull request*; their first conversation with maintainers happens in the pull request's review thread; their first commit lands by being merged through the pull request UI. The whole social arc of becoming a contributor — finding the project, understanding it, making a change, having it reviewed, getting it merged — happens inside one piece of UI.

Pull requests are also the unit of *coordination* inside teams. Internal company workflows have adopted pull requests for almost all changes; pull request templates, required reviewers, automated checks, branch protection rules, merge policies — all of these are GitHub features that codify the workflow into the platform. A team's pull request configuration is, in 2026, often the most important piece of process documentation the team has.

The pull request has costs the field underestimated for years. The diff-as-UI-element does not show the *intent* of a change as well as a structured commit message does, but most reviewers read the diff, not the message. The branch-and-merge model encourages large, batched changes (because each pull request is overhead), where smaller, more frequent changes would be more reviewable. The "approve" button conveys "I am OK with this" without distinguishing between "I read this carefully" and "this looks fine, I trust the author"; the difference is a real one and the UI flattens it. The dominant pull-request culture has produced a generation of developers who have never reviewed a patch by email, never sent a `git format-patch` to a maintainer, and treat the pull request as the only way contributions happen.

## What GitHub provides

The platform's surface in 2026 is large. The major components, in rough order of when they shipped:

*Repositories.* The core function. Git hosting, with a web UI for browsing, blame, history, search, and a basic editor for in-browser file edits. Free for public repositories without limit; private repositories free up to a contributor cap, paid above it.

*Issues.* A bug and task tracker, integrated with the repository. Issues have titles, descriptions, comments, labels, milestones, assignees, references to commits and pull requests, and a markdown rendering layer.

*Pull Requests.* As above. The integration with issues — `Fixes #123` in a PR description automatically closes the issue on merge — is the kind of cross-tool integration that integrated platforms can deliver and composable tools cannot.

*Pages.* Static site hosting from a repository. Most open-source project documentation hosted on `*.github.io` runs on Pages.

*Releases.* Tagged commits, with attached release artifacts (tarballs, binaries, signed checksums), release notes, and a permanent URL. The release page is what most users see when they download software.

*Wiki.* Per-repository wiki, with its own Git history. Less prominent than it was in the 2010s; many projects have moved documentation to Pages or to dedicated documentation hosts.

*Discussions.* A forum-like discussion tool, distinct from issues, for community questions and longer-form conversations. Added in 2020; adoption is uneven.

*Actions.* CI/CD as a YAML workflow language, executed on GitHub-hosted runners or self-hosted runners. Integrated with pull requests: a PR's CI status is part of the merge gate. Actions has its own marketplace of pre-packaged steps; the marketplace is significant in its own right.

*Packages.* Artifact registry for npm, Maven, NuGet, RubyGems, Docker images, and others. Integrated with the repository; access controlled through the same teams and permissions.

*Codespaces.* Cloud development environments. A pull request gets a one-click button to open a full development environment, with the repository checked out and dependencies installed, in the browser or in a local VS Code. The pricing has been controversial; the technology is impressive.

*Copilot.* AI code completion and chat, originally based on OpenAI's models, since expanded. Copilot is a major Microsoft product line in 2026, with multiple tiers, strong adoption in many development teams, and a complicated relationship with the open-source code on which it was originally trained.

*Sponsors.* Direct payment from users to project maintainers, integrated with the repository. The model has been a meaningful revenue source for a small number of high-profile maintainers; for most projects, it has not changed the economics.

*Advanced Security.* Code scanning, secret scanning, dependency vulnerability tracking, supply-chain provenance through SLSA support, signed commits via Sigstore integration, audit logs. Sold as part of GitHub Enterprise.

*API.* REST and GraphQL APIs that expose all of the above. The API is the part that makes GitHub a programmable platform; integrations, bots, and tooling depend on it.

*Marketplace.* Apps, Actions, and integrations sold through the platform. A market in its own right.

The list keeps growing. The pace has accelerated under Microsoft; the amount of functionality the platform has acquired in the years 2018–2026 is larger than what it had before.

## The exemplar on GitHub

The exemplar from Chapter 3 plays out in GitHub as a series of interactions with the platform layered on Git operations.

*Operation 1 (initial import)* uses `git push` to create the repository on GitHub; the platform creates a project page automatically.

*Operation 2 (linear development)* is plain Git. The commits show up on the project's commit list page; the file blame is generated automatically.

*Operation 3 (branch)* is `git push -u origin currency-conversion`; the branch becomes visible on the platform; Jonas can open a pull request against `main` whenever he wants.

*Operation 4 (file rename)* is plain Git. GitHub's diff view renders the rename as a rename, using its own heuristic on top of Git's blob comparison.

*Operation 5 (binary file added)* is plain Git, with optional Git LFS for large binaries. The image renders inline in the file browser; subsequent edits show a "binary file" placeholder in the diff.

*Operation 6 (parallel edits)* and *Operation 7 (merge with conflict)* happen through pull requests. Aditi opens a PR for her README change; Jonas opens a separate PR for his change; the second PR to merge sees the conflict in the platform's UI and prompts Jonas to resolve it. The conflict resolution can happen in the web UI for simple cases; for the `parser.py` conflict, Jonas resolves locally and force-pushes to the branch.

*Operation 8 (botched commit)* is `git commit --amend` plus `git push --force-with-lease`. The platform records the force-push in the audit log and notifies any subscribers. For a published commit, the conventional GitHub workflow is a follow-up commit reverting the change, not a force-push.

*Operation 9 (tag)* is `git tag` plus `git push --tags`. The tag becomes available on the platform's Releases page.

*Operation 10 (release)* uses GitHub Releases. The tag triggers a Release object; uploaded artifacts attach to it; release notes are rendered as markdown; the Release page is the public-facing URL for the version. GitHub Actions workflows can automate the build-and-upload pipeline on tag push.

The platform absorbs operations 5 (binary handling, somewhat), 6, 7, 8, 9, and 10 into platform-specific UIs that simplify the common cases at the cost of binding the project to GitHub for those workflows.

## What the platform gets right

*Discoverability.* The platform makes finding projects, finding maintainers, and finding contributors trivial. The cost reduction here is hard to overstate.

*Onboarding.* A new contributor's first interaction with an open-source project is, in the GitHub world, low-friction in a way it never was in the SourceForge world.

*Integration.* Cross-references between issues, pull requests, commits, and releases work; CI status is part of the merge gate; release artifacts are linked to tags; the API exposes all of this for automation.

*Reach.* Most working developers have a GitHub account. Building on GitHub means meeting users where they are.

*Investment.* The platform's engineering team is large and competent; functions are maintained, security issues are addressed, the API is stable.

## What the platform gets wrong

*Lock-in.* Projects on GitHub have data — issues, pull requests, discussions, Actions configurations, security advisories — that does not migrate cleanly to other platforms. The `git` part of the data is portable; the platform-specific part is not.

*Centralization.* A platform outage is an outage of the world's open-source development. GitHub has had several multi-hour outages over the years; each has been a noticeable event. A jurisdictional restriction on GitHub (the platform was, at various points, restricted in Iran, in Russia, in regions of China) becomes a restriction on the open-source projects of developers in that jurisdiction.

*Standardization of workflow.* The pull request, GitHub Actions, the platform's review model, the Issues data structure, the Markdown flavor — all of these have become *the* way to do those things, even when other ways might be better for some projects. Projects that try alternatives (mailing-list workflows, Gerrit-style reviews, etc.) feel like they are swimming against the current.

*Monetization pressure.* GitHub is a Microsoft product; Microsoft is a public company; the platform has commercial obligations. Some functions are offered free for public repositories; some are paid; some have changed pricing over the years. A project's reliance on GitHub is a reliance on the future of GitHub's pricing.

*Copilot's training data.* GitHub's free hosting of open-source projects has been used as training data for Copilot, a paid commercial product. The legal and ethical contours of this are still being argued. For some maintainers, this changes the relationship with the platform; for others, it does not. The fact that maintainers have to think about it is itself a sign of the absorption Chapter 27 named.

*The platform's view of "version control" is incomplete.* GitHub's UI is shaped around Git's snapshot model; the things this book has called out as buried — patch identity, free cherry-picking, structural rename tracking, conflicts as objects — are not visible in GitHub's UI. The platform has trained a generation of developers in a particular shape of version control; that shape has narrowed the field's notion of what version control is.

## Ecosystem reality

GitHub in 2026 is the platform. The competitors (covered in the next chapter) are real but smaller. Many large open-source projects host on GitHub by default, sometimes with mirrors elsewhere. Most companies' internal Git hosting is GitHub Enterprise (for the self-hosted version), GitHub Enterprise Cloud, or one of the major alternatives.

The platform's API is what most tooling targets first. CLI tools (`gh`), IDE integrations, code search engines, issue triage bots, security scanners — all are GitHub-shaped first, often other-platform-shaped second.

## When to use it; when not to

Use GitHub when:

- The project is open source and wants the discoverability of the dominant platform.
- The team's contributors expect GitHub's UX.
- The integrated platform features (Actions, Pages, Packages, Codespaces) match the team's needs and the pricing is acceptable.

Do not use GitHub when:

- Lock-in is a primary concern. Self-hosted alternatives (Gitea/Forgejo, GitLab CE, Sourcehut) preserve the project's ability to leave.
- The project's audience or contributors are in jurisdictions where GitHub access is restricted.
- The team has strong preferences for non-GitHub workflows (mailing-list patches, Gerrit reviews) and wants tooling that supports those workflows natively.

## Epitaph

GitHub is the platform that turned the pull request into a social object and made the term "open source" mean "hosted on a Microsoft-owned server" — and is the largest concentration of programmer culture in one place that has ever existed.
