# 23. DVC

## Origin

DVC — *Data Version Control* — was started by Dmitry Petrov and Ivan Shcheklein in 2017 at the company that became Iterative.ai. Petrov had been working in machine learning at Microsoft and had encountered the problem the system addresses: ML projects produce experiments, the experiments produce models, the models depend on specific code, specific hyperparameters, and specific data, and reproducing a result from six months ago requires assembling all of those pieces in their correct historical state.

Git handles the code part. It does not handle the data part — gigabyte CSVs, multi-gigabyte image archives, model checkpoints — and it does not handle the *pipeline* part: the declarative description of how data flows through code to produce models. DVC was designed to layer over Git and provide both: data versioning and pipeline versioning, with the resulting artifacts (models, metrics) themselves tracked.

The system has grown over the years to include experiment tracking, metric comparison, plot generation, and a hosted dashboard product (Studio). The open-source DVC tool remains the foundation; Iterative's commercial offerings layer on top. The user base is concentrated in ML and data science teams that want to bring rigor to a domain that has historically been ad-hoc.

## The system on its own terms

DVC is a Python tool that runs alongside Git. A repository is a Git repository with DVC enabled (`dvc init`). The DVC-specific state lives in `.dvc/` and in `.dvc` pointer files placed alongside data files in the working tree.

For *data versioning*, the model is similar to LFS: data files are tracked by content hash, the hashes are recorded in `.dvc` pointer files in Git, the actual content lives on a *remote* (typically S3, GCS, Azure Blob, or any storage with the appropriate protocol). On `dvc push`, content goes to the remote; on `dvc pull`, content comes from the remote; the Git repository contains only the pointer files.

For *pipeline versioning*, DVC introduces `dvc.yaml`: a declarative description of stages in a data pipeline. A stage has a command, declared inputs (data files, code files, parameters), and declared outputs (data files, model files, metrics files). DVC computes a graph of stages; running `dvc repro` walks the graph, executes stages whose inputs have changed, and produces consistent outputs.

The pipeline file makes the *experiment* — the combination of code, data, and parameters that produced a model — explicit and version-controllable. Two team members running `dvc repro` against the same Git commit and the same DVC remote get bit-identical results, modulo non-determinism in the underlying training code.

For *experiment tracking*, DVC layers an *experiment* concept over Git. `dvc exp run` records a run with parameter values, metrics, and outputs; `dvc exp show` displays a table comparing runs; `dvc exp diff` shows what changed between runs. Experiments are stored as Git references in `refs/exps/` (an internal namespace), so they are versioned alongside code without cluttering the visible branch list.

The CLI is `dvc`. Subcommands: `dvc init`, `dvc add`, `dvc remote`, `dvc push`, `dvc pull`, `dvc run`, `dvc repro`, `dvc dag`, `dvc metrics`, `dvc params`, `dvc plots`, `dvc exp run`, `dvc exp show`, `dvc gc`, `dvc fetch`, `dvc import`. The grammar is regular and the help is good.

## Scenario walkthrough

The exemplar's `ledger` project is not an ML project, so the exemplar maps onto DVC awkwardly. The chapter renders the operations that make sense (initial import, linear development, branch, rename, parallel edits, conflict, botched commit, tag, release) using DVC's data-and-pipeline mode for the binary file (the logo) and notes where DVC adds value beyond the standard Git case.

For an ML-shaped scenario — training data, a feature engineering stage, a training stage, a model artifact, evaluation metrics — DVC's value is much more visible. Readers interested in DVC for ML should consult Iterative's documentation; the scenario here treats DVC as a general data-versioning layer.

### Operation 1 — Initial import

```
$ cd /path/to/initial/ledger
$ git init
$ dvc init
$ git add .dvc .dvcignore
$ git commit -m "Initial DVC config"
$ git add .
$ git commit -m "Initial import"
```

`dvc init` creates `.dvc/` with the initial DVC configuration. The directory is committed to Git as part of the project's tracked state.

### Operation 2 — Linear development

Standard Git for code. For data files (none in the early operations), DVC commands would track them.

### Operation 3 — Branch

Standard Git branching. DVC's data tracking follows the branch: each branch sees the data files relevant to its commits.

### Operation 4 — File rename

Standard Git rename. DVC's data files (if any) are renamed via `dvc move` to keep DVC's metadata consistent.

### Operation 5 — Binary file added

The logo is a binary asset. To track it with DVC:

```
$ dvc remote add origin s3://my-bucket/ledger-data
$ dvc add docs/logo.png
... creates docs/logo.png.dvc with the file's hash ...
... moves docs/logo.png into .dvc/cache/ and creates a working-tree copy ...
$ git add docs/logo.png.dvc docs/.gitignore
$ git commit -m "docs: add project logo (DVC-tracked)"
$ dvc push    # uploads the file to S3
```

The Git repository now tracks `docs/logo.png.dvc` (the pointer file) and `docs/.gitignore` (which excludes the actual `logo.png` from Git). Other developers run `dvc pull` to retrieve the actual file.

### Operations 6–10

Standard Git for code-side operations. Data-side: any committed data file is accompanied by a `.dvc` pointer file; pulls and pushes synchronize both Git and DVC remotes; merges of branches with diverging data files are merge of the pointer files (with conflict on the hash if the same file diverges).

DVC's value beyond Git becomes visible in *pipelines and experiments*. For a project with a `dvc.yaml`:

```
stages:
  parse_journal:
    cmd: python src/ledger/preprocess.py raw_data/journal.txt -o processed/parsed.json
    deps:
      - raw_data/journal.txt
      - src/ledger/preprocess.py
    outs:
      - processed/parsed.json
  build_index:
    cmd: python src/ledger/build_index.py processed/parsed.json -o processed/accounts.idx
    deps:
      - processed/parsed.json
      - src/ledger/build_index.py
    outs:
      - processed/accounts.idx
```

`dvc repro` walks the stages, runs only those whose inputs have changed, and produces the outputs. The outputs are tracked by DVC; their contents go to the DVC remote on push.

For experiments — varying parameters, training models, comparing metrics — `dvc exp run --set-param X=Y` records a parameterized run, with metrics captured in DVC's experiment metadata.

## Model and mental load

What you have to hold:

- Two parallel namespaces: Git tracks code and pointer files; DVC tracks data content via remotes.
- The `.dvc` pointer file convention.
- The pipeline grammar: stages, deps, outs, params.
- Experiments as a Git-references-with-metrics layer.

The mental load is moderate. DVC users typically know Git well and add DVC's vocabulary on top; the cost is real but tractable. New ML practitioners learn both at once and find the combined model more useful than either alone.

## Evolution and history rewriting

DVC inherits Git's history model for code and adds its own for data. Data history rewriting is constrained by the cache: removing a file from history without removing it from the cache is straightforward; full removal requires `dvc gc` operations on the cache and rewriting the Git pointers.

## Ecosystem reality

DVC is alive, well-funded (Iterative has commercial offerings), and growing in ML circles. The open-source tool is current; releases are frequent. The Studio product (hosted dashboards, experiment comparison) is sold to teams. The community is substantial and largely on the ML/data-science axis of programming culture.

The product has competition: MLflow (Databricks), Weights & Biases (Wandb), ClearML, Pachyderm, and various hosted notebook platforms. DVC is the most Git-native of these; teams that want their ML lifecycle to live in Git tend toward DVC.

## When to reach for it; when not to

Reach for DVC when:

- You are doing ML or data-science work where the data is part of the artifact and changes over time.
- You want experiments to be reproducible from Git commits.
- You want pipeline declarations as version-controlled artifacts.
- The team is Git-fluent and wants to extend Git's model into data.

Do not reach for DVC when:

- The project is pure code with no data dependencies.
- The team's data lives in databases or data warehouses, not in files. DVC is file-oriented; warehouse versioning needs different tools (Delta Lake, Iceberg, dbt with version control on SQL).
- The ML lifecycle is already on a different platform (Vertex AI, SageMaker, Databricks) that handles its own versioning.

## Epitaph

DVC is the version control system for the artifacts Git was not designed for, and is the proof that "the code repository" and "the project's reproducible artifact" are not the same thing once data and pipelines enter the picture.
