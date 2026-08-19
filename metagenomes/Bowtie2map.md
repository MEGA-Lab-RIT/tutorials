# Removing host reads from raw shotgun metagenomic reads

Author: Elle M. Barnes
Created: 2026-07-29

## Objective

This tutorial guides you through using `Bowtie2` on the RIT Research Computing cluster to align raw metagenomic sequencing reads against the *Ambystoma mexicanum* (Axolotl) reference genome, filtering out host contamination and saving only non-host (microbial) reads for downstream analysis.

---

### Linux Commands Used Here

| Command | What It Does |
| --- | --- |
| **`bowtie2-build`** | Indexes a reference genome FASTA file so Bowtie2 can align reads against it rapidly. | 
| **`bowtie2`** | Aligns sequencing reads against a pre-built index and filters out mapped/unmapped reads. | 
| **`samtools`** | A suite of utilities for manipulating high-throughput sequencing data formats (SAM/BAM). | 
| **`gzip`** / **`gunzip`** | Compresses or decompresses FASTQ files to save cluster storage space. | 

---

### Step 1: Building the Bowtie2 Genome Index

Before aligning reads, Bowtie2 must convert the genome FASTA into a fast-lookup index. Because the Axolotl genome is over 4 GB, Bowtie2 will automatically build a **Large Index** (`.bt2l` extensions instead of `.bt2`).

Create a batch script named `build_index.sh` inside your lab directory:

```bash
cd /shared/rc/metagenome/MEGALab/SalMetaG26
nano build_index.sh
```
> **Note**: You *do not* need to install Bowtie2 yourself! The command `spack env` and accesses the pre-installed version maintained by the RIT Research Computing team.

Paste the following script into `build_index.sh`:

```bash
#!/bin/bash
#SBATCH --account=metagenome
#SBATCH --partition=sporc
#SBATCH --job-name=bowtie2_index_axolotl
#SBATCH --output=build_index_%j.log
#SBATCH --error=build_index_%j.err
#SBATCH --time=0-24:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=16
#SBATCH --mem=300GB

set -e

# Set working directory paths
WORKDIR="/shared/rc/metagenome/MEGALab/SalMetaG26/genomes/axolotl"
GENOME_FASTA="${WORKDIR}/axolotl_dataset/ncbi_dataset/data/GCF_040938575.1/GCF_040938575.1_UKY_AmexF1_1_genomic.fna"
INDEX_DIR="${WORKDIR}/bowtie2_index"
INDEX_PREFIX="${INDEX_DIR}/axolotl_large_index"

# Create directory for index files
mkdir -p "${INDEX_DIR}"
cd "${WORKDIR}"

echo "Starting Bowtie2 index construction at $(date)..."

# Load the RIT Spack environment containing Bowtie2
spack env activate default-genomics-x86_64-25091001

# Build large index using 16 CPU threads
bowtie2-build --threads 16 --bmaxdiv 8 "${GENOME_FASTA}" "${INDEX_PREFIX}"

echo "Indexing successfully completed at $(date)!"

```

Submit the indexing job:

```bash
sbatch build_index.sh
```

> **Note on Indexing:** Indexing an ~30 GB genome requires substantial RAM (~64 GB) and time (roughly 4–8 hours). This step **only needs to be run ONCE**. All future metagenome samples can share this single index.

---

### Step 2: Mapping Metagenomes & Extracting Non-Host Reads

Once the index is built, run Bowtie2 to align paired-end metagenomic FASTQ files (`R1.fastq.gz` and `R2.fastq.gz`) against the index.

We use the flags `--un-conc-gz` (or `--un-conc`) to automatically save paired-end reads that **failed to align to the host genome** directly into new, compressed FASTQ files.

Create a batch script named `remove_host.sh`:

```bash
nano remove_host.sh
```

Paste the following script:

```bash
#!/bin/bash
#SBATCH --account=metagenome
#SBATCH --partition=sporc
#SBATCH --job-name=bowtie2_host_removal
#SBATCH --output=host_removal_%j.log
#SBATCH --error=host_removal_%j.err
#SBATCH --time=0-08:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=16
#SBATCH --mem=32G

set -e

# Define directory locations
WORKDIR="/shared/rc/metagenome/MEGALab/SalMetaG26"
INDEX_PREFIX="${WORKDIR}/genomes/axolotl/bowtie2_index/axolotl_large_index"
RAW_DIR="${WORKDIR}/raw_data"
OUTPUT_DIR="${WORKDIR}/clean_reads"

mkdir -p "${OUTPUT_DIR}"
cd "${WORKDIR}"

echo "Loading Spack environment: default-genomics-x86_64-25091001..."
spack env activate default-genomics-x86_64-25091001

# Loop through all forward read (R1) files in raw_data
for R1 in "${RAW_DIR}"/*_R1.fastq.gz; do
    # Infer the matching reverse read (R2) file name
    R2="${R1/_R1.fastq.gz/_R2.fastq.gz}"
    
    # Extract just the base sample name (e.g., "sampleA" from ".../sampleA_R1.fastq.gz")
    SAMPLE_NAME=$(basename "${R1}" _R1.fastq.gz)

    echo "=================================================="
    echo "Processing sample: ${SAMPLE_NAME}"
    echo "Start time: $(date)"
    echo "=================================================="

    bowtie2 \
      -x "${INDEX_PREFIX}" \
      -1 "${R1}" \
      -2 "${R2}" \
      --threads 16 \
      --very-sensitive-local \
      --un-conc-gz "${OUTPUT_DIR}/${SAMPLE_NAME}_nonhost_R%.fastq.gz" \
      -S "${OUTPUT_DIR}/${SAMPLE_NAME}_temp.sam"

    # Clean up temporary SAM file immediately to conserve storage
    rm -f "${OUTPUT_DIR}/${SAMPLE_NAME}_temp.sam"

    echo "Finished sample: ${SAMPLE_NAME} at $(date)"
done

echo "All samples successfully processed!"
```

> **IMPORTANT**: As you can see in the script above under "Define directories" there is a directory for your raw shotgun metagenome reads. You need to make sure to create this directory and place all `...R1.fastq.gz` and `...R2.fastq.gz` files within it before attempting to run the script.

Submit the host-filtering job:

```bash
sbatch remove_host.sh
```

---

### Step 3: Understanding Key Bowtie2 Parameters

| Flag | Purpose |
| --- | --- |
| **`-x`** | Specifies the path and prefix of the pre-built genome index. |
| **`-1` / `-2**` | Paths to input paired-end forward (R1) and reverse (R2) FASTQ reads. |
| **`--threads 16`** | Distributes alignment across 16 CPU cores to speed up execution. |
| **`--very-sensitive-local`** | Higher accuracy alignment preset; helps catch reads that partially match host sequences. |
| **`--un-conc-gz`** | **The key filter step:** Saves paired reads that **did NOT align** to the host genome into gzipped files (creates `sample1_nonhost_R1.fastq.gz` and `sample1_nonhost_R2.fastq.gz`). |

---

### Step 4: Verification & Quality Control

Once the job finishes, inspect the Slurm log file (`host_removal_<JOBID>.err` or `.log`) to check alignment statistics:

```bash
cat host_removal_*.err

```

Look for the summary lines printed by Bowtie2:

```text
10000000 reads; of these:
  10000000 (100.00%) were paired; of these:
    8500000 (85.00%) aligned concordantly 0 times  <-- (YOUR NON-HOST READS)
    1500000 (15.00%) aligned concordantly >1 times <-- (REMOVED HOST READS)
15.00% overall alignment rate

```

Your clean, microbial-only reads ready for downstream metagenomic assembly or profiling will be stored in:
`/shared/rc/metagenome/MEGALab/SalMetaG26/clean_reads/`