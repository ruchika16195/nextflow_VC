# nextflow_VC
65
This repository contains a minimal **Nextflow** workflow designed to mimic an on‑premises HPC
cluster running **Slurm** and using **Singularity** containers. The pipeline performs the
following basic tasks:

1. Quality control (FASTQ or BAM) - exist
2. Alignment (from FASTQ using **bwa**) - YTD
3. BAM indexing - YTD
4. Variant calling (using **freebayes**) - YTD

The configuration is written so it can be executed locally on a Mac for testing and then
switched to a real Slurm cluster by selecting the appropriate profile.

---

## Directory layout

```
/nextflow_VC
├── workflow/main.nf         # Nextflow pipeline definition
├── nextflow.config          # configuration profiles (local, slurm, test)
├── test_data/               # small test inputs (BAM + reference)
│   ├── test.bam
│   ├── test.bam.bai
│   └── ref.fa
├── README.md                # this file
├── .gitignore
└── ...
```

## Prerequisites

- macOS or Linux workstation
- [Nextflow](https://www.nextflow.io/) installed (`brew install nextflow` or `curl -sS
  https://get.nextflow.io | bash`)
- Singularity (for container execution) or Docker/Podman if you are on a macOS
  machine.  The examples use `-with-singularity` but `-with-docker` is an easy
  alternative when Singularity is not installed.
- `bwa`, `samtools`, `freebayes`, `fastqc` are pulled via containers defined in the pipeline

> On a real Slurm cluster these binaries do **not** need to be installed locally; the
> containers provide them.

## Testing locally with a sample BAM

The repository includes a tiny example BAM (`test_data/test.bam`) and its index.  If you
wish to refresh or obtain the file yourself run:

```bash
mkdir -p test_data
curl -L -o test_data/test.bam \
    https://samtools.github.io/hts-specs/examples/test.bam
curl -L -o test_data/test.bam.bai \
    https://samtools.github.io/hts-specs/examples/test.bam.bai
```

Once the BAM is in place, execute the pipeline using the `test` profile:

```bash
cd /workspaces/nextflow_VC          # adjust path if necessary
nextflow run workflow/main.nf \
    -profile test                  \
    -with-singularity             \
    -resume                        \
    -c nextflow.config
```

This will:

* use the `test` profile which points `params.bam` to the sample BAM and uses a tiny
  reference in `test_data/ref.fa`;
* execute the `QC_BAM` process (samtools flagstat) and then the `VARCALL` process
  (freebayes);
* store output under `results/` in sub‑directories `qc/` and `variants/`.

You can also run the full FASTQ‑to‑VCF pipeline by providing your own reads and reference:

```bash
nextflow run workflow/main.nf \
    --reads '/path/to/reads/*_{1,2}.fastq.gz' \
    --reference '/path/to/genome.fa' \
    --outdir '/shared/results' \
    -with-singularity
```

### Simulating Slurm

Local testing uses the `standard` profile (executor=`local`).  To prepare for deployment on
an HPC cluster with Slurm, switch to the `slurm` profile.  The configuration in
`nextflow.config` sets a generic queue and an example `workDir` on a shared filesystem.
Adjust these settings to match your environment.

```bash
nextflow run workflow/main.nf \
    -profile slurm \
    --reads '...' --reference '...' \
    -with-singularity
```

> On a cluster the `-with-singularity` flag can be omitted if the containers are
> accessible through the `SINGULARITY_CONTAINER` environment variable or a local cache.

## Extending the workflow

The pipeline is intentionally minimal.  Add additional steps such as trimming, merging
multiple BAMs, advanced QC, or joint calling by editing `workflow/main.nf` and defining
new processes.  See the [Nextflow documentation](https://www.nextflow.io/docs/latest/) for
syntax and examples.

## Notes

* Results are published in `results/` by default; change with `--outdir` or in the config
  file.
* The Slurm profile is a template – modify `process.queue`, `clusterOptions`, and
  `workDir` according to your HPC policies.

---

## Docker image

A `Dockerfile` is included at the repository root that installs the JVM, Nextflow,
and Python so you can build a container for running the workflow in an isolated
environment. The image also installs any Python packages listed in
`requirements.txt`.

To build the image locally:

```bash
cd /workspaces/nextflow_VC
docker build -t nextflow-vc .
```

Once built you can run the shell inside the container and invoke Nextflow from
there:

```bash
docker run --rm -it -v $PWD:/workspace nextflow-vc
# inside container:
nextflow run workflow/main.nf -profile test -with-singularity -resume -c nextflow.config
```

Edit `requirements.txt` to add or remove Python libraries; the container will
install whatever is listed when it is built.

---

Happy genotyping! 😉
