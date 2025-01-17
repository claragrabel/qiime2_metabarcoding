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

The manifest file is a tab-separated values (TSV) file that lists:

* Sample IDs
* File paths for both forward and reverse sequences

It would look 

First, we import raw sequencing data into QIIME2 to create a .qza (QIIME artifact) file. QIIME works with .qza (data files) and .qzv (visualization files). .qza files can be easily converted into .qzv files for visualization.

If we take a look at the data, the samples are already demultiplexed. This means that each sample is separated in a different file, not all sequences are grouped together in the same file with different barcodes that identify the sample from where the sequences come.
Also, the samples are paired-end, since we have a forward and a reverse sample.
Therefore are going to use the  PairedEndSequencesWithQuality importing format. 
For this format, we need to prepare a manifest file that states the location/paths of all of our samples, because the import tool requires the input paths.

    1. Prepare a manifest file: Create a manifest.tsv that lists all your sample IDs, file paths, and read directions: