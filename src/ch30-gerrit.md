# 30. Gerrit and the Code Review Lineage

This chapter covers code review as its own discipline. Code review predates GitHub's pull request by decades; serious code-review tooling exists in forms that the pull request only approximates; and the design choices in those tools represent positions on what code review *is* that the pull-request world has flattened. The chapter focuses on Gerrit, the most prominent example, and notes the rest of the lineage.

## The premise that distinguishes code review from collaboration

The pull request, as it exists on GitHub and its imitators, treats *contribution* and *review* as a single unit. A contributor opens a pull request; the maintainer reviews; on approval, the work is integrated. Review is one ceremony in the contribution flow.

Code review as a discipline — as practiced at Google, at Cisco, at projects that take review seriously — treats *review* as the central thing, with everything else (the change, the contribution, the integration) shaped around it. A change is something to be reviewed; review consists of multiple rounds of feedback and revision; the unit of integration is *a reviewed-and-approved change*, not a branch's worth of accumulated work. The system is designed to support this view, and the workflow primitives reflect it.

Gerrit is the most coherent expression of this view in a deployable system.

## Origin

Gerrit was started at Google around 2007–2008 by Shawn Pearce, who needed a code review tool that would support Google's Android development. Android, then in its pre-launch period, was being built collaboratively across Google and partner companies; the existing Google-internal code review tooling (Rietveld, written by Guido van Rossum, public-facing as `codereview.appspot.com`) served well for Subversion and the early phases but was not the right fit for the Git-based workflow Android needed.

Pearce built Gerrit as a Git-based code review system, drawing on Rietveld's UX ideas and on Google's internal practices. The first public release was 2008. Pearce continued to maintain Gerrit through Google and after; Luca Milanesio and the Eclipse Foundation took over substantial parts of the maintenance later. Gerrit is a Java application, deployed against a Git server, with a web UI, an SSH interface for the Git operations, and a substantial plugin architecture.

In 2026, Gerrit is alive and used. Major users include the Android Open Source Project (AOSP), Chromium, Eclipse, OpenStack, the Linux Foundation projects (LFEdge, LF AI, several others), Qt, and parts of Wikipedia's infrastructure. The user base is concentrated in projects that have explicit policies of "every change is reviewed" and that want tooling shaped around that policy.

## The model

A *change* in Gerrit is the central object. A change has:

- A unique *Change-Id* (a hash, recorded in the commit message via a `Change-Id:` trailer).
- One or more *patch sets* — successive revisions of the change in response to review.
- A set of *reviewers*, with their votes (`Code-Review+2`, `Verified+1`, etc.).
- A set of *comments*, attached to specific lines or to the change as a whole.
- A *status* (open, merged, abandoned).

The contribution workflow:

1. Contributor commits locally with a `Change-Id:` line in the commit message (a `commit-msg` Git hook generates one automatically).
2. Contributor pushes to a special ref: `git push origin HEAD:refs/for/main`. The `refs/for/` namespace is Gerrit's signal: this push is a request for review on the named target branch.
3. Gerrit creates (or updates, if Change-Id matches an existing change) a change object and notifies reviewers.
4. Reviewers comment, request changes, or approve.
5. Contributor responds by amending the commit (preserving the Change-Id) and pushing again to `refs/for/main`. The amended commit becomes a new patch set on the same change.
6. When the change has the required votes (typically `Code-Review+2` and `Verified+1`), a project committer hits *Submit*. Gerrit applies the change to the target branch, typically as a fast-forward or rebase.

The model has several distinctive properties:

*Each change is one commit.* Gerrit's normal operating mode treats one commit as the unit of review. If a contributor wants to submit multiple related changes, they push multiple commits, each with its own Change-Id. The commits form a *stack* in Gerrit's terminology, with later commits depending on earlier ones being merged first.

*Change identity is preserved across revisions.* The `Change-Id:` trailer is the stable identifier; commit hashes change with each amend; the change-as-an-object accumulates patch sets without losing identity. This is the same insight that Sapling and Jujutsu encode in change-IDs natively, expressed here through commit-message convention.

*Review is structured.* Vote levels (`Code-Review-2`, `-1`, `0`, `+1`, `+2`; `Verified-1`, `0`, `+1`) are part of the model. Different reviewers can have different vote ceilings; project policies determine what combination of votes is required to submit. The "approve" / "request changes" / "comment" trichotomy of GitHub is a flattening of this richer structure.

*Comments are first-class.* Inline comments on specific code locations, with thread-style replies, with resolution status, with the ability to mark "done" — these are first-class objects in Gerrit, not a side feature. The review UI is built around the comments.

*Submit is a separate operation.* The contributor does not merge; a project committer does. The committer's role is gating; only those with the appropriate permissions can submit changes that have the required votes. This separation enforces a discipline that the pull-request world's "I'll merge my own PR" pattern does not.

## The exemplar walkthrough

Some operations from Chapter 3 take a different shape in Gerrit's workflow.

### Operations 1, 2, 9, 10

The initial import, linear development, tagging, and release are essentially Git plus Gerrit hooks. The contributor's local Git is unchanged; pushes go through `refs/for/` for review and `refs/heads/` (with appropriate permissions) for direct pushes when a change has been submitted.

### Operation 3 — Branch and stacked changes

Jonas's `currency-conversion` work is two commits. He pushes them both:

```
$ git push origin currency-conversion:refs/for/main
```

Gerrit creates two change objects, recognizing that the second commit's parent is the first. The two changes form a stack; the second's review can begin while the first is still under review, with the dependency made explicit in the UI.

### Operation 4 — File rename

The rename commit is a single change. Gerrit shows the rename in the diff view; reviewers can comment on the rename specifically.

### Operation 5 — Binary file added

Gerrit handles binaries through Git's normal storage; review comments can be attached to the binary file (asking about its source, the licensing, etc.) but not to specific bytes. Most projects with serious binary asset workflows use a different system entirely (Perforce's Helix Swarm, Plastic SCM's review).

### Operation 6 — Parallel edits

Aditi and Jonas push their respective changes; each becomes a separate change in Gerrit. The changes have the same target branch; merge conflicts arise on submit if both are approved before either is merged. The first to submit succeeds; the second's submit fails until they rebase their change on top of the new branch state and re-submit. Rebase-on-submit can be configured per project.

### Operation 7 — Merge with conflict

Merges in Gerrit are typically explicit: a merge commit is a change like any other, reviewed and submitted. Most projects discourage merge commits in favor of rebase-and-fast-forward; the workflow naturally produces a linear main branch.

### Operation 8 — Botched commit

Mireille's botched change can be revised by amending and re-pushing:

```
$ vi src/ledger/report_register.py    # remove debug line
$ git commit --amend
$ git push origin HEAD:refs/for/main
```

Gerrit recognizes the same Change-Id, attaches the amended commit as patch set 2 of the existing change, and notifies reviewers. The botched patch set is preserved (Gerrit retains all patch sets for audit), but the change as a whole now has a clean version awaiting review.

The Gerrit model treats this as the *normal* case: changes are revised in response to review. The Git operation is `commit --amend`; the workflow operation is "submit a new patch set." There is no separate force-push or rebase needed; the system is designed for this.

## What Gerrit gets right

*Review as the central unit.* Code review is not bolted onto a contribution flow; the contribution flow is shaped around review.

*Patch sets as the revision history of a change.* Gerrit retains every patch set; reviewers can compare any two; the audit trail is structured.

*Inline comments with thread structure and resolution.* The UI for comment-by-comment discussion is mature; threads can be resolved, marked unresolved, replied to.

*Vote structure with project-specific policies.* The flattening of approval into "approve / request changes" loses information that the multi-level vote model captures.

*The Change-Id mechanism.* Stable change identity through commit-message convention is a workable solution that does not require modifying Git.

## What Gerrit gets wrong (or makes hard)

*The UI is dense.* Gerrit's web interface is feature-rich and information-dense in a way that GitHub's pull request UI deliberately is not. New users find Gerrit unfriendly; the friendliness gap is part of why GitHub's model spread.

*Multi-commit changes are awkward.* The "one commit per change" convention works well for single, focused changes; it works less well for refactors that genuinely require multiple commits as a single logical unit. Stacked changes help but are not as smooth as some users would like.

*The infrastructure is heavy.* Gerrit is a Java application with a database (H2 for small deployments, PostgreSQL for larger), a Git backend, a web server, an SSH server, plugins, and a configuration that grows with use. Self-hosting is non-trivial.

*The model is opinionated.* Projects that don't want every change reviewed find Gerrit's defaults too rigid. The opinions are part of why projects that *do* want review choose Gerrit; the opinions are also the reason Gerrit has not won the broader market.

## The rest of the lineage

*Rietveld* — Guido van Rossum's earlier review tool, the predecessor to Gerrit, ran for years on `codereview.appspot.com` for Google open-source projects. The tool is now discontinued; its ideas live in Gerrit and elsewhere.

*Phabricator* — Facebook's review tool, open-sourced in 2011, deprecated by Facebook in 2021. Phabricator was a sprawling suite of tools (Differential for review, Maniphest for tickets, Phriction for wikis, Diffusion for code browsing) that handled review with the same seriousness as Gerrit but with different conventions: the unit of review was a *Differential revision*, contributors used the `arc` tool to upload reviews, and the model was less Git-native than Gerrit's. *Phorge* is the community fork that continues active development; it is alive in 2026.

*ReviewBoard* — an earlier (started 2006) self-hosted review tool, popular in some companies, still maintained, smaller user base. The model is similar to Phabricator's: reviews are first-class objects, uploaded with a tool, separate from the version control system.

*Crucible* — Atlassian's review tool, integrated with Bitbucket and Fisheye. Used in some Atlassian-stack environments. Less prominent in 2026 as Atlassian has consolidated around Bitbucket Cloud.

*Stacked-diff tools* — `Phorge` continues the Differential lineage; *Reviewable* is a third-party review tool layered on GitHub PRs; *ReviewStack* (Meta, open-sourced alongside Sapling) is the modern stacked-diff tool. These are the spiritual continuations of Gerrit's serious-review-as-discipline position, in forms that fit other workflows.

## When to reach for serious review tooling

Reach for Gerrit when:

- The project requires every change to be reviewed and approved before integration.
- The team values structured comments, multi-patch-set revisions, and explicit vote semantics.
- The contributor pool is large and the review discipline needs to scale.
- Open-source compatibility matters and the project would benefit from joining the Gerrit-using ecosystem (Android, Chromium, OpenStack).

Reach for Phorge or ReviewBoard when:

- The team's review needs are similar to Gerrit's but the workflow is non-Git or the cultural fit is different.

Use GitHub-style pull requests when:

- The project's review needs are simpler.
- The team's contributors expect the GitHub model and resist learning something different.
- The integration with the rest of the GitHub platform (CI, deploys, releases) outweighs the loss of review-specific features.

## Epitaph

Gerrit is the version control adjunct that took code review seriously enough to build a system around it, and is the working evidence that the pull request is an approximation of something that has its own structure and history.
