# LACE2_SE Workflow Pipeline

## Description

Crosslinking site identification software for LACE-seq2 data. To improve library compatibility and to tailor the protocol for cancer cell applications, we optimized the original LACE-seq method from Dr. Yuanchao Xue's lab, renaming the updated version LACE-seq2. LACE-seq2 could identify RNA binding protein targets using UV crosslinking and linear amplification of complementary DNA ends. This pipeline processes paired-end RNA-seq data through adapter trimming, polyA trimming, quality control, rRNA removal, alignment, and peak calling. It's designed for analyzing LACE-seq2 or similar data.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Installation](#installation)
3. [Environment Setup](#environment-setup)
4. [Index Files Deployment](#index-files-deployment)
5. [Input Data Organization](#input-data-organization)
6. [Running the Pipeline](#running-the-pipeline)
7. [Demo Test](#demo-test)
8. [Output Files](#output-files)
9. [Troubleshooting](#troubleshooting)
10. [Citation](#citation)

## Prerequisites

- Linux/Unix operating system
- Conda package manager (Miniconda or Anaconda)
- Basic bioinformatics tools (wget, tar, gzip)
- Sufficient storage space for index files (~30GB recommended)

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/princewang2018/LACE-seq2.git
cd LACE-seq2
chmod +x LACE2

LACE2_PATH="/your/custom/path"
sudo echo "export PATH=\$PATH:$NEW_PATH" >> ~/.bashrc
source ~/.bashrc

```

### 2. Set up Conda environment

```bash
conda create -n LACE2 python=3.7.6 -y
conda activate LACE2

# Install all required tools
conda install -c bioconda \
    cutadapt=1.18 \
    fastqc=0.11.8 \
    star=2.7.3a \
    samtools=1.9 \
    bowtie2=2.5.4 \
    bedtools=2.27.1 \
    -y
    
    
OR With manba

conda install -n base -c conda-forge mamba

mamba create -n LACE2 python=3.8 -y
mamba activate LACE2
mamba install -c bioconda cutadapt fastqc star samtools bowtie2 bedtools -y
```

### 3. Install Piranha

Download and install Piranha as introduced in: https://github.com/smithlabcode/piranha

```shell
git clone https://github.com/smithlabcode/piranha.git

./configure
make all
make install
make test

cd /home/wzx/BIG_REF/biosoft/Piranha/piranha-1.2.1/bin
chmod +x Piranha

Piranha_PATH="/home/wzx/BIG_REF/biosoft/Piranha/piranha-1.2.1/bin"
sudo echo "export PATH=\$PATH:$Piranha_PATH" >> ~/.bashrc
source ~/.bashrc
```



### 4. Verify installation

```bash
conda activate LACE2
cutadapt --version
star --version
bowtie2 --version

```

### 

## Environment Setup

The pipeline uses a single Conda environment ("LACE2") containing all required tools:

- Cutadapt (adapter trimming)
- FastQC (quality control)
- Bowtie2 (rRNA removal)
- STAR (alignment)
- Samtools (BAM processing)
- Bedtools (BED conversion)
- Piranha (peak calling)

The pipeline requires:
- STAR genome index for your species (hg38, mm10, or hg19)
- Bowtie2 rRNA index

## Index Files Deployment

### 1. Download genome references

Example for hg38:

```bash
mkdir -p refs/hg38
cd refs/hg38
wget https://hgdownload.soe.ucsc.edu/goldenPath/hg38/bigZips/hg38.fa.gz
wget https://hgdownload.soe.ucsc.edu/goldenPath/hg38/bigZips/genes/hg38.knownGene.gtf.gz
gunzip *.gz
```

### 2. Build STAR indices

```bash
conda activate LACE20000000000000000000.............
STAR --runThreadN 16 \
     --runMode genomeGenerate \
     --genomeDir /path/to/your/STAR_index_hg38 \
     --genomeFastaFiles refs/hg38/hg38.fa \
     --sjdbGTFfile refs/hg38/hg38.knownGene.gtf \
     --sjdbOverhang 100
```

OR you can directly download pre-build index.

### 3. Build rRNA indices

First download rRNA sequences from NCBI or other databases:

```bash
# Download appropriate rRNA sequences for your organism
wget https://path/to/rRNA_sequences.fa
wget RNA_AND_TAXONOMY9606_AND_so_rna_type_nameRRNA.fasta.gz
```

Then build Bowtie2 index:

```bash
conda activate LACE2
bowtie2-build rRNA_sequences.fa Cen_rRNA
```

OR you can directly download our pre-build index:

```bash
wget https://github.com/PrinceWang2018/Cen_rRNA/blob/master/Cen_rRNA.zip
```



## Input Data Organization

Organize your input FASTQ files as follows:

```
work_directory/
└── 01.RawData/
    ├── Sample1/
    │   ├── Sample1_1.fq.gz
    │   └── Sample1_2.fq.gz
    ├── Sample2/
    │   ├── Sample2_1.fq.gz
    │   └── Sample2_2.fq.gz
    └── ...
```

## Running the Pipeline

For Demo data:

```bash
conda activate LACE2
#export PATH=/home/wzx/project16t_new/CD47/LACE-seq2-HNRNPA1-Merge-final-20250101/Test_Demo:$PATH
cd /home/wzx/project16t_new/CD47/LACE-seq2-HNRNPA1-Merge-final-20250101/Test_Demo
LACE2 \
  -w /home/wzx/project16t_new/CD47/LACE-seq2-HNRNPA1-Merge-final-20250101/Test_Demo \
  -s /home/wzx/BIG_REF/index_STAR_hg38/ \
  -r /home/wzx/BIG_REF/ncRNA_fasta/index/Cen_rRNA \
  -t 16 \
  -p 0.001 \
  -b 20
```

Basic command:

```bash
conda activate LACE2
LACE2 \
  -w /path/to/work_directory \
  -s /path/to/STAR_index_directory \
  -r /path/to/rRNA_index_prefix \
  -t 16 \
  -p 0.001 \
  -b 20
```

Parameters:
- `-w`: Working directory containing 01.RawData folder
- `-s`: Path to STAR index directory
- `-r`: Path to Bowtie2 rRNA index prefix
- `-t`: Number of threads (default: 8)
- `-p`: Piranha -p parameter (default: 0.001)
- `-b`: Piranha -b parameter (default: 20)

## Demo Test

A small test dataset is available at Github. To run the demo:

1. Download and prepare test data:

```bash
mkdir -p test_run/01.RawData/test_sample
cd test_run/01.RawData/test_sample
wget [link_to_test_R1].gz -O test_sample_1.fq.gz
wget [link_to_test_R2].gz -O test_sample_2.fq.gz
```

2. Run the pipeline:

```bash
LACE2 \
  -w $(pwd)/Test_Demo \
  -s /path/to/STAR_index_directory \
  -r /path/to/rRNA_index_prefix \
  -t 16 \
  -p 0.001 \
  -b 20
```

## Output Files

The pipeline creates the following directory structure:

```
work_directory/
├── 1remove_adapter/      # Adapter-trimmed files
├── 2remove_A/            # PolyA-trimmed files
├── 3QC/                  # Quality control reports
├── 4rmr/                 # rRNA-depleted files
├── 5Align/               # Alignment files (BAM, BAI)
├── 8Piranha/             # Peak calling results
└── LACE2_SE_workflow.done # Completion marker
```

## Troubleshooting

1. **Conda environment issues**:
   - Ensure you've activated the correct environment
   - Recreate environments if packages fail to load

2. **Memory errors**:
   - Reduce the number of threads if you encounter memory issues
   - STAR alignment may require significant RAM for large genomes

3. **Input file errors**:
   - Verify FASTQ files are properly named and paired
   - Check file integrity with `zcat file.fq.gz | head`

4. **Index file problems**:
   - Ensure STAR and Bowtie2 indices are built with compatible versions
   - Verify paths to index files are correct

## Citation

If you use this pipeline in your research, please cite:

> Wang, Z., Yang, L., Yang, S., Li, G., Xu, M., Kong, B., Shao, C., & Liu, Z. (2025). Isoform switch of CD47 provokes macrophage-mediated pyroptosis in ovarian cancer. bioRxiv, 2025.2004.2017.649282. https://doi.org/10.1101/2025.04.17.649282

For questions or issues, please open an issue on GitHub or contact wangzixiang@sdu.edu.cn.
