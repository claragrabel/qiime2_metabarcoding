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

It is necessary to import raw sequencing data into QIIME2 to create a .qza (QIIME artifact) file. 

* Load the data into the server

```bash
scp -r /home/ecosystems/Desktop/clara/marco/data/ user@server:/home/bio/u80064476/ScpFiles/marco_data/
```

The command to import the data would be:

```bash
qiime tools import \
--type 'SampleData[PairedEndSequencesWithQuality]' \
--input-path manifest.tsv \
--input-format PairedEndFastqManifestPhred33V2 \
--output-path demux-paired-end.qza
```

#### Prepare a metadata file

The metadata file for QIIME2 must be a tab-delimited .tsv file and meet some requirement:

* Mandatory:
  + Headers: The first row must contain column names.
  + Sample IDs: The first column must include unique sample identifiers. This column is the only mandatory one; a file containing only sample IDs would be a valid QIIME 2 metadata file.
* Additional Columns: Additional columns can be included to provide group information (e.g., Treatment, Location, Timepoint).

Example Metadata File:
sample-id  treatment  location
sample1    control    siteA
sample2    treatment  siteB
sample3    control    siteB
sample4    treatment  siteA

See the guides for reference: 
https://docs.qiime2.org/2024.10/tutorials/metadata/
https://use.qiime2.org/en/latest/references/metadata.html


#### Validate the metadata file

There is an option in QIIME2 to validate and check if the metadata file is compatible.

```bash
qiime metadata tabulate \
  --m-input-file sample-metadata.tsv \
  --o-visualization metadata-summary.qzv
```

To view the .qzv file, there are two options:

* Upload it to QIIME 2 View: Drag and drop the file into the QIIME 2 View website at https://view.qiime2.org/.

* View locally: If the server supports displaying visualizations, execute the following command. This will automatically open the file in the default web browser:

```bash
qiime tools view demux-summary.qzv
```


## Step 2: Quality Check and Denoising

Since poor-quality reads can introduce errors in downstream analyses, it is necessary to assess the quality of the sequencing reads and decide where to trim or truncate low-quality regions.

### Quality Check

```bash
qiime demux summarize \
  --i-data demux-paired-end.qza \
  --o-visualization demux-summary.qzv
```

Based on the quality score plots from the .qzv file, which can be complemented with FASTQC quality reports, we need to make the following decisions:

* Trimming Low-Quality Bases at the Start:
Decide whether to trim low-quality bases at the start of the reads and the number of bases to trim using the --p-trim-left parameter.

* Truncating Reads:
Determine where to truncate reads using the --p-trunc-len parameter and how much to truncate.
+ Forward and reverse reads can be truncated at different lengths. Reverse reads typically have lower quality towards the end, so truncating them earlier is often beneficial. Just ensure there is sufficient overlap between forward and reverse reads for accurate merging during downstream analyses, typically at least 20–30 nucleotides.

* Choosing Between Paired-End and Single-End Reads:
If the quality of one read direction (forward or reverse) is significantly lower than the other, consider using only the higher-quality reads (single-end data).
While paired-end reads provide more information, using single-end reads may improve overall data quality if merging is problematic.

### Denoising

Denoising is a process used in bioinformatics workflows to clean raw sequencing data, removing errors and noise introduced during the sequencing process and reconstruct the true biological sequences (Amplicon Sequence Variants, or ASVs) by distinguishing real biological variation from sequencing artifacts.

In the context of QIIME2, DADA2 is the primary algorithm used for denoising (along with Deblur). It is designed to correct sequencing errors, remove low-quality data, and produce highly accurate ASVs with single-nucleotide resolution.

#### Key Steps in Denoising with DADA2 

* Quality Filtering and Trimming
Low-quality sequences are filtered out based on quality score thresholds.
Reads are trimmed at the start (--p-trim-left) to remove adapter sequences or low-quality bases, and truncated at the end.

* Error Modeling
DADA2 builds an error model for the dataset by analyzing the distribution of errors in the data.
The model predicts the probability of observing each base at a given position, allowing it to differentiate between true sequences and sequencing errors.

* Dereplication
Identical sequences within a sample are grouped together (dereplicated) to improve computational efficiency. Unique sequences are retained, but their frequencies are recorded too.

* Elimination of chimeric sequences
Chimeric sequences, which are artifacts resulting from the PCR process, are detected and removed.

* Merging Paired-End Reads
Forward and reverse reads are merged to reconstruct the full amplicon sequence.


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

_If single-end reads are chosen due to quality, use qiime dada2 denoise-single instead._

* Outputs of DADA2

    + Feature table (table.qza): Rows represent ASVs (exact Amplicon Sequence Variants at single-nucleotide resolution.), columns represent samples, and values indicate the frequency of each ASV in each sample.
    + Representative sequences (rep-seqs.qza): A list of the unique ASVs inferred by DADA2. Used for taxonomic classification and phylogenetic analyses.
    + Denoising statistics (denoising-stats.qza): A file that summarizes key statistics for each sample (nthe input reads, the number and percentage of reads that have been filtered and merged chimeric reads detected and removed, and the number of reads retained in the end)


Visualize:

```bash
 qiime feature-table summarize \
    --i-table table.qza \
    --o-visualization table.qzv
```


## Step 3: Clustering into OTUs

Since we aim to explore overall diversity and we do not need single-nucleotide resolution, we will cluster unique sequences into 97% OTUs.

```bash
qiime vsearch cluster-features-de-novo  \
  --i-sequences rep-seqs.qza  \
  --p-perc-identity 0.97  \
  --o-clustered-table otu-table.qza  \
  --o-clustered-sequences otu-rep-seqs.qza 
  --i-table table.qza
```

* Outputs

  + Feature table (otu-table.qza): Rows represent OTUs (clustered groups of sequences), columns represent samples nd values indicate the abundance of each OTU in each sample. This table is used for downstream diversity analyses, such as alpha and beta diversity.
  + Representative sequences (otu-rep-seqs.qza): A file containing representative sequences for each OTU. These sequences represent the centroid of each cluster (the most representative sequence for that OTU). Used for taxonomic classification and phylogenetic analyses.
  + table.qza (Optional Input): If provided, this input ensures that the feature table that contained the ASVs resulting from the denoising process is updated to reflect the clustered OTUs.


## Step 4: Taxonomy Assignment

This step assigns taxonomy to OTUs using a reference database (e.g., SILVA, Greengenes, or UNITE for fungal ITS). 
Since our primers are .../... (not 503F/806R), we use a classifier trained on the full-length 16S rRNA gene sequence to ensure compatibility.

Machine-learning classifiers (e.g., classify-sklearn) are now the trend for taxonomy assignment because they offer higher resolution, probabilistic predictions (assign taxonomy even for sequences with partial matches), and confidence scores that evaluate the reliability of each assignment.
Pre-trained classifiers, such as classifiers trained on the SILVA 138 or Greengenes 13_8 databases, are readily available for download in the QIIME2 Data Resources and save time compared to training a custom classifier.
Specifically, the classify-sklearn method uses the Naive Bayes algorithm to predict taxonomy based on k-mer patterns in input sequences.

In contrast, VSEARCH assigns taxonomy by aligning sequences to a reference database based on similarity thresholds (e.g., 97%). It assigns taxonomy to the top-scoring match or a consensus of multiple matches if specified. This approach lacks the fine resolution and confidence scoring provided by machine-learning classifiers.

We have to download the classifier from the QIIME2 resources page https://resources.qiime2.org/ and import the .qza file into the server. We will use either Greengenes2 2022.10 full length sequences or Silva 138 99% OTUs full-length sequences, which are compatible with QIIME2 versions from 2021.4 to 2024.2 (we are using qiime2-amplicon-2023.9).

Execute the command:

```bash
qiime feature-classifier classify-sklearn \
  --i-classifier gg_2022_10_backbone_full_length.nb.qza \
  --i-reads otu-rep-seqs.qza \
  --o-classification taxonomy.qza
```

we will actually run the command on the background using nohup (no hang up), so that it keeps on running even if the session is closed:

```bash
nohup qiime feature-classifier classify-sklearn \
  --i-classifier gg_2022_10_backbone_full_length.nb.qza \
  --i-reads otu-rep-seqs.qza \
  --o-classification taxonomy.qza > classify-sklearn.log 2>&1 &
```

Visualization of the Taxonomic classifications for OTUs:

```bash
qiime metadata tabulate \
  --m-input-file taxonomy.qza \
  --o-visualization taxonomy.qzv
```


## Step 5: Rarefaction (Optional)

Rarefaction involves subsampling sequences to a fixed depth across all samples. Some samples may appear to have a higher abundance of bacteria simply due to deeper sequencing, which can lead to misleading results. This process normalizes the number of sequences in each sample to account for differences in sequencing depth, mitigating bias introduced by uneven sequencing efforts.

* Why is rarefaction used for alpha diversity?
Alpha diversity metrics, particularly richness-based metrics like Observed OTUs or Chao1, are highly sensitive to sequencing depth. Rarefaction is used to address this sensitivity. Therefore, rarefaction is applied on the OTU feature table.

  + Richness bias:
Samples with higher sequencing depths tend to detect more rare species, inflating richness values.
Rarefaction ensures that all samples are evaluated at the same sequencing depth for fair comparisons.

  + Standardization across samples:
Differences in alpha diversity will reflect true biological variation, not sequencing depth differences.

  + Statistical comparisons:
Rarefaction prevents sequencing depth from acting as a confounding variable in group comparisons.

* Why is rarefaction performed after OTU generation?
Rarefaction is applied after OTU generation because OTU clustering relies on the complete dataset to accurately detect and classify species. Performing rarefaction before OTU generation would artificially reduce sequencing depth, leading to loss of rare OTUs and reduced accuracy in clustering.
Therefore, rarefaction is applied after generating a feature table (abundance table), which represents the number of sequences (features/OTUs) per sample.

* How to decide the rarefaction depth
We have to look at the minimum and maximum sequencing depth (22431-49046).
For example, it is possible to use the 25th or 50th percentile of sequencing depths as a guideline, ensuring the depth is sufficient to retain at least 75% of samples.
In our case, since we have few samples, we will choose the lowest depth (22431) to retain all samples.

* Determining sufficiency: Is rarefaction at this depth enough?

To know if the sequences threshold that we have chosen is enough to retain diversity information or if we are losing too much information, we can examine the rarefaction curves. Rarefaction curves illustrate how diversity metrics stabilize as sequencing depth increases.

Generate Rarefaction Curves:
```bash
qiime diversity alpha-rarefaction \
  --i-table otu-table.qza \
  --p-max-depth 22431 \
  --m-metadata-file metadata.tsv \
  --o-visualization alpha-rarefaction.qzv
```
+ If the curves plateau below the number of sequences that we have chosen as the threshold, this depth is sufficient to retain diversity information.
+ If not, rarefying at this depth may result in loss of biological diversity information.

```bash
qiime feature-table rarefy \
  --i-table otu-table.qza \
  --p-sampling-depth 22431 \
  --o-rarefied-table rarefied-otu-table.qza
```
Visualize:

```bash
 qiime feature-table summarize \
    --i-table rarefied-otu-table.qza \
    --o-visualization rarefied-otu-table.qzv
```

## Step 6: Diversity Metrics

### Alpha Diversity

Calculate metrics like richness (Observed OTUs) and evenness (Shannon, Simpson) in each sample using the rarefied table.

#### Concepts
* Richness: The total number of species (or units, such as OTUs or ASVs) in a sample.
* Evenness: How evenly individuals are distributed among the species. If one species dominates, evenness will be low.

#### Indexes
  + Shannon Index: Combines richness and evenness.
  + Simpson Index: Focuses on evenness.
  + Observed OTUs: Counts the number of unique OTUs.

Observed OUTs:
```bash
qiime diversity alpha \
  --i-table rarefied-otu-table.qza \
  --p-metric observed_features \
  --o-alpha-diversity observed-otus.qza
```

Visualize: 
```bash
qiime metadata tabulate \
  --m-input-file observed-otus.qza \
  --o-visualization observed-otus.qzv
```

*_The Shannon Index or Shannon entropy in QIIME2 is calculated using log2 instead of ln. Therefore, instead of values between 1.5-3.5, we can expect values in the range of 6-8._

Shannon index:
```bash
qiime diversity alpha \
  --i-table rarefied-otu-table.qza \
  --p-metric shannon \
  --o-alpha-diversity shannon.qza
```

Visualize:
```bash
qiime metadata tabulate \
  --m-input-file shannon.qza \
  --o-visualization shannon.qzv
```

### Beta Diversity

Calculate and visualize differences in community composition between samples.
Beta diversity calculates differences in species composition and abundance between communities. It responds to questions such as: How similar are the comunities of the different samples? Are there patterns that group comunities according to certain factors? 
#### Concepts
* Bray-Curtis: Considers differences in abundance.
* Jaccard: Considers only presence/absence.

#### Bray-Curtis

The Bray-Curtis dissimilarity metric considers both presence/absence and abundance of features (e.g., OTUs or ASVs) and calculates pairwise distances between samples based on their community composition.
Output is a distance matrix, where each cell represents the dissimilarity between two samples.

```bash
qiime diversity beta \
  --i-table rarefied-otu-table.qza \
  --p-metric braycurtis \
  --o-distance-matrix braycurtis-distance.qza
```

##### Principal Coordinates Analysis (PCoA)

Takes a distance matrix (e.g., from Bray-Curtis) as input and reduces its dimensionality. It represents samples in a lower-dimensional space, usually 2D or 3D, while preserving the pairwise distances as much as possible.
The reduced-dimensional space highlights patterns or clusters in the data.

Basically, it helps us visualize beta diversity results to explore relationships between samples and identify patterns.

```bash
qiime diversity pcoa \
  --i-distance-matrix braycurtis-distance.qza \
  --o-pcoa braycurtis-pcoa.qza
```

PCoA Visualization:
```bash
qiime emperor plot \
  --i-pcoa braycurtis-pcoa.qza \
  --m-metadata-file sample-metadata.tsv \
  --o-visualization braycurtis-emperor.qzv

```

## Step 7: Statistical Testing

Test for significant differences in diversity metrics across groups.

* Alpha Diversity Group Significance

The following code tests whether alpha diversity metrics (e.g., richness or evenness) differ significantly between groups using the Kruskal-Wallis test. This non-parametric test compares group medians and is well-suited for non-normally distributed data, common in microbial community analyses. The output is a .qzv visualization with boxplots and p-values indicating the statistical significance of group differences.

```bash
qiime diversity alpha-group-significance \
  --i-alpha-diversity observed-otus.qza \
  --m-metadata-file sample-metadata.tsv \
  --o-visualization observed-otus-group-significance.qzv
```

* Beta Diversity Group Significance (PERMANOVA):

PERMANOVA (Permutational Multivariate Analysis of Variance) tests whether the variation in a distance matrix is linked to group differences. It compares within-group distances to between-group distances to see if groups are more distinct than expected by chance. The test uses permutations (default: 999) to calculate significance.

```bash
qiime diversity beta-group-significance \
  --i-distance-matrix braycurtis-distance.qza \
  --m-metadata-file sample-metadata.tsv \
  --m-metadata-column <group-column> \
  --p-method permanova \
  --o-visualization braycurtis-group-significance.qzv
```
