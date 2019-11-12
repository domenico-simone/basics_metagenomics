# QC tutorial

<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [A little warning before starting](#a-little-warning-before-starting)
- [Create personal folder and copy (link) data](#create-personal-folder-and-copy-link-data)
- [FastQC on raw data](#fastqc-on-raw-data)
- [Aggregate FastQC reports with MultiQC](#aggregate-fastqc-reports-with-multiqc)
- [Quality filtering: Trimmomatic](#quality-filtering-trimmomatic)
- [Aggregate FastQC and Trimmomatic reports with MultiQC](#aggregate-fastqc-and-trimmomatic-reports-with-multiqc)
- [FastQC on filtered data](#fastqc-on-filtered-data)
- [Aggregate Trimmomatic, raw/processed FastQC reports with MultiQC](#aggregate-trimmomatic-rawprocessed-fastqc-reports-with-multiqc)
- [Compare the metagenomes: SourMash](#compare-the-metagenomes-sourmash)

<!-- /TOC -->

## A little warning before starting

This tutorial has been crafted to give you the opportunity to copy/paste commands as much as possible, in order to get a bit familiar with the command line without getting mad about it! This said, **please check every block of code** you're going to run, since some lines **need** to be customized.

## Create personal folder and copy (link) data 

This is your safe space where you'll perform your analyses (related to ALL tutorials) without interfering with your classmates. We **softlink** the data without creating actual copies - although we have 2TB of storage in the course project on UPPMAX, it's good practice to save space whenever possible :-).

**Note**: replace `YOURNAME` with your name through this tutorial.

```bash
cd /proj/g2019027/2019_MG_course/student_folders
mkdir -p YOURNAME/qc_tutorial
cd YOURNAME/qc_tutorial

mkdir raw_data && cd raw_data
ln -sf /proj/g2019027/2019_MG_course/qc_tutorial/*.*.fastq.gz .
```

## FastQC on raw data

FastQC is not multithreaded, but it can process as many files at the same time as the number provided to the option `-t` (but make sure you have 250MB of memory/file).

```bash
# go to working directory
cd /proj/g2019027/2019_MG_course/student_folders/YOURNAME/qc_tutorial

# start interactive session on UPPMAX
interactive -p core -n 16 -t 1:00:00 --reservation=g2019027_12 -A g2019027 

# create output directories
mkdir -p fastqc_raw
mkdir -p logs

# load and run fastqc
module load bioinfo-tools
module load FastQC
cd raw_data
fastqc -o ../fastqc_raw/ -t 16 $(ls ????_?.R*.fastq.gz)

# copy logs/outputs in appropriate folder
cp ../fastqc_raw/*_fastqc.html ../logs
cp ../fastqc_raw/*_fastqc.zip ../logs
```

## Aggregate FastQC reports with MultiQC

What would you do with your results now? You have 16 x 2 = 32 results to go through. You could open them one by one, or find a way to display all of them at once. **MultiQC** is a tool capable of parsing outputs and logs of a plethora of tools (not only QC-related) and collapse them to make easier to get an overview of results from multiple datasets. During this tutorial, we'll use this tool three times to collapse reports from FastQC and quality filtering stats. To this purpose, we have created a folder `logs` where we'll copy FastQC reports and Trimmomatic logs.

```bash
# go to working directory
cd /proj/g2019027/2019_MG_course/student_folders/YOURNAME/qc_tutorial

# load MultiQC
module load MultiQC/1.0

# go into the folder with the logs
cd logs

# run MultiQC
multiqc -n overview_1 .
```

Examine the report. Do you think it's consistent with a metagenomic dataset?

## Quality filtering: Trimmomatic

The command to be run is not contained in a script, but it's encapsulated in a special syntax called **heredoc** (between the two `EOF`s). This clearly shows the command you're running, so it might be useful for documentation purposes.

```bash
# go to working directory
cd /proj/g2019027/2019_MG_course/student_folders/YOURNAME/qc_tutorial/raw_data

#Here starts the heredoc

sbatch -p core -t 1:00:00 --reservation=g2019027_12 -A g2019027 \
--array=1-$(wc -l < datasets) -J trimmomatic<<'EOF'
#!/bin/sh

module load bioinfo-tools
module load trimmomatic

SAMPLE=$(sed -n "$SLURM_ARRAY_TASK_ID"p datasets)

mkdir -p ../processed_data
mkdir -p ../logs

trimmomatic PE -phred33 -threads 1 \
${SAMPLE}.R1.fastq.gz \
${SAMPLE}.R2.fastq.gz \
${SAMPLE}.R1.trim.fastq.gz \
${SAMPLE}.U1.trim.fastq.gz \
${SAMPLE}.R2.trim.fastq.gz \
${SAMPLE}.U2.trim.fastq.gz \
ILLUMINACLIP:/sw/apps/bioinfo/trimmomatic/0.36/rackham/adapters/TruSeq3-SE.fa:2:30:10 \
LEADING:3 TRAILING:3 SLIDINGWINDOW:4:15 MINLEN:36 &> ${SAMPLE}.trim.log

# copy outputs to relevant files
mv ${SAMPLE}.*.trim.fastq.gz ../processed_data

# copy logs to log folder for MultiQC
mv ${SAMPLE}.trim.log ../logs

EOF
```

## Aggregate FastQC and Trimmomatic reports with MultiQC

Now the `logs` directory includes also the Trimmomatic reports. What happens if we run MultiQC now?

```bash
# go to log directory
cd /proj/g2019027/2019_MG_course/student_folders/YOURNAME/qc_tutorial/logs

module load MultiQC/1.0

multiqc -n overview_2 .
```

Examine the report. What is the proportion of discarded reads? Do you think we should use the unpaired reads for downstream analyses?

## FastQC on filtered data

Let's run FastQC on filtered data.

```bash
#interactive -p core -n 16 -t 1:00:00 --reservation=g2019027_<day>
interactive -p core -n 16 -t 1:00:00 --reservation=g2019027_12 -A g2019027

# go to working directory
cd /proj/g2019027/2019_MG_course/student_folders/YOURNAME/qc_tutorial/processed_data

module load bioinfo-tools
module load FastQC

#create output directory
mkdir -p ../fastqc_processed

fastqc \
-o ../fastqc_processed/ \
-t 16 \
$(ls ????_?.R*.trim.fastq.gz) # list of input files 

# copy logs/outputs in appropriate folder
cp ../fastqc_processed/*_fastqc.html ../logs
cp ../fastqc_processed/*_fastqc.zip ../logs
```

## Aggregate Trimmomatic, raw/processed FastQC reports with MultiQC

Let's aggregate everything! What are your comments about the datasets and their processing?

```bash
module load MultiQC/1.0

cd ../logs

multiqc -n overview_3 .
```

## Compare the metagenomes: SourMash

Install sourmash through python3 & pip.  



```bash
module load python3
pip3 install --user sourmash
```

Create signatures for each metagenome

```bash

mkdir -p signatures
cd processed_data

sbatch -p core -t 30:00 --reservation=g2019027_12 -A g2019027 \
--array=1-$(wc -l < ../raw_data/datasets) -J sourmash<<'EOF'
#!/bin/sh

SAMPLE=$(sed -n "$SLURM_ARRAY_TASK_ID"p ../raw_data/datasets)

$HOME/.local/bin/sourmash compute \
--scaled 10000 \
${SAMPLE}.R1.trim.fastq.gz \
-o ${SAMPLE}.R1.trim.sig \
-k 31

mv ${SAMPLE}.R1.trim.sig ../signatures

EOF
```

Compare signatures!

```bash
cd /proj/g2019027/2019_MG_course/student_folders/YOURNAME/qc_tutorial/signatures

$HOME/.local/bin/sourmash compare -k 31 *.sig -o biofilm_cmp
$HOME/.local/bin/sourmash plot --pdf --labels biofilm_cmp
```

Display the distance matrix + dendrogram. Is the sample clustering expected? 

```bash
firefox biofilm_cmp.matrix.pdf
```