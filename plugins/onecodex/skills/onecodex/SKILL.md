---
name: onecodex
description: Working with the One Codex platform — both the `onecodex` Python client (Samples, Classifications, Analyses, querying, plotting) and authoring/running custom Workflows (shell_script and nextflow jobs, runtime contract, dependencies, assets). Use when writing Python against the One Codex API, or when building/debugging per-sample pipelines on the platform.
---

# One Codex

One Codex is a microbial-genomics platform: upload sequencing samples, run classifications and other analyses against them, store and query results via an API, and run custom per-sample pipelines (Workflows) in the platform's compute environment.

This skill has two halves:

- [Python client](#python-client) — primary API for fetching samples, running analyses, querying results, and plotting.
- [Workflows](#workflows) — authoring, registering, and debugging custom per-sample pipelines (`shell_script` and `nextflow` job types).

## Python client

Quick reference for the `onecodex` Python library.

### Install & auth

```bash
pip install onecodex          # core
pip install 'onecodex[all]'   # + altair, pandas, scikit-bio, etc. (for analysis/plotting)
onecodex login                # one-time; stores API key in ~/.onecodex
```

Prebuilt Docker image with the client + `all` and `reports` extras installed (based on `python:3.13-slim-bullseye`): `quay.io/refgenomics/docker-onecodex-notebook`. Use it instead of building your own when you need a ready-to-run env for notebooks, scripts, or `shell_script` Jobs that just call the client.

Source: <https://github.com/onecodex/onecodex>. Auth via `onecodex login`, env var `ONE_CODEX_API_KEY`, or `Api(api_key=…)`.

```python
import onecodex
ocx = onecodex.Api()
```

### Resources & Access

Standard: `Samples Classifications Analyses Projects Tags Documents Users Jobs Alignments FunctionalProfiles Panels`. Experimental (need `experimental=True`): `Assets Genomes Assemblies AnnotationSets Taxa`.

```python
ocx.Samples.get("id")
ocx.Samples.where(...)          # SampleCollection; fetches ALL pages by default
ocx.Projects.all()              # The preferred way to fetch all
sample.filename | .created_at | .size | .metadata | .project | .tags | .primary_classification
classification.results()        # {'n_reads', 'table', ...}
analysis.analysis_type          # Returns type (str) of analysis (e.g., classification, report)
```

### Querying

```python
ocx.Samples.get("id") # fetch a single sample by ID
ocx.Samples.where(limit=50)
ocx.Samples.all() # prefer .all() over .where() for fetching all instances of a model
ocx.Samples.where(project=project_or_id)
ocx.Samples.where(tags=["trimmed"])                 # has tag (name | id | Tag obj)
ocx.Samples.where(filename={"$icontains": "patient"})
ocx.Samples.where("size": {"$gte": 1_000_000}})
from datetime import datetime, timedelta, timezone
since = (datetime.now(timezone.utc) - timedelta(days=30)).isoformat()
ocx.Samples.where(updated_at={"$gte": since})       # RFC3339 w/ tz offset required
```

Operators: `$eq $ne` (scalar/ref/None → IS [NOT] NULL), `$lt $lte $gt $gte` (num/datetime), `$between [a,b]` (inclusive), `$in [..]` (scalars/refs), `$contains $icontains $startswith $istartswith $endswith $iendswith` (str), `$containsall $containsany` (list-ref fields only). Single-ref fields → `$eq/$ne/$in`; list-ref `[Model]` → `$containsall/$containsany`.

Client-side (when no server operator exists):

```python
[s for s in ocx.Samples.where(public=True, limit=100) if "bad" not in s.filename.lower()]
[s for s in ocx.Samples.where() if not any(t.name == "trimmed" for t in s.tags)]

# metadata search by API is not possible: filter locally after retrieving broader samples
samples.filter(lambda s: s.metadata.get("sample_type") == "clinical")  # SampleCollection only
```

Bulk = fast (server-side, paged, relations included). Per-id loops = slow.

### Fields

- **Samples** (filterable): `created_at updated_at? filename? size? status visibility error_msg? metadata→Metadata owner→Users primary_classification→Classifications? project→Projects? tags→[Tags]`
  - IMPORTANT: `sample.primary_classification` is the PREFERRED WAY to get a Classification for a given sample
- **Metadata** (own samples only): `sample→Samples starred updated_at? date_collected? date_sequenced? description? external_sample_id? library_type? location_lat? location_lon? location_string? name? platform? sample_type? custom(dict, NOT filterable)`
- **Classifications/Alignments/FunctionalProfiles/Panels**: `created_at complete draft success? error_msg? job→Jobs sample→Samples dependencies→[Analyses] job_args/cost(NOT filterable)`. **Analyses** adds `analysis_type`.
- **Projects**: `name? project_name? description? external_id? public owner→Users permissions(NOT filterable)`
- **Tags** `name` · **Users** `email` · **Jobs** `created_at name analysis_type public` · **Documents** `created_at filename size? uploader→Users downloaders→[Users]`

### Upload

```python
ocx.Samples.create("s.fastq.gz", metadata={...})
ocx.Samples.create_paired("R1.fastq.gz", "R2.fastq.gz", metadata={...})
```

### CLI-only utilities

These CLI commands have no Python-client equivalent. (Job/analysis CLI commands are in [Workflows](#cli-commands).)

```bash
onecodex download samples OUTDIR [--project … | -t TAG | -s SAMPLE_ID …]   # bulk FASTX download
onecodex documents {list,download,upload}                                  # Document Portal
```

`onecodex scripts`:

```bash
# Extract reads assigned to a taxon from a classification (host removal, taxon pull-out, etc.)
onecodex scripts subset_reads <CLASSIFICATION_ID> reads.fastq.gz -t 562 -o ecoli.fastq.gz
#   -t TAX_ID (repeatable, required) · --with-children (incl. strains) · --exclude-reads (keep non-matching)
#   -r R2.fastq.gz (paired) · --include-lowconf · default skips low-confidence assignments

# Interleave paired FASTQs (counterpart to the workflow `deinterleave`); streams to stdout w/o OUTPUT arg
onecodex scripts interleave R1.fastq.gz R2.fastq.gz out.fastq.gz --gzip   # inputs must be sorted

# Bulk functional-profile export to one CSV (long default; --table-format wide)
onecodex scripts export_functional_metric -a ko -m cpm -o out.csv {-p PROJECT | -s ID,ID,…}
#   -a: reaction|ec|metacyc|ko|pathways|eggnog|pfam|go
#   -m: completeabundance|coverage|abundance (pathways) · rpk|cpm (other annotations)
```

### Enums

All live in `onecodex.lib.enums`, are `str` subclasses, and have rich docstrings — inspect them for the full set. Highlights:

```python
from onecodex.lib.enums import Rank, Metric, AlphaDiversityMetric, BetaDiversityMetric
Rank.{Auto,Superkingdom,Kingdom,Phylum,Class,Order,Family,Genus,Species}  # rank=
```

**Default `Metric` values used by the client (pass as `metric=`):**

| Member | Value | Notes |
|---|---|---|
| `Metric.ReadcountWChildren` | `"readcount_w_children"` | Default count metric — readcount for a taxon plus all taxonomic descendants. int. |
| `Metric.Abundance` | `"abundance"` | Default when classifier abundance is available — genome-size-adjusted, computed at species, sums to 1.0, not always available per-species. float. |
| `Metric.NormalizedReadcountWChildren` | `"normalized_readcount_w_children"` | Default normalized metric — `ReadcountWChildren` / Σ `ReadcountWChildren` at rank. Sums to 1.0. float. |

Other members exist (`Readcount`, `Prop*`, `Unfiltered*`, …) — read the `Metric` docstring if a task needs them. ⚠ Filtered-readcount metrics (`metric.is_filtered_readcount_metric`) are computed differently by the API depending on whether a sample has abundance data, so they are **not directly comparable across samples with mixed abundance status** — prefer the `Unfiltered*` family of metrics.

**Diversity metrics** (pass as `diversity_metric=`):

- `AlphaDiversityMetric`: `Chao1`, `ObservedTaxa`, `Shannon`, `Simpson`
- `BetaDiversityMetric`: `Aitchison`, `BrayCurtis`, `Jaccard`, `CityBlock`, `Manhattan`, `Euclidean`, `UnweightedUnifrac`, `WeightedUnifrac`

**Stats:**

- `AlphaDiversityStatsTest`: `Auto`, `Wilcoxon`, `Mannwhitneyu`, `Kruskal`
- `BetaDiversityStatsTest`: `Permanova`
- `PosthocStatsTest`: `Dunn`, `PairwisePermanova`
- `AdjustmentMethod`: `BenjaminiHochberg`, `HolmBonferroni`

### Assets (client)

```python
ocx.Assets.upload("f.txt", name="..."); asset.download(...); asset.save(); asset.delete(); Assets.uploaded_by→Users
```

### Analysis (SampleCollection)

```python
df = samples.to_classification_df(rank=Rank.Species, metric=Metric.Readcount,
                                  top_n=20, table_format="long")  # default wide
# wide: rows=samples (idx=classification_id), cols=tax_id, values=abundance
df.ocx_metadata          # metadata DataFrame      df.ocx_taxonomy   # taxonomy info
samples.taxonomy.loc[tax_id, "name"]               # tax_id-indexed lookup
samples.alpha_diversity(rank=Rank.Species, diversity_metric=AlphaDiversityMetric.Shannon)
samples.beta_diversity(rank=Rank.Species, diversity_metric=BetaDiversityMetric.BrayCurtis)
```

### Plotting (Altair; `return_chart=True` → chart object)

```python
samples.plot_pca(rank=, metric=, title=)
samples.plot_bargraph(rank=, top_n=, metric=, title=)
samples.plot_heatmap(rank=, top_n=, title=)
samples.plot_distance(rank=, diversity_metric=, title=)
chart.save("out.png", scale_factor=2.0)            # or .svg / .html
```

### Experimental Api(experimental=True)

```python
ocx.Genomes.all(); genome.assemblies; genome.primary_assembly; genome.taxon
ocx.Assemblies.get("id").download("a.fasta")
ocx.AnnotationSets.get("id").download("a.gbk"); .download_csv("a.csv")
ocx.Taxa.where(taxon_id="821")[0]; taxon.genomes(); taxon.parents()
```

Relations: `Assemblies.{owner, genome?, input_samples?, job?, primary_annotation_set?}` · `AnnotationSets.{assembly, job?}` · `Genomes.{assemblies, primary_assembly?, tags, taxon}` · `Taxa.{taxon_id, name?, rank?, parent?(self)}`

### Included Dependencies

These packages are installed with `onecodex[all]` and may be used in conjunction with the library itself: altair, numpy, openpyxl, pandas, scikit-bio, scikit-learn, scikit-posthocs, scipy

## Workflows

Author, register, run, and debug per-sample pipelines on the One Codex platform. Covers what isn't in the Python client docs above: the runtime contract your code runs under, and the iteration loop around it.

### Data model

- **Sample** — a sequencing file uploaded to OCX. FASTA or FASTQ (.gz). Has a UUID, a filename, an owner, optional `metadata.*`. The unit of input to a workflow.
- **Job** — a registered, runnable workflow definition. Two job types: `shell_script` (one bash script in a container of your choice) and `nextflow` (a Nextflow pipeline cloned from a git repo). Owned by a user; can be private (`public=False`) or published to an org. Has an immutable `job_type`, a mutable `script`/`image_uri`/`assets`/ `arguments_schema`/etc. Identified by a UUID.
- **Analysis** — one run of a Job against one Sample. Created via `Jobs.run(sample, job_args=…)`. Has its own UUID, `complete`/`success` flags, `logs()`, and `get_files()`. Per-run state lives here.
- **Workflow** — the user-facing term for a custom Job. `analysis_type` is literally `workflow` for jobs you create through this API.
- **Arguments** (`job_args`) — a JSON dict passed at run time. OCX injects each `key=value` into the container as both an env var (`$ARGS_<KEY>`) and an entry in `/out/input_params.json` (for Nextflow pipelines). The set of allowed keys is declared by the Job's `arguments_schema`.
- **Dependencies** — a Job can declare that it requires the output of another Job (typically a parent that produces intermediate files). When a child runs, OCX automatically runs the parent first (or reuses a prior parent analysis) and stages its outputs into a subdirectory of `/out`. See [Dependencies](#dependencies).
- **Assets** — files (typically reference DBs) that get mounted into the container under `/share/asset_<UUID>/<filename>`. Independent of the per-run staged sample. See [Assets](#assets).
- **Results** — the platform tarballs the contents of any directory whose name starts with `output` (or just CWD for shell_script jobs) after a successful run, surfaces files in the UI, and exposes a `results.json` (if you write one to CWD) via `/api/v1/analyses/<id>/ results`.

### Mental model

A workflow on OCX runs **once per sample**. The user picks a sample in the UI (or API), OCX spins up a container (with your script for shell_script jobs, or the orchestrator + your cloned pipeline for nextflow jobs), stages the sample on disk, runs to completion, captures outputs. Cross-sample analysis isn't a thing — you either redesign as per-sample + a downstream aggregation step, or run the aggregation out-of-band.

### `shell_script` vs `nextflow`

Pick `shell_script` when you can express the work as one bash script that takes one sample. You control the container, the deps, the runtime shape entirely. Required at create time: `--cpu`, `--ram-gb`, `--storage-gb`. Optional: `--repository-url`/`--repository-tag` to clone a git repo alongside the script (cloned into CWD; path in `$REPOSITORY_DIR`) — useful for shipping helper files (notebooks, configs, templates) versioned with the script.

Pick `nextflow` when you want to run an existing Nextflow pipeline (e.g. `nf-core/<x>`) and let its modules handle per-process containers, resource labels, and parallelism. OCX provides Nextflow + Docker-in-Docker in a pinned orchestrator image; you write a thin entrypoint that builds a samplesheet for one sample and calls `nextflow run`. Per-process resources are set by the pipeline, not by the Job — you cannot pass `--cpu`/`--ram-gb`/`--storage-gb`.

### The API and One Codex CLI

See the [Python client](#python-client) section above for the full client and resource model (Samples, Analyses, Assets, Jobs). This section focuses on the workflow-specific surface and the iteration loop.

#### CLI commands

```bash
onecodex jobs create  --name … --script … --image-uri … --job-type shell_script \
                      --cpu N --ram-gb N --storage-gb N
                      # → prints JOB_ID

onecodex jobs update <JOB_ID> --script …    # push new script/image/resources
onecodex jobs run    <JOB_ID> <SAMPLE_ID>   # kick off an analysis
        ↑ add --await to block until terminal state
        ↑ add -a key=value (repeatable) to pass job_args

onecodex analyses await <ANALYSIS_ID>       # block on a pre-started run
onecodex analyses logs  <ANALYSIS_ID>       # stdout/stderr of the script
```

#### Iteration loop

1. Write `jobscript.sh` (or `entrypoint.sh` for Nextflow).
2. Smoke-test locally — `docker run … bash jobscript.sh` with the same image and env vars the platform sets (see [Local iteration](#local-iteration)).
3. `onecodex jobs create …` once. Save the `JOB_ID`.
4. `onecodex jobs run <JOB_ID> <SAMPLE_ID> --await` against a tiny sample.
5. Failure? `onecodex analyses logs <ANALYSIS_ID>` (or `ocx.Analyses.get(id).logs()` for tail control). Fix.
6. `onecodex jobs update <JOB_ID> --script …` and re-run. Updates take effect on the next `run`; no redeploy.

A `make publish` (or equivalent) wrapping step 6 makes the inner loop one command.

#### Job lookup gotcha (Nextflow)

`Jobs.where(name=…)` does **not** find freshly-created custom Nextflow Jobs — they land in a draft state (`current=False`) and the listing endpoint filters drafts out by default. `Jobs.get(id)` works fine. **Cache the Job ID locally on create** (e.g. write to `.job_id`) and reuse via `Jobs.get(id)`. Publishing flips `current=True` but locks most fields — see [Best Practices](#best-practices).

### Runtime contract

#### Env vars (both job types)

| Variable | What it is |
|---|---|
| `OCX_SAMPLE_FILENAME` | Filename of the sample. CWD = `/out` (both job types); the staged sample is at `./$OCX_SAMPLE_FILENAME`. Read this env var, not `$1`. |
| `OCX_SAMPLE_NAME` | Human-readable sample name. |
| `OCX_SAMPLE_UUID` | Sample ID. |
| `OCX_ANALYSIS_UUID` | This run's analysis ID. Use in output dir names. |
| `OCX_INSTRUMENT_VENDOR` | Detected platform (`Illumina`, `Oxford Nanopore`, `PacBio`, …) or empty. |
| `OCX_IS_LONG_READ` | `"true"`/`"false"` — falls back to `"false"` if vendor unknown. |
| `OCX_DEPENDENCY_UUIDS` | Space-separated UUIDs for analyses staged as dependencies. |
| `$ARGS_<KEY>` | One per entry in the Job's `arguments_schema`, value from the per-run `job_args`. |

#### Filesystem layout

| Path | What's there |
|---|---|
| `/out/` (CWD, both job types) | The staged sample, generated `input_params.json`, `script.sh` (your entrypoint), orbiter runtime metadata. Write outputs here for shell_script (everything gets captured). |
| `output-$OCX_ANALYSIS_UUID/` (nextflow) | Where you must write outputs. Anything matching `output*` is captured. |
| `$REPOSITORY_DIR` (both job types, if `--repository-url` set) | The cloned git repo. **Relative path** under CWD (e.g. `Hello-World`), not absolute. For nextflow this is the pipeline; for shell_script it's whatever helper files you want versioned with the script. |
| `/share/asset_<UUID>/<filename>` | Mounted assets. See [Assets](#assets). |
| `/share/asset_<UUID>/` | Auto-extracted contents of `.tar.gz` assets. |
| `./results.json` (CWD) | If present after the run, surfaced via the `/results` API endpoint without a download. |

#### Paired-end FASTQs (interleaved by default)

OCX stages paired-end samples as a single **interleaved** FASTQ. Pipelines that want `R1`/`R2` must split it first. Use the `deinterleave` helper provided in the nextflow orchestrator image:

```bash
deinterleave "$OCX_SAMPLE_FILENAME"
# → ${SAMPLE_NAME}_R1.fastq.gz, ${SAMPLE_NAME}_R2.fastq.gz
#   where SAMPLE_NAME = $OCX_SAMPLE_FILENAME with extensions stripped

# Or pick the output names:
deinterleave "$OCX_SAMPLE_FILENAME" reads_R1.fastq.gz reads_R2.fastq.gz
```

Then reference those files in your `input.csv` / samplesheet. Long-read samples are not interleaved — gate on `$OCX_IS_LONG_READ`.

### Dependencies

A Job can declare it depends on another Job. When you run the child, OCX runs the parent first (or reuses an existing successful parent analysis on the same sample), then stages the parent's outputs into a subdirectory of `/out/` for the child to consume.

#### At Job creation

```python
ocx.Jobs.create(
    name="child",
    ...,
    dependencies=[{"job": parent_job, "output_dir": "parent_out"}],
)
```

At run time, the parent's output files appear under `/out/parent_out/…` in the child container.

#### Overriding for one run

`job.run(sample, dependency_overrides=[DependencyOverride(analysis=prior)])` forces the child to use a specific prior analysis instead of running the parent fresh. CLI: `onecodex jobs run … -d <analysis_id>` or `-d <analysis_id>=parent_out` to rename the staging dir.

#### Inspecting at runtime

`OCX_DEPENDENCY_UUIDS` is a space-separated list of parent analysis UUIDs that were staged. Useful for verifying a specific parent was used, or fetching extra metadata via `ocx.Analyses.get(uuid)`.

### Git repositories

Attach a git repo with `--repository-url <HTTPS_URL>` (optional `--repository-tag <annotated-tag>`; omit to track the default branch HEAD). Works for both `shell_script` and `nextflow` jobs — required for Nextflow, useful for shell_script when you want notebooks, config, or auxiliary scripts versioned alongside the workflow.

On every run, OCX clones the repo into CWD; the directory name is in `$REPOSITORY_DIR` (relative, e.g. `myrepo`). For shell_script, the common shape is a tiny "shim" script as the job's `--script` that `exec`s the real entrypoint from the repo:

```bash
#!/bin/bash
exec bash "$REPOSITORY_DIR/jobscript.sh"
```

After that, every workflow change is one `git push` away — no `jobs update`.

#### Private repos: GitHub integration

Cloning private repos needs OCX to be connected to GitHub. Set this up at <https://app.onecodex.com/settings> → **GitHub Integration**:

- **OCX GitHub App** (org-wide, recommended): an org owner/admin installs the app once on behalf of all members. If installed, this takes precedence and per-user PATs are disabled.
- **Personal Access Token** (per-user): each user supplies their own.

For the PAT route, the minimum fine-grained permissions are:

- **Repository access**: the specific private repo(s) to attach to jobs (not "all repos")
- **Repository permissions → Contents: Read-only**
- (Metadata: Read-only is auto-granted as a dependency)

That's all OCX needs — it clones, never pushes. For a classic PAT, `repo` scope works but is overprivileged; prefer fine-grained.

Common gotcha: a newly-created private repo isn't automatically covered by an existing fine-grained PAT. Edit the PAT in GitHub settings and add the new repo to its allowlist, otherwise `jobs create`/`update` fails with `ERROR: Unable to access repository`.

### Assets

Reference DBs, model files, anything you don't want to re-download on every run. Uploaded once, attached to a Job, mounted under `/share/`.

#### Uploading

```python
ocx = onecodex.Api(experimental=True)   # `Assets` is experimental
asset = ocx.Assets.upload("/path/to/file.tar.gz", name="my-db")
```

`Assets.upload()` takes a single file path. To upload a directory, tar it first.

#### Format choice

| Extension | OCX auto-extracts? | When to use |
|---|---|---|
| `.tar.gz` | **Yes**, into `/share/asset_<UUID>/` | Default for directory-shaped DBs. |
| `.tar.xz` / `.tar` / `.zip` | No | Avoid. Either re-pack as `.tar.gz` or extract yourself in the entrypoint. |
| Any other single file (`.hmm.gz`, `.fa`, etc.) | No | Lands at `/share/asset_<UUID>/<filename>` as-is. |

#### Attaching to a Job

```bash
onecodex jobs create  … --asset-id <ASSET_ID>      # at creation
onecodex jobs update <JOB_ID> --asset-id <ASSET_ID> # later, replaces the asset list
```

#### Resolving the path at runtime

The mount path includes the asset UUID, so glob it rather than hardcoding:

```bash
shopt -s nullglob
DB=""
for cand in /share/asset_*/db-light /share/asset_*/db; do
    [[ -d "$cand" ]] && DB="$cand" && break
done
shopt -u nullglob
```

#### When *not* to bother with an asset

Many tools/pipelines support downloading their DB on first run. For prototyping, the in-pipeline download is often the lazy win — skip the asset prep entirely, pay the download cost per run, promote to an asset when the cost matters. nf-core/bacmodel exposes `pfamdb_download=true` and `baktadb_download=true` for exactly this.

### Best practices

- **Read `OCX_SAMPLE_FILENAME`, not `$1`.** The sample isn't passed positionally.
- **Branch on `OCX_IS_LONG_READ`.** Don't infer from the filename; treat unknown samples as short-read (or fail loudly).
- **Set `HOME=/tmp` + redirect caches** for tools that scribble in `$HOME/.cache` — the container's home directory may be read-only:

  ```bash
  export HOME=/tmp PIXI_CACHE_DIR=/tmp/pixi-cache XDG_CACHE_HOME=/tmp/xdg
  mkdir -p "$PIXI_CACHE_DIR" "$XDG_CACHE_HOME"
  ```

- **Smoke-test locally first** with the same image and env vars (see [Local iteration](#local-iteration)). Iterating against the OCX runner adds minutes per round trip.
- **Cache the Job ID on create.** `Jobs.where(name=…)` doesn't find draft Jobs (see [Gotchas](#gotchas)). Persist to `.job_id` and use `Jobs.get(id)`.
- **Publish only when settled.** Drafts (the default) are fully runnable and freely mutable. Publishing (`POST /api/v2/jobs/<id>/ publish`) makes the Job visible in the org dropdowns but locks most fields and requires `repository.tag` to be a real git tag — so do it last, not first.
- **Keep `arguments_schema` curated.** It's a UI affordance, not the param surface. Don't restate every upstream pipeline param; expose only what users should control. Required upstream params (`input`, `outdir`) are handled by the entrypoint.

### Local iteration

Run the entrypoint in the same image locally to catch most issues before paying for an OCX run:

```bash
docker run --rm --platform linux/amd64 \
    --volume "$(pwd)/out":/out \
    --volume "$(pwd)/jobscript.sh":/usr/local/bin/jobscript.sh \
    --workdir /out \
    --env OCX_SAMPLE_FILENAME=mysample.fastq.gz \
    --env OCX_IS_LONG_READ=false \
    <your image> \
    bash /usr/local/bin/jobscript.sh
```

Drop the sample into `./out/` first. Flip `OCX_IS_LONG_READ` to exercise both branches. For Nextflow jobs, also mount `/share/` with some mocked asset directories.

### Gotchas

#### Both job types

- **Empty `logs()` + "We were unable to process this file."** in ~30–90 seconds = entrypoint crashed before its first stdout flush. Almost always `set -e` + a non-existent path or failed `ls`/glob. Add diagnostic `ls -la` at the top and use `shopt -s nullglob` for globs.
- **403 on `Jobs.run`** = you don't own the sample. You can only run Jobs against samples you own.
- **Jobs aren't deletable** via API (HTTP 405). Rename stale ones: `job.update(name="…-DELETED")`.
- **`ERROR: Unable to access repository`** on `jobs create`/`update` with `--repository-url` set = OCX can't clone the repo. Either (a) no GitHub integration configured for the user/org (see [Git repositories](#git-repositories)), or (b) a fine-grained PAT is installed but the target repo isn't on its allowlist (newly-created repos aren't auto-included — edit the PAT to add it).

#### `shell_script`-specific

- **`--job-type cannot be changed after job creation`.** Drop the `--job-type` flag from `jobs update` calls.
- **`cpu, ram_gb, storage_gb must be set for shell script jobs`.** Required on `jobs create`.

#### `nextflow`-specific

- **`cpu, ram_gb, storage_gb must not be set for Nextflow jobs`.** Resources are pipeline-defined per-process. To override pipeline defaults, ship a `nextflow.config` snippet via `-c` from the entrypoint.
- **`Unable to find tag in repository`** = `--tag` must be an annotated git tag, **not** a branch or commit SHA. Omit `--tag` entirely to track the default branch HEAD; fork-and-tag when the upstream has no tags.
- **`image_uri` is the OCX orchestrator image**, not an arbitrary Docker URI. Current pin: `976006418372.dkr.ecr.us-east-1.amazonaws.com/onecodex/orbiter-nextflow-orchestrator:v0.1.0-25100`.
- **`arguments_schema` is `list[FieldGroup]`**, not a flat field list: `[{title, description, fields: [{name, type, default, choices?, ...}]}]`. Ship a flat list and the server creates empty groups silently. Verify with `GET /api/v1/jobs/<id>` and check that `job_args_schema.definitions.<group>.properties` is populated. Field `type` values: `string`, `integer`, `number`, `boolean`, `enum` (needs `choices`), `regex` (needs `pattern`).
- **Asset symlinks break in S3-backed `workDir`.** E.g. Bakta's `amrfinderplus-db/latest -> 2024-12-18.1` survives OCX's `.tar.gz` auto-extract but breaks when Nextflow stages the DB into its S3 work dir. Workaround: `cp -rL "$DB" ./db_local` in the entrypoint and pass the local path so all files are real.
- **`updated_at` doesn't tick for hours after a run** = the orchestrator never scheduled the task. Almost always a resource-label mismatch — `process_high` is 12 CPU / 72 GB; if no worker matches, the task wedges silently with no logs and no error. There's no cancel endpoint. Drop the offending tool, or inject a config: `nextflow run … -c <(echo "process { withLabel:process_high { cpus = 8; memory = '32.GB' } }")`.
- **`Jobs.where(name=…)` returns `[]` for your own freshly-created Job.** Drafts are hidden from the listing endpoint (filtered by default `current=True`). `Jobs.get(id)` works. Cache the ID.
