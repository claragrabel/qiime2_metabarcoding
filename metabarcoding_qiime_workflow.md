# Workflow for Biodiversity Analysis Using OTUs in QIIME2

## Overview

QIIME2 uses two main file formats:

* .qza: Data files (QIIME artifacts)

* .qzv: Visualization files

These formats allow seamless data processing and visualization inside QIIME2. 

## Understanding the Dataset

The data is already demultiplexed, meaning each sample is stored in a separate file. This contrasts with multiplexed data, where all sequences are grouped in a single file with barcodes to identify the source sample.

Moreover, the dataset contains paired-end reads, consisting of forward and reverse reads for each sample.

## Step 1: Data Import

First, we import raw sequencing data into a QIIME2 .qza (QIIME artifact) file that can be handled by QIIME2.

To import paired-end data that is already demultiplexed, we will use the PairedEndSequencesWithQuality format. This requires creating a manifest file that specifies the file paths of all samples.

 #### Prepare a Manifest File

The manifest file is a tab-separated values (TSV) file that lists in three columns:

* Sample IDs
* Absolute file paths for forward sequences
* Absolute file paths for forward reverse 

It would look like this:

| sample-id    | file-path-forward | file-reverse |
| -------- | ------- | -------- |
| sample_1  | path/file/    | path/file/
| sample_2 | path/file/     | path/file/
| sample_3    | path/file/   | path/file/

It is necessary to import raw sequencing data into QIIME2 to create a .qza (QIIME artifact) file. QIIME works with .qza (data files) and .qzv (visualization files). .qza files can be easily converted into .qzv files for visualization.

* Load the data into the server

```bash
scp -r /home/ecosystems/Desktop/clara/marco/data/ u80064476@urania01:/home/bio/u80064476/ScpFiles/marco_data/
```

The command to import the data would be:

```bash
       qiime tools import \
         --type 'SampleData[PairedEndSequencesWithQuality]' \
         --input-path manifest.tsv \
         --input-format PairedEndFastqManifestPhred33V2 \
         --output-path demux-paired-end.qza
```

## Step 2: Quality Check

Since poor-quality reads can introduce errors in downstream analyses, it is necessary to evaluate the quality of the sequencing reads and decide where to trim or truncate low-quality regions.

    • Trimming: Removing low-quality bases at the beginning of the read.
    • Truncating: Cutting off reads at a certain length when quality drops significantly.

Checking the quality:

```bash
qiime demux summarize \
  --i-data demux-paired-end.qza \
  --o-visualization demux-summary.qzv
```
To view the .qzv file, you have two options:

* Upload it to QIIME 2 View: drag and drop the file into the QIIME 2 View website at https://view.qiime2.org/.

* View locally: If the server supports displaying visualizations, execute the following command. This will automatically open the file in the default web browser:

```bash
qiime tools view demux-summary.qzv
```

Based on the quality score plots from the .qzv file, we need to make the following decisions:

* Trimming Low-Quality Bases at the Start:
Decide whether to trim low-quality bases at the start of the reads and the number of bases to trim using the --p-trim-left parameter .

* Truncating Reads:
Determine where to truncate reads using the --p-trunc-len parameter and how much to truncate.
+ Forward and reverse reads can be truncated at different lengths. Reverse reads typically have lower quality towards the end, so truncating them earlier is often beneficial. Just ensure there is sufficient overlap between forward and reverse reads for accurate merging during downstream analyses, typically at least 20–30 nucleotides.

* Choosing Between Paired-End and Single-End Reads:
If the quality of one read direction (forward or reverse) is significantly lower than the other, consider using only the higher-quality reads (single-end data).
While paired-end reads provide more information, using single-end reads may improve overall data quality if merging is problematic.

## Step 3: Denoising

qiime dada2 denoise-paired \
  --i-demultiplexed-seqs demux-paired-end.qza \
  --p-trim-left-f <trim-left-forward> \
  --p-trim-left-r <trim-left-reverse> \
  --p-trunc-len-f <trunc-length-forward> \
  --p-trunc-len-r <trunc-length-reverse> \
  --o-table table.qza \
  --o-representative-sequences rep-seqs.qza \
  --o-denoising-stats denoising-stats.qza


If single-end reads are chosen due to quality, use qiime dada2 denoise-single instead.