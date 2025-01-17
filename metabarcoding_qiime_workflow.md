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

First, we import raw sequencing data into QIIME2 to create a .qza (QIIME artifact) file. QIIME works with .qza (data files) and .qzv (visualization files). .qza files can be easily converted into .qzv files for visualization.

