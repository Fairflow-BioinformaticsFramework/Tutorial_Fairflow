# FairFlow Tutorial

This tutorial describes the **canonical FairFlow path**: from a bare analysis script to **audited, runnable, and reproducible artifacts** across **R, Python, Bash, and Galaxy**.

The workflow mirrors **Figure 1C** of the FairFlow paper and is organized into five steps.  
Minimal, copy‑pasteable snippets are included where helpful. A complete example can be adapted to any analysis; we refer to *“the pipeline”* generically.

---

## Step 0 — Understanding the Front‑End Function

In FairFlow, a pipeline interface is defined by a **`.bala` file**.

A `.bala` file declares:
- **what a tool needs** (parameters)
- **how it runs** (container image, volumes, arguments)
- **what it produces** (outputs)

The generated front end is a **thin, validated wrapper** that:
- mounts inputs deterministically
- executes the containerized script
- returns a well‑defined output directory

There is **no hidden state** and **no implicit behavior**.

---

## A complete `.bala` example (annotated)

```ini
[research]
description=Selects genes with the highest values according to a chosen metric (expression or variance) and returns only the top X from the count matrix.
name=topX
#
[run]
command=docker run --rm
script= Rscript /bin/top.R
image= repbioinfo/topxv2:1
usage= <matrixName> <format> <separator> <logged> <threshold> <type>
#
[directory]
name=workDir
description=Path to the working directory
flag=io
mount=/workdir
#
[directory]
name=data
description=Path to the folder containing input data and receiving output results
mount=/data
flag=io
#
[parameter]
name=matrixName
description=Input file name without extension
value=annotated
#
[parameter]
name=format
values=csv, txt
description=Input file format
#
[parameter]
name=threshold
description=Threshold for selecting top genes (typically between 10 and 2000 depending on dataset size)
value=10
#
[parameter]
name=separator
values=',','\t'
description=File separator (use "," for CSV, "\t" for TSV)
#
[parameter]
name=logged
values=FALSE,TRUE
description=Indicates whether the count matrix values are already log-transformed (TRUE) or not (FALSE).
#
[parameter]
name=type
values=expression, variance
description=Type of analysis to perform.
```

## Step 1 — Build an immutable container (environment)

Hand‑crafted Dockerfiles are brittle for long‑term reproducibility: transitive dependencies drift, network mirrors change, and `latest` tags silently update.

FairFlow uses **CREDO (Customizable, REproducible DOcker)** to create **time‑stable environments** by recording installation actions and pinning artifacts into a frozen snapshot.

### Two practical consequences

- **Determinism**: the same CREDO script replays to the same resolved environment (no hidden `apt upgrade` or conda solver drift)  
- **Portability & auditability**: the frozen environment can be copied across images, hashed, archived (e.g., TAR with a DOI), and inspected post‑hoc

### Two‑stage build (recommended)

We use a two‑stage process (implemented in the helper repo `Fairflow-BioinformaticsFramework/DockerBuilder`):

1. **Stage 1 (builder)**: run CREDO install commands to assemble the environment under `/credo_env`, then `credo save` to freeze it  
2. **Stage 2 (runtime)**: start from a clean CREDO base, copy frozen `/credo_env`, then `credo apply` to materialize the exact same environment with **no online resolution**

This keeps the runtime image lean and fully reproducible.

### Example repository layout

```text
.
├── Dockerfile.stage1        # builds the environment (from CREDO base)
├── Dockerfile.final         # applies the frozen environment for runtime
├── build_all.sh             # orchestrates the two-stage build
├── credo_install.sh         # reads requirements.txt and runs CREDO commands
└── requirements.txt         # declarative list of tools (CREDO dialect)
```

### Example `requirements.txt` (CREDO dialect)

```text
apt build-essential
apt curl
conda python==3.11
pip jupyterlab==4.2.5
pip pandas==2.2.2
pip numpy==2.0.1
pip scikit-learn==1.5.1
pip matplotlib==3.9.1
pip umap-learn==0.5.6
cran Seurat
github fastai/fastcore@1.7.12
```

In short: for a reproducible environment, users typically write `requirements.txt` and run `build_all.sh` (macOS/Linux). On Windows, a dedicated Docker container can be used to run the same scripts.

---

## Step 2 — Write and test the script (FairFlow coding guidelines)

Once the environment is frozen (Step 1), write the analysis script that runs **inside** the container.
This script is the boundary between the immutable environment and the generated front end.

Guidelines:

- Clearly define all inputs and outputs; avoid hidden dependencies  
- Use absolute, fixed paths inside the container (avoid relative paths / `$PWD`)  
- Never assume host‑specific directories or user names  
- Handle errors explicitly and exit with non‑zero codes on failure  
- Treat mounted inputs as read‑only; write results to a dedicated output directory  

### Conventional container layout

```text
/scratch/    → working data (mounted volume)
/genome/     → reference or configuration files (mounted volume, usually read-only)
/output/     → persistent results
```

Validate required inputs early (example in R):

```r
if (!file.exists(matrix_file)) {
  stop(paste("Input matrix not found:", matrix_file))
}
```

If the pipeline uses large static resources (e.g., genomes), mount them externally under `/genome` rather than embedding them in the image.
Write persistent outputs under `/output` and keep temporary files in `/scratch`.

---

## Step 3 — Define the interface with Baryon‑lang

**Baryon‑lang** is a small DSL based on the **`.bala`** format to define and validate front ends in a **declarative** and **portable** way.

Each `.bala` describes:
- inputs and parameters (typed, documented)
- container execution (image, script, arguments)
- expected outputs

By construction, `.bala` files are:
- **self‑describing**
- **validatable**
- **transpilable** to R / Python / Bash and workflow engines (e.g., Nextflow, Galaxy, StreamFlow)

### Key principles

- **Declarative**: define what the tool is, not hand‑rolled glue code  
- **Portable**: same spec can target multiple back ends  
- **Self‑contained**: parameters, volumes, outputs are explicit  
- **Reproducible**: consistent execution and transparent validation  

### Best practices

- Always define explicit types and clear `(description ...)` strings  
- Use a shared `scratch_directory` for multi‑file workflows and mount it to `/scratch`  
- Keep `(arguments ...)` explicit and ordered (match the script’s CLI order)  
- Document outputs under `(outputs ...)` with meaningful names and descriptions  
- Avoid hidden defaults: no implicit paths, env vars, or assumptions  

---

## Step 4 — Validate and generate a front end

Clone Baryon sourcecode (Linux/macOS; on Windows use Git bash):

```bash
git clone https://github.com/Fairflow-BioinformaticsFramework/baryon-lang.git
cd baryon-lang
```

Generate wrappers:

```bash
python baryon.py myprogram.bala -l r
python baryon.py myprogram.bala -lang python

See [baryonlang](https://github.com/Fairflow-BioinformaticsFramework/Baryonlang) for more information on parameters.
```

Supported targets include: `r`, `python`, `bash`, `nextflow`, `galaxy`, `streamflow`.

The generated wrapper contains:
- callable function with typed arguments
- validation + error handling
- reproducible Docker invocation from the `run` block

---

## Step 5 — Optional: Galaxy front end (FairFlow‑compatible Galaxy + Lemaitre)

Galaxy is widely used for point‑and‑click bioinformatics, but extending it with new tools is traditionally cumbersome.

FairFlow provides:
- a **Baryon‑compatible immutable Galaxy distribution**: `Fairflow-BioinformaticsFramework/galaxy-formed`  
- an alternative Ansible installer: `Fairflow-BioinformaticsFramework/galaxy-formation`  

Recommended (Dockerized Galaxy):

```bash
git clone https://github.com/Fairflow-BioinformaticsFramework/galaxy-formed.git
cd galaxy-formed
docker-compose up -d
```

Galaxy will be available at:

```text
http://localhost:8080
```

Stop:

```bash
docker-compose down
```

Remove persistent volumes (destructive):

```bash
docker-compose down -v
```

### Lemaitre: drag‑and‑drop tool installation

To automate registration of FairFlow‑generated Galaxy tools, we bundle **Lemaitre**, a lightweight service available at:

```text
http://localhost:8000
```

Upload a FairFlow‑generated Galaxy wrapper and it becomes available immediately—no manual XML editing and no server restart.

---



## Links

- FairFlow organization: https://github.com/Fairflow-BioinformaticsFramework  
- Baryon lang: https://github.com/Fairflow-BioinformaticsFramework/baryonlang  
- Galaxy distribution (immutable): https://github.com/Fairflow-BioinformaticsFramework/galaxy-formed  
- Lemaitre (bundled with Galaxy‑formed): http://localhost:8000
