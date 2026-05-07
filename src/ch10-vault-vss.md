# 10. Vault, Visual SourceSafe, and the Corporate Backwater

This chapter covers a category, not a single system. The category is *commercial centralized version control for the corporate Windows world from roughly 1990 to 2010* — the systems that shipped on price lists alongside compilers, that came preinstalled in corporate IT inventories, that handled the actual day-to-day version control for millions of working programmers in industries the open-source community did not pay attention to. They are mostly gone now, in different stages of decay. Treating them per-system at full chapter length would belabor the same points; treating them together gives the reader the texture of an era.

The exemplar scenario is not walked through in full for any of these systems. Each profile notes how the exemplar would render and where the system was distinctive.

## Microsoft Visual SourceSafe

Visual SourceSafe was the corporate version control system most working Windows programmers encountered first, for the worst possible reason: it shipped with Microsoft developer tools.

The product began life at One Tree Software in the early 1990s. Microsoft acquired it in 1994 and rebranded it as Visual SourceSafe, starting at version 3.0. Subsequent releases — 4.0, 5.0, 6.0, 6.0d — extended the system through the late 1990s. Version 6.0d was the last substantial update; Microsoft continued to ship it through the Visual Studio 2005 era and effectively retired it when Team Foundation Server became the recommended path.

The original design choice that defined VSS was *the database is a set of files on a shared network drive*. There was no server process. Two developers ran the VSS client on their own machines, and the client reached out to a UNC path (`\\server\share\repo`) holding the database files, locking them at the filesystem level for the duration of operations. This was simple to set up, cheap to administer, and prone to subtle and unsubtle failures.

The failures were many. Concurrent operations sometimes corrupted the database, especially under network glitches. The locking was advisory; a crashing client could leave files locked indefinitely. The integrity-checking tool (`analyze.exe`) was a routine part of operations because corruption was a routine occurrence. The wire protocol was based on SMB file sharing, which had no notion of transactions, so the database's consistency depended on the client's good behavior. There were no atomic multi-file commits in the modern sense; checkin was per-file with a shared comment.

The locking model was pessimistic and per-file by default. A developer ran *Check Out* on a file, edited it, and ran *Check In*. While checked out, the file was locked from other developers. The file showed up as read-write on disk after checkout, read-only otherwise, much like Perforce's model — except enforced through file-attribute manipulation rather than a server-aware kernel layer. There was a *multiple checkouts* mode that loosened this, with the consequence that two developers could overwrite each other's changes if they were not careful.

The exemplar in VSS plays out painfully. The initial import is a wizard. The branch operation is a *share with branch*: VSS shared files between projects (creating a soft link in the repository), and a branch was a share that diverged after the share point. Renames were possible but not robust; rename history was lost in many edge cases. The botched-commit recovery was through *Rollback*, available only for the latest version. Tags were called *labels* and were applied by walking the project recursively. The release workflow was manual: get a clean working copy at the label, build, ship.

VSS's reputation in the open-source community is famously bad — Joel Spolsky's mid-2000s essays on it were unkind and largely accurate — but its install base in the early 2000s was vast. Banks, insurance companies, government agencies, and medium-sized software shops in industries Joel did not write for ran VSS for a decade or more. The migration to TFS, then to Git via TFS-Git interoperability, then to Azure Repos and finally GitHub Enterprise, took the better part of fifteen years.

In 2026, VSS is dead. Microsoft's documentation has retreated to historical pages. Repositories that survive are either being read-only-archived or being migrated; the standard tool for migration is `vss2git`, which extracts history into Git with various levels of fidelity. The corruption that plagued production use in the 2000s makes this migration occasionally impossible: some VSS repositories have damage so old it cannot be cleanly extracted, and the team accepts that the pre-2008 history is gone.

## SourceGear Vault

Vault was SourceGear's response to the question *what would Visual SourceSafe look like if it had been designed correctly?* SourceGear was Eric Sink's company; the team had built SourceOffsite, a tool for using VSS over the internet without the file-share dependency, and had learned where VSS's structural problems were. Vault, first released in 2003, was the new system built from scratch with the lessons applied.

Vault used Microsoft SQL Server as its backend. There was a real server process, real transactional commits, real atomic multi-file operations, and real referential integrity. The wire protocol was an HTTP-based one (Vault used SOAP early on, then a more efficient binary protocol). Branches and labels were first-class. The client UI was deliberately VSS-like, so the migration from VSS to Vault required minimal retraining.

Vault did not become dominant. By the time it shipped, the field was already starting to move toward Subversion in the open-source community and toward TFS in Microsoft shops. Vault occupied a middle ground: better than VSS for teams that wanted to stay on a Microsoft-stack commercial product, more familiar than Subversion for those teams, but not free, not Microsoft-blessed, and not as feature-rich as TFS for shops that were going to spend money anyway.

Vault is, however, *still maintained* in 2026. SourceGear continues to ship it; the company has been small and focused for the entire run of the product's life. Vault Pro and Vault Standard are current. Customers tend to be small to medium teams in industries where they want a paid-for, supported, SQL-Server-backed centralized VCS and do not want Git or Subversion. The use case is narrow but it has not disappeared.

In the exemplar, Vault behaves much like Subversion: atomic commits with global revision numbers, real branches, real labels, real merging, sane history. Mireille's botched commit cannot be amended; the correction is a follow-up. The rename is a first-class operation. The binary file is handled via the file-type mechanism (text, binary).

If you encounter a Vault repository in 2026 in a small business setting, it is likely working fine, the team is satisfied, and there is no live engineering problem. SourceGear's support is reachable and competent. The interesting case is rare.

## StarTeam

StarTeam was Borland's version control system, originally from Starbase Corporation (acquired by Borland in 2003). It survived through Borland's acquisition by Embarcadero, through Embarcadero's various ownership changes, and through a transfer to Micro Focus, and as of 2024 sits inside OpenText after that acquisition. The product is currently called Micro Focus StarTeam, though OpenText branding is taking over.

StarTeam was a high-end commercial centralized system with a SQL backend (originally proprietary, later supporting MS SQL Server, Oracle, and others). It had branches, labels, change packages (the unit-of-work concept comparable to Perforce changelists), strong integration with project management tools, and a Java client that ran on Windows, Mac, and Linux. The system targeted regulated and quality-sensitive industries; its audit trail was thorough, its access control was elaborate, and it integrated with bug tracking and requirements management as a single platform.

The exemplar in StarTeam plays out cleanly. Atomic multi-file commits, first-class branches and labels, real renames, binary file handling via file-type configuration, optional pessimistic locking. Mireille's botched commit is correctable through change-package operations available to her role; the audit trail records what she did. The release workflow ties into StarTeam's promotion model, with named promotion states (development → testing → production) that gate transitions.

In 2026, StarTeam is still sold and supported, at low volume. Its users are concentrated in industries — financial services, manufacturing, defense — where the integrated project/version-control/requirements model matches the regulatory environment. New customer acquisition is rare; existing customer retention is high.

## PVCS Version Manager

Originally a product of Polytron Corporation in the 1980s; acquired by Serena Software in 1989 along with the rest of the Polytron product line; became part of Micro Focus in 2014; now part of OpenText. PVCS Version Manager is one of the oldest commercial version control systems still being sold.

The system targeted Unix and Windows enterprise developers from its inception. The storage was per-file, similar to RCS, with PVCS-specific extensions. Later versions added project-level commit semantics, branches, and Windows-friendly tooling. The associated product line — PVCS Tracker (issue tracking), PVCS Configuration Builder, PVCS Dimensions (a heavier configuration-management product, separately developed) — formed an integrated suite that competed with Rational and ClearCase in the enterprise.

The exemplar in PVCS Version Manager rendered as something close to RCS-with-project-management. Atomic project-level commits were available in later versions but not always used. Branches and labels existed. Locking was the default model. Merge support was present but unloved.

In 2026, PVCS Version Manager is in deep maintenance. OpenText sells it; customers run it; new adoption is essentially zero. Migration tools to Git exist and are routinely used by teams whose long contracts have expired and whose new corporate standards are GitHub Enterprise.

## MKS Source Integrity / PTC Integrity

MKS (Mortice Kern Systems, the Canadian company that also wrote the MKS Toolkit of Unix utilities for Windows) shipped Source Integrity as part of its product line from the 1990s. The system became Integrity Source after a rename, then PTC Integrity after PTC acquired MKS in 2011, and is currently sold as PTC Codebeamer (after a further rename and product unification).

The system's defining property in its prime was the integration of version control with requirements management, change management, and test management as a single product targeting regulated industries — automotive, medical devices, aerospace. The version control component was solid: atomic commits, branching, labeling, decent merging, with a SQL backend. What made customers buy it was the *Application Lifecycle Management* story, in which version control was one of several integrated capabilities.

The exemplar plays out competently. The renames work; the binary file handling is configurable; the merge is solid. The botched-commit recovery follows the standard centralized model: forward correction, audit trail preserved.

In 2026, PTC Codebeamer is current. New customers exist. The market is narrow — companies that need ISO 26262 traceability, FDA 21 CFR Part 11 audit, or DO-178C documentation, and want it integrated. The price reflects the market; the alternatives are Polarion (also from Siemens, similar pitch) and bespoke chains of Git-plus-Jira-plus-Polarion.

## Synergy / CM Synergy

Continuus's CM Synergy, acquired by Telelogic, then IBM in 2008, then sold around in 2017, is now part of OpenText and called OpenText Synergy. The system shipped as a *task-based* configuration management product: the unit of work was a *task*, and the version control system tracked which versions of which files belonged to which tasks.

The task-based model was a real idea. A change to the codebase was associated with a specific task ID, and the system maintained the relationship between tasks, files, and *projects* (configurations of files chosen by task membership). This pre-figured the changeset/activity model in UCM and many of the later DVCS systems' commit-as-changeset model. It was also, like ClearCase's config specs, expressive in proportion to operational overhead.

In 2026, Synergy is in legacy maintenance. The customer base is concentrated in defense, aerospace, and large enterprise IT shops that have run it for two decades.

## AccuRev

AccuRev shipped from 2003 under the AccuRev brand; the company was acquired by Micro Focus in 2017 and the product is now part of OpenText. AccuRev's distinctive feature was its *stream-based* model. Streams were named, hierarchical lines of development, with parent-child relationships and inheritance: changes could be promoted up a stream hierarchy from development to staging to production, and child streams inherited unchanged content from parents while overriding the rest.

This stream model was prescient. Perforce streams (2011) borrowed its vocabulary and model freely. Plastic SCM's branch-explorer-driven workflow has overlaps. The pattern of "lines of development with inheritance and explicit promotion" is a real piece of the design space, and AccuRev was where it was first commercialized at scale.

The exemplar in AccuRev would look much like the Perforce streams version: branches as first-class objects with parent relationships, atomic commits, real merges, named promotion paths from development streams to integration streams to release streams. Locking is supported but not the default; the model is optimistic concurrency with strong stream discipline.

AccuRev is alive in 2026 but quiet. OpenText sells it; the customer base has not grown; migration to Git is occasional but not a stampede.

## What this category has in common

Several patterns recur across this set:

- *SQL backend.* All except VSS used a real database. The SQL backend gave them transactional integrity and the ability to scale beyond what VSS's filesystem-database model could.
- *Pessimistic locking by default, optimistic available.* The corporate world expected file locks; the systems provided them and let teams loosen the constraint as desired.
- *Integrated process tooling.* StarTeam, MKS, Synergy, AccuRev all came with bug tracking, project management, and other tools as part of the same product. Buying one bought the suite.
- *Atomic multi-file commits.* By the late 1990s, all of these systems had this; VSS was the exception that proved why everyone else had it.
- *Strong audit semantics.* Customers in regulated industries paid for the audit trail; the systems were optimized to keep it.
- *Per-seat pricing.* Salesforce-style commercial software pricing meant the systems were expensive enough to require organizational decisions to adopt, and equally expensive to leave.

The category exists in the past tense, partly. VSS is dead. PVCS, AccuRev, StarTeam, Synergy are all in slow decline. Vault is small but stable. PTC Integrity / Codebeamer is genuinely current in regulated industries. The aggregate trend is migration toward Git, sometimes through TFS or Azure DevOps as a stepping stone, with the regulatory affordances being recreated through Git plus a layer of certified tooling (signed commits, immutable hosted-forge audit logs, documented branch protection rules).

## What got lost

Two specific things from this category deserve preservation in the inventory.

First, *integrated process tooling*. The systems in this chapter shipped version control bundled with bug tracking, requirements management, and review tools. The split between version control and forge that defines the modern Git world is, in part, a regression from the integration these systems offered. Plastic SCM and Fossil are the surviving systems in this book that retain the integration. GitLab tries to recreate it on top of Git. Most teams have accepted the split.

Second, *per-file pessimistic locking as a default option*. VSS aside, the systems in this chapter offered locking as a real, working, supported part of the workflow. Git LFS later bolted this on for binaries, but the cultural defaults of the Git world are fundamentally optimistic. For some teams — small, regulated, with binary content — pessimistic locking on a server they trust was a better answer than what they got from migrating away.

## Epitaph

The corporate backwater was where most version control happened in the 1990s and early 2000s, and almost none of it survives in the form of dominant ideas — but the bundling of audit, process, and version control as one product was a real position that the field has not entirely re-occupied.
