# Running PICRUSt2 on high-throughput amplicon sequencing data

Author: Elle M. Barnes

Created: 2026-08-11

## Objective

This tutorial guides you through executing the PICRUSt2 pipeline on amplicon sequencing data to predict functional gene families and metabolic pathways present in a microbial community.

### Citations
Make sure to cite all the associated publications when using PICRUSt2. The full list can be found here: https://github.com/picrust/picrust2/wiki/PICRUSt2-Tutorial-(v2.5.2)#how-to-cite-picrust2

---
## Step 1: PICRUSt2 installation
The easiest way to install PICRUSt2 is with bioconda, which will install a new environment which contains all the software needed to run PICRUSt2. 

```bash
conda create -n picrust2 -c bioconda -c conda-forge picrust2=2.6.2
```

## Step 2: Data format
PICRUSt2 can be run on amplicon data processed in QIIME2. You only need two files:
1. DNA sequences (as multi-FASTA; .fna)
2. Feature table (as .biom or .tsv) containing the abundances for each ASV (ASV names need to match the headers in the FASTA)

> Note: You can check the format of your FASTA by running `less [file name]` (and go back to the terminal by typing `q`). You can check the format of your biom file by running `biom head -i [file name]`.

### 1. Representative Sequences File

PICRUSt2 requires unaligned, nucleotide sequences in standard FASTA format. Header IDs **must exactly match** the feature IDs in your BIOM/table file.

```fasta
>ASV1
TACGTAGGGTCCAAGCGTTAATCGGAATTACTGGGCGTAAAGCGTGCGCAGGCGGTTTGTTAAGTCAGATGTGAAATCCCCGGGCTCAACCTGGGAACTGC
>ASV2
TACGTAGGGTCCGAGCGTTAATCGGAATTACTGGGCGTAAAGCGTGCGCAGGCGGTTTGTTAAGTCAGATGTGAAATCCCCGGGCTCAACCTGGGAACTGC
>ASV3
TACGTAGGGGGCAAGCGTTATCCGGAATTATTGGGCGTAAAGCGCGCGCAGGCGGTTTGTTAAGTCAGATGTGAAATCCCCGGGCTCAACCTGGGAACTGC
```
### 2. Feature Table

PICRUSt2 accepts standard `.biom files` or plain-text tab-separated (`.tsv`) feature matrices.

**Option A**: Tab-Separated Values (feature-table.tsv)
If supplying a TSV matrix, the header must begin with `#OTU ID` (or `ASV_ID`) and list sample IDs across the columns.

```
#OTU ID	Sample1	Sample2	Sample3	Sample4
ASV1	      145	      0	     89	     12
ASV2	        0	    521	    340	      8
ASV3	       88	     12	      0	    452
```

**Option B**: JSON BIOM Header Structure

If using binary or JSON BIOM format (e.g., exported directly from QIIME 2), inspecting the converted JSON representation reveals this structural header:

```
{
  "id": "Feature Table",
  "format": "1.0.0",
  "format_url": "[http://biom-format.org](http://biom-format.org)",
  "type": "OTU table",
  "generated_by": "QIIME 2",
  "rows": [
    {"id": "ASV1", "metadata": null},
    {"id": "ASV2", "metadata": null},
    {"id": "ASV3", "metadata": null}
  ],
  "columns": [
    {"id": "Sample1", "metadata": null},
    {"id": "Sample2", "metadata": null},
    {"id": "Sample3", "metadata": null},
    {"id": "Sample4", "metadata": null}
  ],
  "matrix_type": "sparse",
  "matrix_element_type": "int",
  "shape": [3, 4],
  "data": [[0, 0, 145], [0, 2, 89], [0, 3, 12], [1, 1, 521], ...]
}
```
## Step 3: Data pre-processing
A few things you should keep in mind for your own data:
- Depending on the pipeline you used to process your raw 16S data there may be many extremely rare ASVs found across your samples. Such rare ASVs typically only add noise to analyses (especially singletons - ASVs found only by 1 read in 1 sample) and should be removed. Reducing the number of input ASVs will also make PICRUSt2 faster and less memory intensive.
  
- Similarly, it is important to check for any low-depth samples that should be removed. These can be identified based on the output of biom summarize-table above.

- The best minimum cut-offs for excluding ASVs and samples varies by dataset since these cut-offs will differ depending on the overall read depth of your dataset.

### Examples
Trimming/pre-processing can easily be done in the terminal using QIIME2 (not in PICRUSt2). Thus, make sure to activate your QIIME2 environment.

> Note: Don't forget to load conda at the start of each login shell session.

```bash
conda activate qiime2-2026.7 # latest install when drafting this tutorial
```

Example 1: Trim **samples** with a total frequency less than 1500
```bash
qiime feature-table filter-samples \
  --i-table tabledada16S.qza \
  --p-min-frequency 1500 \
  --o-filtered-table sample-frequency-filtered-table.qza
```

Example 2: Remove all **features** with a total abundance (summed across all samples) of less than 10
```bash
qiime feature-table filter-features \
  --i-table sample-frequency-filtered-table.qza \
  --p-min-frequency 10 \
  --o-filtered-table feature-frequency-filtered-table.qza
```

Once you are done filtering your feature table, you need to make sure to re-export it in `.biom` format.
```bash
qiime tools export \
  --input-path feature-frequency-filtered-table.qza \
  --output-path exported-feature-table2
```

You also need to filter your **sequences** to match to avoid using unnecessary compute resources
```bash
qiime feature-table filter-seqs \
  --i-data repseqsdada16S.qza \
  --i-table feature-frequency-filtered-table.qza \
  --o-filtered-data filtered-rep-seqs.qza
```
...and then export the fasta file
```bash
qiime tools export \
  --input-path filtered-rep-seqs.qza \
  --output-path exported2
```

## Step 4: Running PICRUSt2

Make sure to request an interactive session using `sinteractive` and then activate your conda environment.
```bash
conda activate picrust2
```

Running PICRUSt2 can be done all in one command! You just need to swap in the names for the sequence and feature table files.

Here is an example:
```bash
picrust2_pipeline.py -s exported2/dna-sequences.fasta -i exported-feature-table2/feature-table.biom -o picrust2_out_pipeline -p 1
```

### Run as a batch script:

Create a batch script named `picrust2_run.sh`:
```bash
nano picrust2_run.sh
```
Paste in this info into the nano box (i.e., script):
```bash
#!/bin/bash
#SBATCH --account=metagenome
#SBATCH --partition=sporc
#SBATCH --job-name=picrust2
#SBATCH --output=picrust2_%j.log
#SBATCH --error=picrust2_%j.err
#SBATCH --time=0-08:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=16
#SBATCH --mem=64G

# Load conda
source $HOME/miniconda3/bin/activate

# activate picrust2 env
conda activate picrust2

# set directory
cd /shared/rc/metagenome/MEGALab/Salamander16S/watersoilswabs_2025/30-1059075613

# Run PICRUSt2 pipeline
picrust2_pipeline.py \
    -s exported2/dna-sequences.fasta \
    -i exported-feature-table2/feature-table.biom \
    -o picrust2_out_pipeline \
    -p ${SLURM_CPUS_PER_TASK:-1}

```

Submit the job:
```bash
sbatch picrust2_run.sh
```