# Prokaryota dry lab: 01 Metagenomic Assembly
### Based on ONT (Oxford Nanopore Technologies) long read sequencing of cow rumen samples
#### Wednesday 5th of April 2023

Hello! Welcome to the metagenomic dry lab session! Here we will take the raw basecalled reads from the nanopore sequenator and try to reconstruct the microbial genomes that they come from.

#### Checking access to conda environment
For these exercises we will be using a conda environment in /mnt/courses/BIO326/PROK/condaenv that contains most of the required software.

You can activate it temporarily, and check that all software is ready to rock and roll:

```bash
conda activate /mnt/courses/BIO326/PROK/condaenv
filtlong --version
#> Filtlong v0.2.1
flye --version
#> 2.9.1-b1780
assembly-stats -v
#> Version: 1.0.1
minimap2 --version
#> 2.24-r1122
racon --version
#> 1.5.0
medaka --version
#> medaka 1.7.2
metabat2 2> >(grep -oE "\(version.+$")
#> (version 2:2.15 (Bioconda); 2022-11-19T22:34:33)
conda deactivate

```




## Quality control and filtering of the raw reads 🛂

Your raw reads from the prokaryotic sequencing session reside in "/mnt/courses/BIO326/PROK/data/metagenomic_assembly/".

There is one file named "raw_reads_nanopore_2308.fastq.gz"

Fastq is a raw read format containing a base quality scores for nucleotide position along each read.
Sometimes when we sequence, we see a lot of low quality reads that we want to get rid of, because they mostly contain noise that confuse the downstream analysis.

By specifying `--min_length 1000` and `--keep_percent 90` we keep only the reads that live up to these requirements.


📝 Create a file named 01a_filter-filtlong.sh with the following contents, and submit the job with sbatch: Make sure to change the <path to raw reads> into the path of the actual raw reads.

```bash
#!/bin/bash

# Define slurm parameters
#SBATCH --job-name=filter-filtlong
#SBATCH --time=01:00:00
#SBATCH --cpus-per-task 1
#SBATCH --mem=1G

# Activate the conda environment
source activate /mnt/courses/BIO326/PROK/condaenv

# Define paths
in="<path to raw reads>"
out="results/filtlong/output.fastq.gz"

# Make sure that the output directory exists
mkdir -p results/filtlong/


filtlong --min_length 1000 --keep_percent 90 $in | gzip > $out


```

Now check your output/filtlong/ directory. There should be a compressed fastq output file.

```bash
tree results/filtlong/
#> results/filtlong/
#> └── output.fastq.gz
#> 
#> 0 directories, 1 file
```




## Assemble reads into a draft assembly 🏗

Currently, we have the sequenced reads that represent fragments of the biological genomic sequences that we want to reconstruct inside the computer. We're going to use an approach that works a bit by solving a puzzle: Each piece is compared to many other pieces, until the whole picture can be put together. In the same way, we overlap the reads with one another, until we get long continuous sequences a.k.a. "contigs". Hopefully, these contigs will represent larger parts of the original biological chromosomes and plasmids that harbor the prokaryotic cells in our sampled microbial community.

Here we will use the Flye assembler (https://github.com/fenderglass/Flye/). It takes in reads from genomic sequencing, and puts out a long draft assembly that contains sequence contigs from all the species that are present in the samples we sequenced.

📝 Create a file named 01b_assemble-flye.sh with the following contents, and submit the job with sbatch: Remember to change the "in" variable to point to your copy of the nanopore_reads.fastq.gz file.

```bash
#!/bin/bash

# Define slurm parameters
#SBATCH --job-name=assemble-flye
#SBATCH --time=24:00:00
#SBATCH --cpus-per-task 8
#SBATCH --mem=30G

# Activate the conda environment
source activate /mnt/courses/BIO326/PROK/condaenv

# Define paths
in="<path to nanopore reads>"
out="results/flye" # Note: this is a directory, not a file.

# Prepare output directory
mkdir results

flye --meta --nano-hq $in --threads $SLURM_CPUS_PER_TASK --out-dir $out --iterations 2


```


**Bonus points**: If you look closely at the Flye program call in the sbatch script above, you can see that we're passing the "--meta" argument. By investigating the Flye documentation (https://github.com/fenderglass/Flye/blob/flye/docs/USAGE.md), can you explain briefly what the "meta" mode does, and argue why we want to use it in this setting?

---

When Flye finishes, you will see a lot of output files in the results/flye directory. The main result file that we'll continue with is the results/flye/assembly.fasta file.

```bash
ls -lh results/flye/
#> total 480M
#>    2 Mar 15 15:00 00-assembly/
#>    3 Mar 15 15:49 10-consensus/
#>    6 Mar 15 16:13 20-repeat/
#>    7 Mar 15 16:15 30-contigger/
#>   11 Mar 16 04:17 40-polishing/
#> 233M Mar 16 04:18 assembly.fasta
#> 224M Mar 16 04:18 assembly_graph.gfa
#> 3.1M Mar 16 04:17 assembly_graph.gv
#> 384K Mar 16 04:18 assembly_info.txt
#>  20M Mar 16 04:18 flye.log
#>   92 Mar 16 04:17 params.json
```


You can investigate some basic statistics of the Flye assembly using the assembly-stats software:

```bash
assembly-stats results/flye/assembly.fasta 
#> stats for results/flye/assembly.fasta
#> sum = 239436989, n = 10266, ave = 23323.30, largest = 1194178
#> N50 = 33581, n = 1668
#> N60 = 25598, n = 2487
#> N70 = 19593, n = 3565
#> N80 = 15137, n = 4951
#> N90 = 10904, n = 6804
#> N100 = 114, n = 10266
#> N_count = 0
#> Gaps = 0
```

As the algorithms implemented in Flye are not deterministic, your output may vary slightly.
As you can see here, we have draft assembly with 10266 contigs totalling 239 Megabases. N50 shows the length of the shortest contig that together with all larger contigs sums to at least half of the total bases in the full draft assembly. 


**Summary**

At this point you will have a draft assembly that represents the longest sequences that can be recovered by overlapping the reads. In the next exercise we will polish this draft assembly and prepare it to be split into each of the species in the microbial community.

