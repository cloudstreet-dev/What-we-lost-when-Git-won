# Appendix A. The Exemplar Repository Rendered in Five Systems

The five systems in this appendix span the design space the book has covered: Subversion (atomic centralized), Perforce (centralized with locking), Mercurial (DVCS with revlog storage), Git (snapshot DVCS), and Pijul (patch theory). Each operation from Chapter 3 is shown in each system's native commands, side by side, for direct comparison.

This is a reference, not a tutorial. The chapters for each system give the model and context; this appendix gives the commands.

## Operation 1 — Initial import

**Subversion**
```
svnadmin create /var/svn/ledger
mkdir -p /tmp/import/{trunk,branches,tags}
cp -r /path/to/initial/ledger/* /tmp/import/trunk/
svn import /tmp/import file:///var/svn/ledger -m "Initial import"
svn checkout file:///var/svn/ledger/trunk ledger
```

**Perforce**
```
p4 client    # configure workspace
cd /path/to/ledger
p4 reconcile -ad
p4 submit -d "Initial import"
```

**Mercurial**
```
cd /path/to/ledger
hg init
hg add .
hg commit -u "aditi@example.org" -m "Initial import"
```

**Git**
```
cd /path/to/ledger
git init
git add .
git commit -m "Initial import"
```

**Pijul**
```
cd /path/to/ledger
pijul init
pijul add --recursive .
pijul record -m "Initial import"
```

## Operation 2 — Linear development

**Subversion**
```
vi src/ledger/parser.py
svn commit -m "parser: handle blank lines and comments"
```

**Perforce**
```
p4 edit src/ledger/parser.py
vi src/ledger/parser.py
p4 submit -d "parser: handle blank lines and comments"
```

**Mercurial**
```
vi src/ledger/parser.py
hg commit -m "parser: handle blank lines and comments"
```

**Git**
```
vi src/ledger/parser.py
git add src/ledger/parser.py
git commit -m "parser: handle blank lines and comments"
```

**Pijul**
```
vi src/ledger/parser.py
pijul record -m "parser: handle blank lines and comments"
```

## Operation 3 — Branch (currency-conversion)

**Subversion**
```
svn copy file:///var/svn/ledger/trunk \
         file:///var/svn/ledger/branches/currency-conversion \
         -m "Create currency-conversion branch"
svn switch file:///var/svn/ledger/branches/currency-conversion
```

**Perforce** (using streams)
```
p4 stream -t development -P //depot/ledger/main //depot/ledger/currency-conversion
p4 client    # set Stream: //depot/ledger/currency-conversion
p4 sync
```

**Mercurial**
```
hg bookmark currency-conversion
# or, for named branches:
hg branch currency-conversion
```

**Git**
```
git switch -c currency-conversion
git push -u origin currency-conversion
```

**Pijul**
```
pijul fork currency-conversion
pijul switch currency-conversion
```

## Operation 4 — File rename (and split)

**Subversion**
```
svn move src/ledger/reports.py src/ledger/report_balance.py
# edit report_balance.py
svn add src/ledger/report_register.py
svn commit -m "reports: split into balance and register modules"
```

**Perforce**
```
p4 edit src/ledger/reports.py
p4 move src/ledger/reports.py src/ledger/report_balance.py
# edit report_balance.py
p4 add src/ledger/report_register.py
p4 submit -d "reports: split into balance and register modules"
```

**Mercurial**
```
hg mv src/ledger/reports.py src/ledger/report_balance.py
# edit report_balance.py
hg add src/ledger/report_register.py
hg commit -m "reports: split into balance and register modules"
```

**Git**
```
git mv src/ledger/reports.py src/ledger/report_balance.py
# edit report_balance.py
git add src/ledger/report_register.py
git commit -m "reports: split into balance and register modules"
```

**Pijul**
```
pijul mv src/ledger/reports.py src/ledger/report_balance.py
# edit report_balance.py
pijul add src/ledger/report_register.py
pijul record -m "reports: split into balance and register modules"
```

## Operation 5 — Binary file added (logo.png)

**Subversion**
```
svn add docs/logo.png    # auto-detects binary
svn commit -m "docs: add project logo"
```

**Perforce** (with `*.png` set to `binary+l` in typemap)
```
p4 add docs/logo.png
p4 submit -d "docs: add project logo"
```

**Mercurial**
```
hg add docs/logo.png
hg commit -m "docs: add project logo"
```

**Git** (with optional LFS)
```
git lfs track "*.png"
git add .gitattributes docs/logo.png
git commit -m "docs: add project logo"
```

**Pijul**
```
pijul add docs/logo.png
pijul record -m "docs: add project logo"
```

## Operation 6 — Parallel edits to README.md (Aditi commits first)

**Subversion** (Jonas's path after Aditi commits)
```
# Aditi: svn commit -m "README: add Quick Start"
# Jonas:
svn commit -m "README: rewrite Overview"    # rejected, out of date
svn update                                   # auto-merges
svn commit -m "README: rewrite Overview"
```

**Perforce**
```
# Aditi: p4 edit README.md; p4 submit -d "..."
# Jonas:
p4 edit README.md
p4 submit -d "README: rewrite Overview"      # blocked, must resolve
p4 sync
p4 resolve -am
p4 submit -d "README: rewrite Overview"
```

**Mercurial**
```
# Jonas, after Aditi pushes:
hg push                                      # rejected, would create heads
hg pull
hg merge
hg commit -m "Merge Aditi's README changes"
hg push
```

**Git**
```
# Jonas, after Aditi pushes:
git push                                     # rejected
git pull --rebase
git push
```

**Pijul**
```
# Jonas, after Aditi pushes:
pijul push                                   # rejected
pijul pull
pijul push
```

## Operation 7 — Merge with conflict (currency-conversion → main)

**Subversion**
```
svn switch file:///var/svn/ledger/trunk
svn merge file:///var/svn/ledger/branches/currency-conversion
# resolve conflict in src/ledger/parser.py
svn resolve --accept=working src/ledger/parser.py
svn commit -m "Merge currency-conversion into trunk"
```

**Perforce**
```
p4 merge --from //depot/ledger/currency-conversion
p4 resolve
# resolve conflict in src/ledger/parser.py
p4 submit -d "Merge currency-conversion into main"
```

**Mercurial**
```
hg update default
hg merge currency-conversion
# resolve conflict in src/ledger/parser.py
hg resolve --mark src/ledger/parser.py
hg commit -m "Merge currency-conversion into main"
```

**Git**
```
git switch main
git merge currency-conversion
# resolve conflict in src/ledger/parser.py
git add src/ledger/parser.py
git commit -m "Merge currency-conversion into main"
```

**Pijul**
```
pijul switch main
pijul pull --from-channel currency-conversion
# resolve conflict in src/ledger/parser.py
pijul record -m "Merge currency-conversion: resolve parser conflict"
```

## Operation 8 — Botched commit (debug print, typo'd message; not yet shared)

**Subversion** (no amend; commit a follow-up)
```
vi src/ledger/report_register.py    # remove debug line
svn commit -m "register: remove stray debug print (typo in r17 message: --csv not --cvs)"
```

**Perforce** (description editable, content via follow-up)
```
p4 change -f -d "register: add --csv flag" 17
p4 edit src/ledger/report_register.py
vi src/ledger/report_register.py    # remove debug line
p4 submit -d "register: remove stray debug print from change 17"
```

**Mercurial** (amend pre-push)
```
vi src/ledger/report_register.py    # remove debug line
hg commit --amend -m "register: add --csv flag"
```

**Git** (amend pre-push)
```
vi src/ledger/report_register.py    # remove debug line
git add src/ledger/report_register.py
git commit --amend -m "register: add --csv flag"
```

**Pijul** (unrecord)
```
pijul unrecord
vi src/ledger/report_register.py    # remove debug line
pijul record -m "register: add --csv flag"
```

## Operation 9 — Tag (v0.1.0)

**Subversion**
```
svn copy file:///var/svn/ledger/trunk file:///var/svn/ledger/tags/v0.1.0 -m "Tag v0.1.0"
```

**Perforce**
```
p4 label -t static v0.1.0
p4 labelsync -l v0.1.0 //depot/ledger/main/...
```

**Mercurial**
```
hg tag v0.1.0
```

**Git**
```
git tag -a v0.1.0 -m "First public release"
git push origin v0.1.0
```

**Pijul**
```
pijul tag create v0.1.0 -m "First public release"
```

## Operation 10 — Release (tarball)

**Subversion**
```
svn export file:///var/svn/ledger/tags/v0.1.0 ledger-0.1.0
tar czf ledger-0.1.0.tar.gz ledger-0.1.0
```

**Perforce**
```
p4 sync //depot/ledger/main/...@v0.1.0
rsync -a --exclude '.p4*' /workspace/ledger/ /tmp/ledger-0.1.0/
tar czf ledger-0.1.0.tar.gz -C /tmp ledger-0.1.0
```

**Mercurial**
```
hg archive -r v0.1.0 ledger-0.1.0.tar.gz
```

**Git**
```
git archive --format=tar.gz --prefix=ledger-0.1.0/ v0.1.0 -o ledger-0.1.0.tar.gz
```

**Pijul**
```
pijul archive --tag v0.1.0 -o ledger-0.1.0.tar.gz
```

## What the side-by-side reveals

Several patterns emerge from the comparison.

*The post-staging-area systems converge.* Mercurial, Git, and Pijul produce nearly identical command shapes for most operations. The differences are in vocabulary, not in workflow.

*Subversion's branch-as-directory-copy stays distinctive.* The `svn copy` for branches and tags is the visible signature of the centralized model.

*Perforce's edit-before-modify rhythm is unique.* The `p4 edit` step that does not exist in any DVCS is the visible signature of the locking model.

*Pijul's record/unrecord vocabulary maps cleanly.* Where Git uses `commit --amend`, Pijul uses `unrecord` followed by re-recording. The model difference (patches with identity vs. snapshot replacement) is encoded in the verb choice.

*The release operation is uniformly under-specified.* Every system has an `archive` or equivalent, but none of them includes the release notes, the publication, or the artifact promotion. That part is the team's, in every system.

The systems differ more in *cultural defaults* than in *commands*. Subversion's culture is "commit once, never rewrite"; Mercurial's is "commit local, rewrite acceptably, push when clean"; Git's is "rewrite freely, push linearly"; Perforce's is "atomic submit with full audit"; Pijul's is "patches commute, structurally". The commands are almost the same; the workflows the commands inhabit are quite different.
