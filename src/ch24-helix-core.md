# 24. Helix Core and the Art Pipeline

Chapter 8 covered Perforce as a system. This chapter covers Perforce — branded *Helix Core* in its current form — as the version control substrate for *art pipelines* in games, film, animation, visualization, and any other project where the dominant content is binary, the contributors are mostly not programmers, and the daily pace is set by fifty- or hundred-megabyte files moving through a pipeline of Digital Content Creation tools. This is the corner of the design space where Git is a malpractice, where Plastic and ClearCase have taken bites but not won, and where Perforce has remained dominant for thirty years. Understanding why is part of understanding the version control design space.

## What an art pipeline actually looks like

A AAA game in 2026 ships with somewhere between five and twenty terabytes of source content. The content is not the build output; it is the *source* — the layered Photoshop files, the Maya scenes, the Houdini graphs, the Substance materials, the audio sessions, the video pre-renders, the motion-capture data. The shippable game is a fraction of this, after baking, compression, and platform-specific processing. The version control system has to hold the source.

The contributors are diverse:

- *Artists*: 3D modelers, texture artists, animators, VFX artists, lighting artists, environment artists. Each works in specific DCC tools (Maya, ZBrush, Houdini, Blender, Substance Painter, Photoshop). Each produces files in formats those tools own.
- *Designers*: level designers, gameplay designers, narrative designers. They work in the engine's editor (Unreal, Unity, proprietary), producing engine-specific asset files.
- *Audio*: composers, sound designers, foley artists. Their working files are in DAW formats; their delivered assets are in WAV, OGG, or proprietary engine formats.
- *Engineers*: gameplay programmers, engine programmers, tools programmers, build engineers. They work in C++, C#, Python, sometimes scripting languages. They produce text source code that fits the standard VCS model.

The art pipeline is the full lifecycle of an asset:

1. *Source creation* in the DCC tool (e.g., a character is modeled in Maya).
2. *Iteration* with feedback from leads, art directors, game designers.
3. *Export* to an intermediate format (FBX, USD, OBJ, etc.).
4. *Import* into the engine, with engine-specific processing producing engine assets.
5. *Use* of the engine asset in scenes, levels, gameplay.
6. *Iteration* on the engine-side, sometimes feeding back into the source-side.

The version control system has to track every step, and the contributors at each step have different needs.

## Why Git is wrong for this

Three properties of art-pipeline work make Git structurally wrong:

*Binaries don't merge.* Two artists cannot independently edit the same Maya scene and have the system merge their changes. There is no three-way diff for a `.ma` file; there is no patch theory of binary blobs; the file format does not permit semantic merging at any scale that has been built. The only correct workflow is *one artist edits at a time*, which means *exclusive locking*, which Git does not have natively. Git LFS bolts on locking in 2016; the locking is advisory; the integration with DCC tools is rough.

*Files are large and cumulative.* A single Photoshop document for a character texture is fifty megabytes. The character has six texture maps. There are three hundred characters in the game. There are twenty revisions of each texture before ship. Multiply: the texture content alone is in the terabytes. Git stores every blob ever committed; cloning the repository means downloading all of it. LFS shards the storage, but cloning everything still means downloading all of it, just from a different server.

*The contributors are not programmers.* Asking an artist to learn `git rebase -i` is not a productivity investment; it is a tax on a workflow that already has its own complexity. The version control system has to be invisible most of the time, and where visible, has to use vocabulary the artist already uses: *check out this file*, *I'm working on it*, *here's my version*, *I'm done*, *integrate it*. Plastic SCM's Gluon is one attempt at the vocabulary; Perforce's P4V is another. Git's model is wrong for this audience even when the storage problem is solved.

## What Perforce/Helix Core gets right

The depot model maps cleanly. Source files live under depot paths; workspaces map subsets to local disk; artists' workspaces select the parts they need (`//depot/art/characters/...` for the character team, `//depot/art/levels/...` for the level team), and they sync only what they need. Multi-terabyte depots are routine; a typical artist's local workspace is fifty to two hundred gigabytes.

The file types and exclusive locking handle binaries correctly. `.psd` is `binary+l`; opening it for edit acquires the lock; nobody else can `p4 edit` it until you submit or revert; the workflow is clear. P4V (the GUI client) shows lock state visibly; `p4 opened -a` shows everyone's open files across the project.

The integration with DCC tools is mature. Plugins exist for Maya (`Perforce for Maya`), Photoshop (the *P4 plugin*), Houdini, Substance, Unreal Engine (built-in), Unity (`Plastic` is also bundled but Perforce is supported), Cinema 4D, ZBrush. Each plugin recognizes Perforce's lock state, lets the artist check out and submit from inside the tool, and integrates with the project's file structure. An artist can spend a whole day in Maya without touching the Perforce CLI; the plugin handles the version control as a side channel.

The streams model handles game project branching. A typical AAA project has:

- A *mainline* stream — the canonical project state.
- *Development* streams for major features (the *Combat* team's stream, the *Narrative* team's stream).
- *Release* streams for ship branches (the `//depot/release/v1.0/`, `//depot/release/v1.1/` lines).
- *Task* streams for individual feature work (light branches off development streams).

Streams know their parents; integration flows up and down the hierarchy through `p4 merge` (and `p4 copy` for promotion); the model handles the typical game-project pattern of "develop on dev streams, integrate to mainline, branch off mainline for release, hotfix on release and back-merge to mainline" with structural support.

The administrative model fits the team. A game studio has IT staff; the studio's Perforce server is administered by humans whose job is to administer it. Triggers, protect tables, file-type maps, replication setup, proxy servers — all of this is real work, and the studio has people whose job it is to do it. The model assumes operational maturity and rewards it.

## The asset pipeline as a build problem

Beyond version control proper, art pipelines have a *build* problem: the source content has to be processed into shippable assets, and the processing is deterministic but expensive. A character model in Maya gets exported, the export produces an FBX, the FBX is imported into Unreal, Unreal processes it into a `.uasset`, the `.uasset` is included in the cooked build. Some of these steps take seconds; some take hours.

The build infrastructure is typically not in Perforce. It is in `Jenkins`, `TeamCity`, Unreal's own *Build Graph*, custom in-house systems. The output of the build — cooked assets, Pak files, platform-specific packages — is sometimes in Perforce (in a `//depot/builds/` tree) and sometimes in separate artifact stores (S3, internal CDNs).

The version control story interacts with the build story at the boundary: the build records *which Perforce changelist it built from*. This is the value of Perforce's monotonic changelist numbers — `Build 12345 from CL 67890` is a unique, reproducible reference. A bug filed against the build can be traced to a specific changelist; a regression can be bisected (sometimes literally) by building intermediate changelists.

This is the kind of integration that Git's hash-based model makes more awkward, not impossible. Build numbers in a Git world are typically the commit hash plus a build counter; the counter is maintained out-of-band by the CI system. The Perforce world's monotonic changelist serves the same purpose, with the difference that the Perforce server *is* the source of truth and there is no out-of-band counter.

## Helix4Git: the bridge

Perforce's *Helix4Git* product (formerly *GitFusion*, and *Git Fusion* before that, with various naming changes over the years) lets developers use Git locally against a Perforce server. The use case is teams that want their engine programmers on Git for code while keeping their artists on Perforce for assets. Engine programmers work in Git, with the Helix4Git server presenting the relevant subset of the Perforce depot as a Git repository; their changes flow through Helix4Git to Perforce changelists.

Adoption has been mixed. The bridge works; the model is honest about what each side wants; the engineering investment to keep both sides happy is non-trivial. Most studios that consider Helix4Git either commit to it (engine team is fully Git-fluent, the bridge is worth it) or don't (engine team is Perforce-fluent enough that the bridge adds friction without enough benefit).

Plastic SCM (now Unity Version Control) has positioned itself as a similar bridge from the other direction: a system that works for both code and assets without requiring two systems and a bridge. The pitch is real; the trade-offs are different; teams in 2026 considering a fresh setup increasingly look at UVCS as an alternative to Perforce-plus-Helix4Git.

## What the art pipeline teaches

Three lessons for the design space in this book.

*Locking is not optional for some workflows.* Optimistic concurrency — let everyone edit, merge later — is a design choice that fits text. It does not fit binary art assets. Any version control system that wants to serve art-pipeline use cases needs locking as a first-class feature, not as an afterthought layered on through LFS. Git's locking story is the work of bolting on a feature the model didn't have; Perforce's is the work of having designed the model around the feature.

*Scale is a feature, not a bug.* Repositories of tens of terabytes are a real working size for game and film projects. Git's design did not contemplate this scale; sparse checkouts and partial clones extend Git's reach but do not match Perforce's natively-large operating point. The systems that handle terabytes well — Perforce, Plastic, ClearCase in its prime — did so because they were designed for it.

*The contributor's tools are part of the version control surface.* An artist's primary tool is Maya, not the version control client. The version control system has to integrate with Maya through a Maya plugin; the plugin has to honor the locking model, the workspace model, and the submit model. Git LFS plus a Maya Git plugin is, in 2026, a worse experience than the Perforce-Maya integration that has had two decades of refinement. The platform investment is part of the system's reach.

The art pipeline is the corner of the version control design space where Git's choices are most clearly the wrong ones. It is also the corner where the specialized systems — Perforce, Plastic, the older ClearCase — have continued to thrive. The dominant Git narrative in software-engineering culture often treats this as a niche, an exception, a peculiarity of game development. From the design-space perspective in this book, it is the rest of programming culture that is the niche; the broader category of "people producing valuable artifacts that need version tracking" is much larger than "people producing source code," and a fair portion of it has never had a reason to give up on the centralized-with-locking model.

## Epitaph

Helix Core in the art pipeline is the proof that the version control system that wins a market wins because it fits the work, and the work in games and film is shaped enough that the version control system that won everyone else's market did not win this one — and probably never will.
