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

| sample-id    | file-path-forward | file-path-reverse |
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

## Step 2: Quality Check and Denoising

Since poor-quality reads can introduce errors in downstream analyses, it is necessary to evaluate the quality of the sequencing reads and decide where to trim or truncate low-quality regions.

    • Trimming: Removing low-quality bases at the beginning of the read.
    • Truncating: Cutting off reads at a certain length when quality drops significantly.

### Quality Check

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

### Denoising

```bash
qiime dada2 denoise-paired \
  --i-demultiplexed-seqs demux-paired-end.qza \
  --p-trim-left-f <trim-left-forward> \
  --p-trim-left-r <trim-left-reverse> \
  --p-trunc-len-f <trunc-length-forward> \
  --p-trunc-len-r <trunc-length-reverse> \
  --o-table table.qza \
  --o-representative-sequences rep-seqs.qza \
  --o-denoising-stats denoising-stats.qza
```

If single-end reads are chosen due to quality, use qiime dada2 denoise-single instead.


## Step 4: Rarefaction (Optional)

Rarefaction involves subsampling sequences to a fixed depth across all samples. This process normalizes the number of sequences in each sample to account for differences in sequencing depth, mitigating bias introduced by uneven sequencing efforts.

Some samples may appear to have a higher abundance of bacteria simply due to deeper sequencing, which can lead to misleading results. Rarefaction ensures that diversity comparisons between samples are fair by standardizing sequencing depth.

* Why is rarefaction used for alpha diversity?
Alpha diversity metrics, particularly richness-based metrics like Observed OTUs or Chao1, are highly sensitive to sequencing depth. Rarefaction is used to address this sensitivity:

+ Richness Bias:
Samples with higher sequencing depths tend to detect more rare species, inflating richness values.
Rarefaction ensures that all samples are evaluated at the same sequencing depth for fair comparisons.

+ Standardization Across Samples:
Differences in alpha diversity will reflect true biological variation, not sequencing depth differences.

+ Statistical Comparisons:
Rarefaction prevents sequencing depth from acting as a confounding variable in group comparisons.

* Why is Rarefaction Performed After OTU Generation?
Rarefaction is applied after OTU generation because OTU clustering relies on the complete dataset to accurately detect and classify species. Performing rarefaction before OTU generation would artificially reduce sequencing depth, leading to loss of rare OTUs and reduced accuracy in clustering.
Therefore, rarefaction is applied after generating a feature table (abundance table), which represents the number of sequences (features/OTUs) per sample.

* How to Decide the Rarefaction Depth
We have to look at the minimum and maximum sequencing depth (56,514-111,936).
For example, it is possible to use the 25th or 50th percentile of sequencing depths as a guideline,ensuring the depth is sufficient to retain at least 75% of samples.
In our case, since we have few samples, we will choose the lowest depth (56,514) to retain all samples.

```bash
qiime feature-table rarefy \
  --i-table otu-table.qza \
  --p-sampling-depth 56514 \
  --o-rarefied-table rarefied-otu-table.qza
```

* Determining Sufficiency: Is Rarefaction at 56,514 Enough?

To know if the sequences threshold that we have chosen is enough to retain diversity information or if we are losing too much information, we can examine the rarefaction curves. Rarefaction curves illustrate how diversity metrics stabilize as sequencing depth increases.

Generate Rarefaction Curves:
```bash
qiime diversity alpha-rarefaction \
  --i-table otu-table.qza \
  --i-phylogeny rooted-tree.qza \
  --p-max-depth 56514 \
  --m-metadata-file sample-metadata.tsv \
  --o-visualization alpha-rarefaction.qzv
```
+ If the curves plateau below the number of sequences that we have chosen as the threshold, this depth is sufficient to retain diversity information.
+ If not, rarefying at this depth may result in loss of biological diversity information.

