# LACE-seq2 Workflow Pipeline
[![DOI](https://zenodo.org/badge/999354173.svg)](https://doi.org/10.5281/zenodo.17149288)
## Description

Crosslinking site identification software for LACE-seq2 data. To improve library compatibility and to tailor the protocol for cancer cell applications, we optimized the original LACE-seq method from Dr. Yuanchao Xue's lab, renaming the updated version LACE-seq2. LACE-seq2 could identify RNA binding protein targets using UV crosslinking and linear amplification of complementary DNA ends. This pipeline processes paired-end RNA-seq data through adapter trimming, polyA trimming, quality control, rRNA removal, alignment, and peak calling. It's designed for analyzing LACE-seq2 or similar data.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Environment Setup](#environment-setup)
3. [Installation](#installation)
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

## Environment Setup

The pipeline uses a single Conda environment ("LACE2") containing required tools:

- Cutadapt (adapter trimming)
- FastQC (quality control)
- Bowtie2 (rRNA removal)
- STAR (alignment)
- Samtools (BAM processing)
- Bedtools (BED conversion)

Piranha was install from Github separately.

The pipeline requires:

- STAR genome index for your species (hg38, mm10, or hg19)
- Bowtie2 rRNA index

## Installation

### 1. Clone the repository

```bash
cd the_dir_you_want_to_install_the_pipeline
git clone https://github.com/princewang2018/LACE-seq2.git
cd LACE-seq2
chmod 777 LACE2

LACE2_PATH=$(pwd)
sudo echo "export PATH=\$PATH:$LACE2_PATH" >> ~/.bashrc
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
```

OR install with manba

```shell
conda install -n base -c conda-forge mamba
mamba create -n LACE2 -y
mamba activate LACE2
mamba install -c bioconda cutadapt fastqc star samtools bowtie2 bedtools -y
```

### 3. Install Piranha

Download and install Piranha as introduced in:

> https://github.com/smithlabcode/piranha
>
> https://blog.csdn.net/weixin_42035282/article/details/131708094

#### 1) Install gsl

```shell
wget https://ftp.gnu.org/gnu/gsl/gsl-2.7.1.tar.gz
tar -zxvf gsl-2.7.1.tar.gz 

mkdir -p /Path/to/install/gsl_gcc
cd gsl-2.7.1
 ./configure CC="gcc" --prefix=/Path/to/install/gsl_gcc
make
make install

# Add environment variables
export C_INCLUDE_PATH=$C_INCLUDE_PATH:/Path/to/install/gsl_gcc/include
export CPLUS_INCLUDE_PATH=$CPLUS_INCLUDE_PATH:/Path/to/install/gsl_gcc/include
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH::/Path/to/install/gsl_gcc/lib
export LIBRARY_PATH=$LIBRARY_PATH::/Path/to/install/gsl_gcc/lib
```

#### 2) Install Piranha

```shell
git clone --recursive https://github.com/smithlabcode/piranha.git

cd ./piranha/
./configure
make all
make install

cd ./bin
chmod +x Piranha
Piranha_PATH=$(pwd)
sudo echo "export PATH=\$PATH:$Piranha_PATH" >> ~/.bashrc
source ~/.bashrc
```

### 4. Verify installation

```bash
conda activate LACE2
cutadapt --version
STAR --version
bowtie2 --version
Piranha -help
```

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

### 2. Build STAR index

```bash
conda activate LACE2
STAR --runThreadN 16 \
     --runMode genomeGenerate \
     --genomeDir /path/to/your/STAR_index_hg38 \
     --genomeFastaFiles refs/hg38/hg38.fa \
     --sjdbGTFfile refs/hg38/hg38.knownGene.gtf \
     --sjdbOverhang 149
```

You can also directly download or use pre-build STAR index compatible with STAR (corresponding version).

### 3. Build  bowtie2 index of rRNA

Pre-build index is available in our Github repository:

```bash
#Human rRNA index
wget https://github.com/PrinceWang2018/Cen_rRNA/blob/master/Human_rRNA_bt2_index.tar.gz
wget https://github.com/PrinceWang2018/Cen_rRNA/blob/master/Mouse_rRNA_bt2_index.tar.gz
tar -zxvf Human_rRNA_bt2_index.tar.gz
tar -zxvf Mouse_rRNA_bt2_index.tar.gz
```

You can also build the index by yourself. First download rRNA sequences from NCBI according to the instructions:

> https://ucdavis-bioinformatics-training.github.io/2017-June-RNA-Seq-Workshop/wednesday/contamination.html

or our Github repository:

```bash
# Download appropriate rRNA sequences for your organism
wget https://github.com/PrinceWang2018/Cen_rRNA/blob/master/Human_rRNA_NCBI.fasta.gz
wget https://github.com/PrinceWang2018/Cen_rRNA/blob/master/Mouse_rRNA_NCBI.fasta.gz
gzip -d Human_rRNA_NCBI.fasta.gz
gzip -d Mouse_rRNA_NCBI.fasta.gz
```

Then build Bowtie2 index:

```bash
conda activate LACE2
bowtie2-build Human_rRNA_NCBI.fasta ./Human_rRNA_bt2_index/Human_rRNA
bowtie2-build Mouse_rRNA_NCBI.fasta ./Mouse_rRNA_bt2_index/Mouse_rRNA
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

Basic command:

```bash
conda activate LACE2
cd /path/to/work_directory #Containing 01.RawData as above
LACE2 \
  -w /path/to/work_directory \
  -s /path/to/STAR_index_directory \
  -r /path/to/rRNA_index_prefix \
  -t 16 \
  -p 0.001 \
  -b 20
```

Parameters:
- `-w`: Working directory containing 01.RawData folder (**Must use absolute path**)
- `-s`: Path to STAR index directory (Path)
- `-r`: Path to Bowtie2 rRNA index prefix (Path and index prefix)
- `-t`: Number of threads (default: 8)
- `-p`: Piranha -p parameter (default: 0.001)
- `-b`: Piranha -b parameter (default: 20)

## Demo Test

A small test dataset is available at Github. To run the demo:

```bash
conda activate LACE2
cd $LACE2_PATH/Demo_data
LACE2 \
  -w $LACE2_PATH/Demo_data \
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

For questions or issues, please open an issue on GitHub or contact 970214035yl@gmail.com, wangzixiang@sdu.edu.cn.
