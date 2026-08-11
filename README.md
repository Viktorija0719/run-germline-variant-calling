# run-germline-variant-calling

Site-specific configuration and inputs for running the
[`nf-germline-variant-calling`](https://github.com/Viktorija0719/nf-germline-variant-calling)
pipeline on the RTU/Rudens HPC cluster (PBS/Torque). This directory holds no
pipeline code — only what changes per run/per site: parameters, samplesheet,
cluster config, PBS launch script, and logs.

**Pipeline repository:** https://github.com/Viktorija0719/nf-germline-variant-calling

The pipeline itself is **not** checked out here. Nextflow fetches it straight
from GitHub as a named asset (`Viktorija0719/nf-germline-variant-calling`), so
there is no sibling directory to keep in sync — `run_all.pbs` runs the pipeline
by name, not by path.

## Getting the pipeline

Nextflow stores pulled pipelines under `$NXF_HOME/assets/`. `run_all.pbs` sets
`NXF_HOME=/home_beegfs/$USER/nxf_work/nextflow_home`, so pull with the same
`NXF_HOME` or the job will not see what you pulled:

```bash
module load java/jdk-21.0.2
export NXF_HOME=/home_beegfs/$USER/nxf_work/nextflow_home
nextflow pull Viktorija0719/nf-germline-variant-calling
```

Useful follow-ups (all need the same `NXF_HOME`):

| Command | Purpose |
|---------|---------|
| `nextflow pull Viktorija0719/nf-germline-variant-calling` | Fetch, or update an already-pulled copy to the latest commit |
| `nextflow pull Viktorija0719/nf-germline-variant-calling -r <branch\|tag\|sha>` | Pull a specific revision |
| `nextflow info Viktorija0719/nf-germline-variant-calling` | Show the local copy's path and available revisions |
| `nextflow drop Viktorija0719/nf-germline-variant-calling` | Delete the local copy (forces a clean re-pull) |

Pulling up front is optional — `nextflow run <name>` pulls on first use — but
doing it as a login-node step means the download and any GitHub auth problem
surface immediately instead of inside a queued PBS job. It also pins what the
job will run: an already-pulled asset is **not** auto-updated by `nextflow run`,
so re-pull deliberately when you want new pipeline code.

If the repository is private, give Nextflow a token in
`$NXF_HOME/scm` before pulling:

```
providers {
  github {
    user = 'Viktorija0719'
    password = '<github-personal-access-token>'
  }
}
```

## Contents

| Path | Purpose |
|------|---------|
| `samplesheet.csv` | FASTQ input samplesheet — 5 paired-end samples (`RUN_TAG=fastq`, the default) |
| `samplesheet_bam.csv` | BAM input samplesheet for the pre-aligned neuro cohort (`RUN_TAG=bam`) — not in this checkout, add it before running with `RUN_TAG=bam` |
| `params/unified.yaml` | Site parameters for the FASTQ run: reference paths, truth VCFs, annotation caches (all under `/home/groups/rsu/...`) |
| `params/unified_bam.yaml` | Same, for the BAM run |
| `params/snpsift_databases.csv` | ClinVar annotation DB paths for SnpSift |
| `conf/rtu_pbs.config` | PBS executor config (queue, per-process resource overrides, error-handling tweaks) |
| `conf/genomes_local.config` | Site-local `RSU.GRCh38_no_alt` reference (the no-alt analysis set, used instead of the GATK bundle FASTA whose decoy contigs break Strelka) — see the header comment for the one-time reference prep on the HPC |
| `run_all.pbs` | PBS submission script — despite the name, runs the single unified pipeline (not a multi-step chain) |
| `logs/` | Nextflow run reports/timelines/traces and PBS job stdout from past runs (created by the job) |

## Running

```bash
cd run-germline-variant-calling
qsub run_all.pbs                    # FASTQ: 5 samples from samplesheet.csv
qsub -v RUN_TAG=bam run_all.pbs     # BAM:  samples from samplesheet_bam.csv
```

`RUN_TAG` selects the entry point. The pipeline itself picks FASTQ vs BAM by
looking for a `bam` column in the samplesheet
(`Utils.detectSampleSheetType`) — `RUN_TAG` just chooses which params file,
work dir, and Nextflow cache dir to use:

| `RUN_TAG` | params file | work dir | cache dir |
|-----------|-------------|----------|-----------|
| `fastq` (default) | `params/unified.yaml` | `…/nxf_work/germline_fastq` | `.nextflow_fastq/` |
| `bam` | `params/unified_bam.yaml` | `…/nxf_work/germline_bam` | `.nextflow/` |

The work dirs and caches are kept separate on purpose. A bare `-resume` resumes
whichever session ran last, so a shared cache dir would mean each switch between
FASTQ and BAM discarded the other's completed work.

Note that both params files currently set `outdir: "results"`. The cohort-level
outputs (`multiqc/`, `cohort_qc/`, benchmark summaries) are not sample-named, so
running both entry points will have them overwrite each other — give one of the
two its own `outdir` if you need both sets of reports side by side.

`run_all.pbs`:
1. Loads `java` and `singularity` modules
2. Sets up per-job Nextflow/Singularity cache and tmp dirs under
   `/home_beegfs/$USER/nxf_work/` (BeeGFS doesn't support the hard links
   Singularity uses to unpack images, so the SIF build tmpdir is redirected
   to the compute node's local `/tmp`)
3. Runs:
   ```bash
   nextflow run Viktorija0719/nf-germline-variant-calling \
     -profile singularity \
     -params-file "$PARAMS_FILE" \
     -c conf/rtu_pbs.config \
     -c conf/genomes_local.config \
     -work-dir "$WORKDIR" \
     -with-report  "$LOGDIR/unified_${RUN_TAG}_report.html" \
     -with-timeline "$LOGDIR/unified_${RUN_TAG}_timeline.html" \
     -with-trace   "$LOGDIR/unified_${RUN_TAG}_trace.txt" \
     -resume
   ```

Check job status with `qstat`. All runs use `-resume`, so a re-submitted job
reuses completed work from the Nextflow cache in this directory.

To pin a pipeline revision for a run, add `-r <branch|tag|sha>` to the
`nextflow run` line in `run_all.pbs`; without it Nextflow uses the default
branch of whatever is currently pulled.

## Updating for a new run

- Point `samplesheet.csv` at new FASTQ pairs (or run with `RUN_TAG=bam` to
  skip alignment and start from `samplesheet_bam.csv`).
- One row per sample **per lane**: `id` becomes `<sample>_<lane>`, lanes are
  aligned separately and merged before dedup, and duplicate `sample`+`lane`
  keys abort the run up front.
- Adjust `params/unified.yaml` for a different reference genome, target BED,
  or truth set — everything under `/home/groups/rsu/...` is a shared
  resource path, not something this repo owns.
- `gatk_gene_list` / `xhmm_param_file` are intentionally **not** set in
  `params/unified.yaml` — they fall back to the pipeline's own
  `${projectDir}/resources/...` defaults, which resolve to the pulled asset
  directory under `$NXF_HOME/assets/`, regardless of where you launch
  Nextflow from.
- To take new pipeline code, re-run `nextflow pull` (with the job's `NXF_HOME`)
  before submitting; a pulled asset is otherwise reused as-is.
