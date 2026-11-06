# BDBV Phylodynamics Pipeline (Part 2)

BEAST XML generation, postprocessing, and analysis for Bundibugyo ebolavirus phylodynamics. Picks up where `bdbv-pipeline` (Part 1) leaves off — takes a final alignment and ML tree as input.

## Setup

All commands use the `bdbv` conda environment:

```bash
mamba run --live-stream -n bdbv python bin/run.py ...
```

[TreeAnnotator](https://beast.community/treeannotator) must be available on `PATH` for MCC tree generation.

## Usage

### `run.py` — single entry point

Generate BEAST XMLs from a pipeline 1 output:

```bash
mamba run --live-stream -n bdbv python bin/run.py \
    --alignment /path/to/BDBV2026.drc.ADAR_masked.final.aln \
    --outdir output/20261106 \
    --prefix BDBV2026.drc \
    --template-sg templates/GTR_SG_MAT.template.xml \
    --anchor-date 2025-11-08 \
    --stop-after generate_xmls
```

Postprocess BEAST output (logs + trees):

```bash
mamba run --live-stream -n bdbv python bin/run.py \
    --beast-dir /path/to/beast/output \
    --outdir output/20261106 \
    --prefix BDBV2026.drc \
    --burnin 0.1 \
    --skip-to postprocess_logs \
    --stop-after postprocess_trees
```

### Steps

| Step | Name | Description |
|------|------|-------------|
| 1 | `generate_xmls` | Generate BEAST XMLs from alignment + templates |
| 2 | `postprocess_logs` | Combine chain logs, remove burnin, check ESS |
| 3 | `postprocess_trees` | Burnin-remove trees, resample, generate MCC |
| 4 | `generate_figures` | *(TODO)* |
| 5 | `generate_report` | *(TODO)* |

Use `--skip-to <step>` and `--stop-after <step>` to run specific steps.

### ESS gate

Step 2 checks ESS for key parameters (posterior, likelihood, root height, clock rate, SkyGrid parameters, etc.). If any key parameter has ESS below the threshold (default 200), the pipeline stops and reports which parameters are under-sampled.

Options when ESS is too low:
1. Run more BEAST chains and re-run postprocessing
2. Pass `--force-low-ess` to continue anyway
3. Re-enter at step 3: `--skip-to postprocess_trees --force-low-ess`

### Key flags

| Flag | Description |
|------|-------------|
| `--alignment` | Final alignment from pipeline 1 (required for step 1) |
| `--beast-dir` | Directory with BEAST output (required for steps 2-3) |
| `--outdir` | Output directory |
| `--prefix` | Output file prefix |
| `--template-sg` | SkyGrid template XML |
| `--template-egc` | EGC template XML |
| `--template-mat` | MAT SkyGrid template XML |
| `--anchor-date` | SkyGrid anchor date (default: `2025-11-08`) |
| `--chain-length` | MCMC chain length (default: 500M) |
| `--log-every` | Log interval (default: 50000) |
| `--burnin` | Burnin fraction (default: 0.1) |
| `--resample` | Target tree count after resampling (default: 1000) |
| `--min-ess` | Minimum ESS for key parameters (default: 200) |
| `--force-low-ess` | Continue past ESS check despite low values |
| `--linelist` | Genomic linelist Excel (for figures) |
| `--sitrep` | Sitrep CSV (for figures) |
| `--case-data` | status_aggregated CSV (for figures) |
| `--skip-to` | Skip to a specific step |
| `--stop-after` | Stop after a specific step |

## Individual scripts

### `generate_beast_xml.py`

Generate BEAST XML files from templates. Swaps in alignment, updates SkyGrid parameters, adjusts operator weights.

```bash
mamba run --live-stream -n bdbv python bin/generate_beast_xml.py \
    -a /path/to/final.aln \
    -o output/ \
    -p BDBV2026.drc.n1150 \
    --template-sg templates/GTR_SG_MAT.template.xml \
    --anchor-date 2025-11-08
```

### `postprocess_logs.py`

Combine BEAST chain logs, remove burnin, compute ESS for all parameters. Exits with code 2 if key parameters have ESS below threshold.

```bash
mamba run --live-stream -n bdbv python bin/postprocess_logs.py \
    --logs run1.log run2.log \
    --burnin 0.1 \
    --outdir output/ \
    --prefix analysis
```

Outputs: `{prefix}.combined.log`, `{prefix}.ess_report.txt`

### `postprocess_trees.py`

Remove burnin from `.trees` files, resample to manageable count, generate MCC tree via TreeAnnotator.

```bash
mamba run --live-stream -n bdbv python bin/postprocess_trees.py \
    --trees run1.trees run2.trees \
    --burnin 0.1 \
    --resample 1000 \
    --outdir output/ \
    --prefix analysis
```

Outputs: `{prefix}.burninRemoved.trees`, `{prefix}.burninRemoved.resampled.trees`, `{prefix}.mcc.tree`

## Data files

- **`data/aliases.csv`** — HZ name canonicalization (copied from bdbv-pipeline)
- **`data/health_area_zone_province_slim.csv`** — HZ → province lookup
- **`templates/`** — BEAST XML templates (GTR_SG, GTR_SG_MAT, GTR_EGC, GTR_LGC)

## Relationship to other repos

- **`bdbv-pipeline`** (Part 1) — Download, clean, align, build ML tree
- **`2026_BDBV_phylogeography`** — Phylogeographic analysis (DTA/BSSVS on empirical trees)
