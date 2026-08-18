# Instructions for Transfering Genome from NCBI to RC

Author: Elle M. Barnes
Created: 2026-07-29

## Objective

This tutorial guides you through transferring a large reference genome—specifically the *Ambystoma mexicanum* (Axolotl) assembly **GCF_040938575.1** from NCBI—directly to the RIT Research Computing cluster, introducing essential Linux filesystem commands and batch job submission using Slurm.

## Important Information
 
**Understanding `sbatch` vs. Interactive Nodes**: When working on a cluster, interactive nodes let you run commands live in real-time, whereas `sbatch` sends your job to run automatically in the background on a dedicated compute node without requiring you to stay connected. While heavy computational workloads should always be requested via interactive nodes or submitted to compute nodes, it is safe and standard practice to run `sbatch` directly from the login node because the login node simply hand-offs your script to the cluster scheduler rather than running the heavy download itself.

### Basic Linux Commands

Before running the batch script, here are five essential Linux commands you will use during this workflow:

| Command | What It Does |
| --- | --- |
| **`mkdir`** | Creates a new directory (folder) to organize your files |
| **`cd`** | Changes your current working directory (folder) |
| **`nano`** | Opens a simple, terminal-based text editor to create or edit files |
| **`echo`** | Prints text or variable values directly to the terminal screen or log file |
| **`chmod`** | Modifies file permissions, such as making a downloaded file executable |

---
### Step 1: Create the Batch Script

Log into the cluster (`tigris.rc.rit.edu`). From the login node, navigate to our lab's shared project directory: `metagenome/MEGALab` and create a series of new folders for your project called `SalMetaG26`.

```bash
# Create and move to the target project directory
mkdir -p /shared/rc/metagenome/MEGALab/SalMetaG26
cd /shared/rc/metagenome/MEGALab/SalMetaG26
```
> Note: To remove failed downloads, run: `rm -f genomes/axolotl/axolotl_GCF_040938575.1.zip`

You will now add a series of nested folders called `genomes/axolotl`. Then, create a batch script file named `download_axolotl.sh` using a simple text editor like `nano`.

```bash
mkdir -p genomes/axolotl
cd genomes/axolotl
nano download_axolotl.sh
```

Alright, now we need to create the actual information within the `sbatch` script!

**What are `#SBATCH` directives?** Lines starting with #SBATCH are instructions for Slurm (the cluster's job scheduler) that tell it how many computational resources your task needs—such as time, CPU cores, and memory—and where to save your log files. These directives are critical because they ensure your job receives enough resources to finish without crashing, while preventing any single user from hogging cluster resources needed by others.

> UPDATE: Since we are now asking it to download the genome already compressed (`.fna.gz`) via FTP, the file is actually much smaller (~10 GB). In theory, downloading 10 GB at average cluster network speeds (20–50 MB/s) takes 5 to 10 minutes with another ~30 min for `gunzip`.

I have made my best estimation regarding the resources needed for this task, but here is some background:
  - `time` = Four hours is mathematically plenty, but because we are also asking it to `unzip` such a large set of files, it is better to add a little extra time.
  - `cpus-per-task` = Network downloads are almost entirely I/O bound (dependent on internet connection speed and NCBI server throttling), not CPU bound. Requesting 2, 4, or 16 CPUs will not speed up downloading the file from NCBI—1 CPU handles the transfer just fine. Standard `unzip` is a single-threaded operation, meaning it primarily uses 1 CPU core to decompress files. **However**, decompresses on massive multi-gigabyte archives (30+ GB) generate substantial disk I/O load. Having 4 CPUs gives the operating system enough compute room to manage background disk I/O, process streams, and handle system overhead smoothly without bottlenecking.
  - `mem` = (memory) The download uses almost no RAM. However, the `unzip` command on an axolotl assembly zip file (~28–32 GB compressed) requires significant overhead to map and unpack multi-gigabyte files.

Okay, paste the following script into the file:

```bash
#!/bin/bash
#SBATCH --account=metagenome
#SBATCH --partition=sporc
#SBATCH --job-name=ncbi_axolotl_download
#SBATCH --output=download_%j.log
#SBATCH --error=download_%j.err
#SBATCH --time=0-6:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=16GB

# Exit immediately if any command fails
set -e

TARGET_DIR="/shared/rc/metagenome/MEGALab/SalMetaG26/genomes/axolotl"
mkdir -p "${TARGET_DIR}"
cd "${TARGET_DIR}"

# Base FTP URL for assembly GCF_040938575.1 (UKY_AmexF1_1)
FTP_BASE="https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/040/938/575/GCF_040938575.1_UKY_AmexF1_1"

echo "Starting robust FTP download at $(date)..."

# 1. Download Genomic FASTA (The assembly sequence)
echo "Downloading Genomic FASTA..."
wget -c "${FTP_BASE}/GCF_040938575.1_UKY_AmexF1_1_genomic.fna.gz"

# 2. Download Gene Annotation GFF3
echo "Downloading GFF3 Annotations..."
wget -c "${FTP_BASE}/GCF_040938575.1_UKY_AmexF1_1_genomic.gff.gz"

# 3. Decompress the genomic FASTA file (Required for Bowtie2 indexing)
echo "Decompressing Genomic FASTA file for Bowtie2..."
gunzip -k GCF_040938575.1_UKY_AmexF1_1_genomic.fna.gz

echo "Download and extraction completed successfully at $(date)!"
```

Press `Ctrl + O`, hit `Enter` to save, and `Ctrl + X` to exit `nano`.

---

### Step 2: Submit and Monitor the Job

1. **Submit the Script:** Queue the job with Slurm.
Submit the job to the cluster scheduler using `sbatch`:

```bash
sbatch download_axolotl.sh
```

Slurm will respond with a unique job ID (e.g., `Submitted batch job 1234567`). At this point, **you can log off or shut down your computer completely.** The job is now running remotely in the background!

1. **Check Job Status:** Optional status checking.
If you log back in later, you can check whether the job is running or completed using Slurm commands:

Note: You do not need to replace `$USER` in the command below; the cluster will recognize your login.

```bash
# View your active or queued jobs
squeue -u $USER
```

If `squeue` returns empty, your job has finished running.

3. **Check Output Logs & Files:** Verify execution without errors.
To confirm the download finished successfully, inspect the output log file generated by Slurm:

```bash
# View the standard output log (replace 1234567 with your actual Job ID)
cat download_1234567.log

# Verify the unzipped contents
ls -lh /shared/rc/metagenome/MEGALab/SalMetaG26/genomes/axolotl/axolotl_dataset/ncbi_dataset/data/GCF_040938575.1/

```