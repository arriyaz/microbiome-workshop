## # Preparatory Steps

## Install Miniconda

```bash
mkdir -p ~/miniconda3

wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda3/miniconda.sh

bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3

rm -rf ~/miniconda3/miniconda.sh
```

## Second Option: Install Micromamba

```bash
# update and upgrade system
sudo apt update && sudo apt upgrade -y

# install curl
sudo apt install curl -y

# install micromamba
"${SHELL}" <(curl -L micro.mamba.pm/install.sh)
```

For details instruction on installing micromamba go to my github repository:[ bioinfo-pc-setup](https://github.com/arriyaz/bioinfo-pc-set-up.git).

## Install QIIME2

> If the channel priority in conda is set to **strict**, you have to make it **flexible** or **turn it off** to install QIIME2. 
> 
> Look for `.condarc` file in the home folder. Open it in a text editor and change the **channel_priority** to **flexible**. 
> 
> You can change it back to **strict** after installing QIIME2.

Download the installation file.

```bash
wget https://data.qiime2.org/distro/amplicon/qiime2-amplicon-2023.9-py38-linux-conda.yml
```

Create a conda environment and install Qiime2 in it.

```bash
conda env create -n qiime2-amplicon --file qiime2-amplicon-2023.9-py38-linux-conda.yml
```

**Note**: If you've isntalled micromamba use the `micromamba` command, instead of `conda`.

**Optional Cleanup:** remove the installation file:

```bash
rm qiime2-amplicon-2023.9-py38-linux-conda.yml
```

For more instruction go to qiime2 docs: [Natively installing QIIME 2](https://docs.qiime2.org/2023.9/install/native/#install-qiime-2-within-a-conda-environment)

## Activate conda environment

```bash
conda activate qiime2-amplicon
```

### Enable QIIM2 tab-completion option

Each time you activate the qiime2 environment you should also run the following command to enable tab-completion option for qiime2. It will make your life more easier.

```bash
source tab-qiime
```

## Test the installation

```bash
qiime --help
```

Let's check qiime2 and its plugin versions

```bash
qiime info
```

## Download the data for this workshop

We will work inside a directory under home folder.

> Everybody is advised to create same directory name for easy and consistent instruction and follow up.

Let's create the working directory for our workshop.

```bash
mkdir -p ~/metagenomics-workshop
```

Now go to the following link to download the data in zip format and save it inside the **metagenomics-workshop** directory.

Link: [16s-data.zip - Google Drive](https://drive.google.com/file/d/168gvq2gpN5QXUY4nJcjllhbFQVZH0-Lx/view?usp=drive_link)

:point_right: :point_right: :point_right: :point_right:  

## Check an artifact

```bash
mkdir -p ~/metagenomics-workshop/temp


# Download an artifact
curl -sL \
    "https://docs.qiime2.org/2020.11/data/tutorials/moving-pictures/rep-seqs.qza" \
    > ./temp/sequences.qza

# Unzip
unzip  -o ./temp/sequences.qza -d ./temp/

# Unzip or extract with QIIME2 plugin
qiime tools extract --input-path ./temp/sequences.qza --output-path ./temp/
```

If you want to export an artifact to the relevant format.

```bash
# export
qiime tools export \
    --input-path ./temp/sequences.qza \
    --output-path ./temp/exported-sequences
```

:point_right: :point_right: :point_right: :point_right:

## Check available plugins

```bash
qiime --help
```

Check actions under diversity plugin:

```bash
qiime diversity --help
```

:point_right: :point_right: :point_right: :point_right: 

# Redo: Reconstitution of the gut microbiota of antibiotic-treated patients by autologous fecal microbiota transplant

  doi: 10.1126/scitranslmed.aap9489 

```bash
# Create necessary folders
mkdir -p qzv qza others results
```

This metadata file is the compilation of ~12.5k data from several studies.

The title of this study: **"[Compilation of longitudinal microbiota data and hospitalome from hematopoietic cell transplantation patients](https://doi.org/10.1038/s41597-021-00860-8)"**

```bash
# Download the metadata
wget \
  -O './others/compilation-metadata.tsv' \
  'https://docs.qiime2.org/jupyterbooks/cancer-microbiome-intervention-tutorial/data/020-tutorial-upstream/020-metadata/sample-metadata.tsv'
```

## Prepare the metadata

> For ANCOM-BC method, we can't use the column names with **`+`** or **`-`** . Because this method use + and - as part of its formula. That's why, during preparing metadata we should avoid using **`-`** as word separator. Best approach is following **camelCase** naming convention.

From this large metadata we will only use 41 samples from the study by Taury *et al.*, 2018.

```bash
# Extract the sample ids from filenames
ls data/ | sort | cut -d '_' -f 1 | uniq -d > ./others/sampleids.txt
```

At first extract the header lines from the compilation metadata file

```bash
head -2 ./others/compilation-metadata.tsv > ./others/metadata.tsv
```

Now we can use this list to extract metadata for our target ids.

```bash
awk 'NR==FNR{
    a[$1];
    next
    } $1 in a
' ./others/sampleids.txt ./others/compilation-metadata.tsv \
>> ./others/metadata.tsv
```

Validate the metadata

```bash
qiime metadata tabulate \
  --m-input-file ./others/metadata.tsv \
  --o-visualization ./qzv/metadata.qzv
```

Go to https://view.qiime2.org/  and upload the `qvz` file to view it.

[optional for this workshop] If you want to download `fastq` file from SRA database:

```bash
for file in $(cat ./others/SRAids.txt)
do
parallel-fastq-dump --sra-id $file --threads 30 --outdir ./data/ --split-files --gzip 
done
```

:point_right: :point_right: :point_right: :point_right: 

## Prepare the Manifest File

```bash
# get absolute file path for forward and reverse reads.
find $PWD/data/ -type f | sort | grep -E *R1_001.fastq.gz > ./temp/forward-paths.txt
find $PWD/data/ -type f | sort | grep -E *R2_001.fastq.gz > ./temp/reverse-paths.txt
```

To varify the manifest file preparation we can use following tricks.

```bash
for id in $(cat ./others/sampleids.txt)
do
grep $id ./others/manifest.tsv
echo
sleep 1s
done
```

:point_right: :point_right: :point_right: :point_right:  

## Import Data

### Check importable types

```bash
qiime tools list-types  --tsv
```

:loudspeaker: Let's save the importable types list in a tsv file.

### Check importable formats

```bash
qiime tools list-formats --importable --tsv
```

:loudspeaker: Let's save the importable format list in a tsv file.

:loudspeaker: Let's save the exportable format list in a tsv file.

:point_right: :point_right: :point_right: :point_right: 

### Understand your data format

First check the format (Phred score version) of the data. Run the data with `fastqc` tool. 

> Ensure that conda has `fastqc` installed, prefereabley in a different environment. 
> 
> You can install `fastqc` by running `conda install bioconda::fastqc` in the target environment.

Run fastqc for some files.

```bash
fastqc ./data/FMT.0093D_2_L001_R1_001.fastq.gz -o ./temp/
```

```bash
fastqc ./data/FMT.0106I_64_L001_R2_001.fastq.gz -o ./temp/
```

Let's import the data.

```bash
qiime tools import \
  --type 'SampleData[PairedEndSequencesWithQuality]' \
  --input-path ./others/manifest.tsv \
  --input-format PairedEndFastqManifestPhred33V2 \
  --output-path ./qza/paired-end-demux.qza
```

## Summary of the demultiplexed results

```bash
qiime demux summarize \
  --i-data ./qza/paired-end-demux.qza \
  --p-n 10000 \
  --o-visualization ./qzv/demux-summary.qzv
```

Let's view the sequence summary from `demuz.qzv` file.

:point_right: :point_right: :point_right: :point_right:

## Denoising and Clustering for Sequence QC and Feature Table Construction

Let's at first check is truncation length is suitable for merging the reads:

First crate an empty `.sh` file inside `others` folder.

```bash
touch ./others/merging-check.sh
```

Now open the `merging-check.sh` file in text editor, copy the following code and save it. 

Change the values as per your preference and check if merging possible.

```bash
F_primer_start=563
R_primer_start=926

min_overlap=12
F_read_trunc_len=203
R_read_trunc_len=200

F_read_end=$(( $F_primer_start + $F_read_trunc_len - 1 ))
R_read_end=$(( $R_primer_start - $R_read_trunc_len + 1 ))

overlap=$(expr $F_read_end - $R_read_end + 1)

if [ $overlap -ge $min_overlap ]
then
    echo
    echo "Atleast $overlap bp overlapping"
    echo "Merging possible!!"
    echo
else
    echo
    echo "Only $overlap bp overlapping"
    echo "No marging possible!!"
    echo
fi
```

Now to run the bash script:

```bash
bash ./others/merging-check.sh
```

Run DADA2. It will perform:

- quality filtering

- Denoising

- chimera checking

- paired-end read joining and ASV generation

```bash
qiime dada2 denoise-paired \
    --i-demultiplexed-seqs ./qza/paired-end-demux.qza \
    --p-trunc-len-f 203 \
    --p-trunc-len-r 200 \
    --p-trim-left-f 0 \
    --p-trim-left-r 0 \
    --p-min-overlap 12 \
    --p-n-threads 30 \
    --verbose \
    --o-representative-sequences ./qza/asv-rep-seqs.qza \
    --o-table ./qza/feature-table.qza \
    --o-denoising-stats ./qza/denoise-stats.qza
```

Let's generate visualizations from DADA2 outputs to check the summaries.

Summarize dada2 denoising stats. This file will show us how many reads were filtered in each step.

```bash
qiime metadata tabulate \
    --m-input-file ./qza/denoise-stats.qza \
    --o-visualization ./qzv/dada2-stats-summary.qzv
```

Next, to view feature-table summary. The feature table describes which amplicon sequence variants (ASVs) were observed in which samples, and how many times each ASV was observed in each sample.

```bash
qiime feature-table summarize \
    --i-table ./qza/feature-table.qza \
    --m-sample-metadata-file ./others/metadata.tsv \
    --o-visualization ./qzv/feature-table-summary.qzv
```

We can also view the representative ASV sequence (feature data) for each feature.

```bash
qiime feature-table tabulate-seqs \
    --i-data ./qza/asv-rep-seqs.qza \
    --o-visualization ./qzv/asv-rep-seqs-summary.qzv
```

Let's extract the feature-table as a tsv file.

First export the table artifact into biom format:

```bash
qiime tools export \
	--input-path ./qza/feature-table.qza \
	--output-path ./temp/
```

Convert the biom file to tsv file:

```bash
biom convert \
    -i ./temp/feature-table.biom \
    -o temp/feature-table.tsv \
    --to-tsv
```

:point_right: :point_right: :point_right: :point_right: 

## Prepare Taxonomy Classifier

We will use the latest release of Greengenes database: **Greengenes2 2022.10**

This release is quite a bit different than previous release of Greengenes.

To learn more about this release go to this discussion from its developer: **[Introducing Greengenes2 2022.10](https://forum.qiime2.org/t/introducing-greengenes2-2022-10/25291)**

The ftp link of this release: **https://ftp.microbio.me/greengenes_release/2022.10/**

> Normally Qiime2 has a dedicated plugin `q2-feature-classifier` for training taxonomy classifier for specific data. To learn more, go to this link: [Training feature classifiers with q2-feature-classifier](https://docs.qiime2.org/2023.9/tutorials/feature-classifier/). But, **Greengenes2** have its own plugin and a bit different approach for preparing classifier. Hopefully, QIIME2 will update its documentation in future for **Greengenes2**.

> In this study, following primers were used to amplify the region between V4 and V5 of 16s rRNA gene:
>
> 563F: AYTGGGYDTAAAGNG
>
> 926R: CCGTCAATTYHTTTRAGT

For this workshop, we will follow new approach.

### Install greengenes2 plugin

If the **qiime2-amplicon** environment is not activated, at activate the environment.

And, don't forget to  enable QIIM2 tab-completion option by running `source tab-qiime`.

Now, let's install greengene2 plugin for QIIME2

```bash
pip install q2-greengenes2
```

Now restart your terminal and reactivate the qiime2 environment. 

### Download reference data

Create a directory for classifier data

```bash
mkdir -p ~/metagenomics-workshop/classifier
```

Download reference sequence for full-length 16s rRNA.

```bash
wget https://ftp.microbio.me/greengenes_release/2022.10/2022.10.backbone.full-length.fna.qza --no-check-certificate -P ./classifier
```

Download reference taxonomy data for full-length 16s rRNA.

```bash
wget https://ftp.microbio.me/greengenes_release/2022.10/2022.10.taxonomy.asv.nwk.qza --no-check-certificate -P ./classifier
```

:point_right: :point_right: :point_right: :point_right: 

### Map our data to reference

Now we will map our data with reference to create taxonomic classifier for our data.

```bash
qiime greengenes2 non-v4-16s \
    --i-table ./qza/feature-table.qza \
    --i-sequences ./qza/asv-rep-seqs.qza \
    --i-backbone ./classifier/2022.10.backbone.full-length.fna.qza \
    --p-threads 30 \
    --o-mapped-table ./classifier/gg2-2022.10-v4-v5.feature-table.biom.qza \
    --o-representatives ./classifier/gg2-2022.10-v4-v5.rep-seqs.fna.qza
```

:point_right: :point_right: :point_right: :point_right: 

Create feature table summary from greengenes biom table:

> The terms **biom** and **features** denote kind off similar meaning.

```bash
qiime feature-table summarize \
    --i-table ./classifier/gg2-2022.10-v4-v5.feature-table.biom.qza \
    --m-sample-metadata-file ./others/metadata.tsv \
    --o-visualization ./qzv/gg2-biom-table-summary.qzv
```

Similarly, create summary for representative sequence of each feature:

```bash
qiime feature-table tabulate-seqs \
    --i-data ./classifier/gg2-2022.10-v4-v5.rep-seqs.fna.qza \
    --o-visualization ./qzv/gg2-rep-seqs-summary.qzv
```



**[optional:]** If you want to convert the biom (binary format) table to tsv file for other purposes you can convert it by following commands:

First export the table artifact into biom format:

```bash
qiime tools export \
	--input-path ./classifier/gg2-2022.10-v4-v5.biom.qza \
	--output-path ./temp/
```

Convert the biom file to tsv file:

```bash
biom convert \
    -i ./temp/feature-table.biom \
    -o temp/feature-table.tsv \
    --to-tsv
```

## Perform taxonomic classification

```bash
qiime greengenes2 taxonomy-from-table \
	--i-reference-taxonomy ./classifier/2022.10.taxonomy.asv.nwk.qza \
	--i-table ./classifier/gg2-2022.10-v4-v5.feature-table.biom.qza \
	--o-classification ./qza/taxonomy.qza
```

### Create taxonomy visualization

```bash
qiime metadata tabulate \
	--m-input-file ./qza/taxonomy.qza \
	--o-visualization ./qzv/taxonomy.qzv
```

### Create barplot for taxonomic composition

```bash
qiime taxa barplot \
	--i-table ./classifier/gg2-2022.10-v4-v5.feature-table.biom.qza \
	--i-taxonomy ./qza/taxonomy.qza \
	--m-metadata-file ./others/metadata.tsv \
	--o-visualization ./qzv/taxa_barplot.qzv
```

## Generate a tree for phylogenetic diversity analyses

Several alpha and beta diversity metrics in QIIME2 require a rooted phylogenetic tree relating the features to one another. In this case there are two approach in QIIME2: 1. reference-based fragment insertion approach, 2. A *de novo* approach.

To know more about phylogeny in QIIME2 visit this link: [Phylogenetic inference with q2-phylogeny](https://docs.qiime2.org/2023.9/tutorials/phylogeny/)

We will use the *de novo* approach where we have to do the following steps:

1. Generate a multiple sequence alignment within QIIME2
2. Mask the alignment if need (to get rid of noise)
3. Construct phylogenetic tree
4. Root the phylogenetic tree

For multiple sequence alignment QIIME2 uses **MAFFT**.

There are three methods available in QIIME2 to construct  phylogenetic tree:

1. FastTree
2. RAxML
3. IQ-Tree

We will use `align-to-tree-mafft-fasttree` pipeline to do all the steps in on run:

```bash
qiime phylogeny align-to-tree-mafft-fasttree \
	--i-sequences ./classifier/gg2-2022.10-v4-v5.rep-seqs.fna.qza \
	--o-alignment ./qza/aligned-rep-seqs.qza \
	--o-masked-alignment ./qza/masked-aligned-rep-seqs.qza \
	--o-tree ./qza/unrooted-tree.qza \
	--o-rooted-tree ./qza/rooted-tree.qza
```

:point_right: :point_right: :point_right: :point_right: 

## Alpha Rarefaction Plot

Let's view an example: [Alpha rarefaction plot from moving picture tutorial](https://view.qiime2.org/visualization/?type=html&src=https%3A%2F%2Fdocs.qiime2.org%2F2023.9%2Fdata%2Ftutorials%2Fmoving-pictures%2Falpha-rarefaction.qzv)

Now, let's generate alpha rarefaction curve for your data:

> To select the `--p-max-depth` value, check the `feature-table-summary.qzv` file and start from minimum frequency per sample. Inspect the trend in the plot. Try to increase the value in several steps and compare the plot.

```bash
qiime diversity alpha-rarefaction \
	--i-table ./classifier/gg2-2022.10-v4-v5.feature-table.biom.qza \
	--i-phylogeny ./qza/rooted-tree.qza \
	--p-max-depth 24392 \
	--m-metadata-file ./others/metadata.tsv \
	--o-visualization ./qzv/alpha-rarefaction.qzv
```

## Alpha and Beta Diversity Analysis

The `core-metrics-phylogenetic` pipeline under `q2-diversity` plugin calculates alpha and beta diversity based on 8 different metrics (4 for alpha diversity and 4 for beta diversity). It also performs principle coordinate analysis (PCoA) for each beta diversity metrics).

```bash
qiime diversity core-metrics-phylogenetic \
	--i-phylogeny ./qza/rooted-tree.qza \
	--i-table ./classifier/gg2-2022.10-v4-v5.feature-table.biom.qza \
	--p-sampling-depth 24392 \
	--m-metadata-file ./others/metadata.tsv \
	--p-n-jobs-or-threads 'auto' \
	--output-dir core-metrics-results
```

### Explore Alpha Diversity

Let's explore **alpha diversity** in the context of our sample metadata.

Observed Features (a qualitative measure of community richness):

```bash
qiime diversity alpha-group-significance \
	--i-alpha-diversity ./core-metrics-results/observed_features_vector.qza \
	--m-metadata-file ./others/metadata.tsv \
	--o-visualization ./qzv/observed-features-group-significance.qzv
```

Shannon’s diversity index (a quantitative measure of community richness):

```bash
qiime diversity alpha-group-significance \
	--i-alpha-diversity ./core-metrics-results/shannon_vector.qza \
	--m-metadata-file ./others/metadata.tsv \
	--o-visualization ./qzv/shannon-vector-group-significance.qzv
```

Faith’s Phylogenetic Diversity (a qualitative measure of community richness that incorporates phylogenetic relationships between the features):

```bash
qiime diversity alpha-group-significance \
	--i-alpha-diversity ./core-metrics-results/faith_pd_vector.qza \
	--m-metadata-file ./others/metadata.tsv \
	--o-visualization ./qzv/faith-pd-group-significance.qzv
```

Evenness (or Pielou’s Evenness; a measure of community evenness):

```bash
qiime diversity alpha-group-significance \
	--i-alpha-diversity ./core-metrics-results/evenness_vector.qza \
	--m-metadata-file ./others/metadata.tsv \
	--o-visualization ./qzv/evenness-vector-group-significance.qzv
```

The `alpha-group-significance` command performs **comparison** between different **categorical** groups in the metadata. That's why it excludes the numerical column from calculation. 

If you want to check **correlation** between alpha diversity and different **numerical** columns in the metadata then you can use the `alpha-correlation` command under `q2-diversity` plugin. 

For example, if you want to check alpha correlation for ***observed-features*** metric:

```bash
qiime diversity alpha-correlation \
	--i-alpha-diversity ./core-metrics-results/observed_features_vector.qza \
	--m-metadata-file ./others/metadata.tsv \
	--o-visualization ./qzv/observed-features-correlation.qzv
```

:loudspeaker: Check alpha correlation for other metrics.

### Explore Beta Diversity

Similarly we can explore **beta diversity** in the context of our sample metadata

Let's check beta diversity significance for the `categorical-time-relative-to-hct` column.

Jaccard distance (a qualitative measure of community dissimilarity):

```bash
qiime diversity beta-group-significance \
	--i-distance-matrix ./core-metrics-results/jaccard_distance_matrix.qza \
	--m-metadata-file ./others/metadata.tsv \
	--m-metadata-column categorical-time-relative-to-hct \
	--o-visualization ./qzv/jaccard_categorical-time-relative-to-hct_significance.qzv \
	--p-pairwise
```

Bray-Curtis distance (a quantitative measure of community dissimilarity):

```bash
qiime diversity beta-group-significance \
	--i-distance-matrix ./core-metrics-results/bray_curtis_distance_matrix.qza \
	--m-metadata-file ./others/metadata.tsv \
	--m-metadata-column categorical-time-relative-to-hct \
	--o-visualization ./qzv/bray-curtis_categorical-time-relative-to-hct_significance.qzv \
	--p-pairwise
```

Unweighted UniFrac distance (a qualitative measure of community dissimilarity that incorporates phylogenetic relationships between the features)

```bash
qiime diversity beta-group-significance \
	--i-distance-matrix ./core-metrics-results/unweighted_unifrac_distance_matrix.qza \
	--m-metadata-file ./others/metadata.tsv \
	--m-metadata-column categorical-time-relative-to-hct \
	--o-visualization ./qzv/unweighted-unifrac_categorical-time-relative-to-hct_significance.qzv \
	--p-pairwise
```

Weighted UniFrac distance (a quantitative measure of community dissimilarity that incorporates phylogenetic relationships between the features)

```bash
qiime diversity beta-group-significance \
	--i-distance-matrix ./core-metrics-results/weighted_unifrac_distance_matrix.qza \
	--m-metadata-file ./others/metadata.tsv \
	--m-metadata-column categorical-time-relative-to-hct \
	--o-visualization ./qzv/weighted-unifrac_categorical-time-relative-to-hct_significance.qzv \
	--p-pairwise
```

:loudspeaker: Check beta group significance for other categorical columns.

#### PCoA Plot:

The `core-metrics-phylogenetic` command generates visualizations for principle coordinate plots for different beta diversity metrics. 

Move all visualizations from **core-metrics-results** folder to **qzv** folder.

```bash
mv ./core-metrics-results/*.qzv ./qzv/
```

#### Time Series Analysis for Beta Diversity

We can also perform time series analysis for beta diversity metrics in the context of numerical columns in metadata.

Let's view some examples:

```bash
qiime emperor plot \
	--i-pcoa ./core-metrics-results/bray_curtis_pcoa_results.qza \
	--m-metadata-file ./others/metadata.tsv \
	--p-custom-axes week-relative-to-hct \
	--o-visualization ./qzv/bray-curtis_week-relative-to-hct_PCoA.qzv
```

```bash
qiime emperor plot \
	--i-pcoa ./core-metrics-results/weighted_unifrac_pcoa_results.qza \
	--m-metadata-file ./others/metadata.tsv \
	--p-custom-axes week-relative-to-fmt \
	--o-visualization ./qzv/weighted-unifrac_week-relative-to-fmt_PCoA.qzv
```

:loudspeaker: Check this command for other numerical columns and beta diversity metrics.

## Heatmap

### Step-1: Add Taxa Names in Feature Table

```bash
qiime taxa collapse \
	--i-table ./classifier/gg2-2022.10-v4-v5.feature-table.biom.qza \
	--i-taxonomy ./qza/taxonomy.qza \
	--p-level 7 \
	--o-collapsed-table ./qza/gg2-taxa_species-feature-table.qza
```

**[optional:]**

```bash
qiime feature-table summarize \
	--i-table ./qza/gg2-taxa_species-feature-table.qza \
	--m-sample-metadata-file ./others/metadata.tsv \
	--o-visualization ./qzv/gg2-taxa_species-feature-table.qzv
```



### Step-2: Filter Feature Table

```bash
qiime feature-table filter-features-conditionally \
	--i-table ./classifier/gg2-2022.10-v4-v5.feature-table.biom.qza \
	--p-abundance 0.01 \
	--p-prevalence 0.1 \
	--o-filtered-table ./qza/filtered-abun_0.01-prev_0.1-feature-table.qza
```

**[optional: ]** Check summary of the filtered feature table.

```bash
qiime feature-table summarize \
	--i-table ./qza/filtered-abun_0.01-prev_0.1-feature-table.qza \
	--m-sample-metadata-file ./others/metadata.tsv \
	--o-visualization ./qzv/filtered-abun_0.01-prev_0.1-feature-table-summary.qzv
```

### Step-3: Generate the Heatmap

```bash
qiime feature-table heatmap \
	--i-table ./qza/filtered-abun_0.01-prev_0.1-feature-table.qza \
	--m-sample-metadata-file ./others/metadata.tsv \
	--m-sample-metadata-column categorical-time-relative-to-hct \
	--o-visualization ./qzv/heatmap.qzv
```

## Core Features

```bash
qiime feature-table core-features \
	--i-table ./qza/gg2-taxa_species-feature-table.qza \
	--p-min-fraction 0.5 \
	--o-visualization ./qzv/core-features.qzv
```

## Beta Rarefaction

```bash
qiime diversity beta-rarefaction \
    --i-table ./classifier/gg2-2022.10-v4-v5.feature-table.biom.qza \
    --p-metric jaccard \
    --p-clustering-method upgma \
    --m-metadata-file ./others/metadata.tsv \
    --p-sampling-depth 24392 \
    --i-phylogeny ./qza/rooted-tree.qza \
    --o-visualization ./qzv/beta-rarefaction.qzv
```

Export:

```bash
qiime tools export \
    --input-path ./qzv/beta-rarefaction.qzv \
    --output-path ./results/upgma-tree
```

## Differential Abundance

## ANCOM

#### Step -1: Filter low abundance features

We have to filter out lowly abundant features/OTUs/taxa from the feature-table. Because according to the following paper ([Analysis of microbial compositions: a review of normalization and differential abundance analysis](https://www.nature.com/articles/s41522-020-00160-w)), ANCOM and ANCOM-BC (bias correction) fail to control FDR at sample sizes <10.

```bash
qiime feature-table filter-features \
	--i-table ./qza/gg2-taxa_species-feature-table.qza \
	--p-min-samples 15 \
	--o-filtered-table ./qza/filtered-feature-table-ANCOM.qza
```

#### Step-2: Add pseudocount

ANCOM operates on `FeatureTable[Composition]` artifact, which is per-sample basis, ***\*but cannot tolerate frequencies of zero.\**** To build the composition artifact, a `FeatureTable[Frequency]` artifact must be provided to ***\*add-pseudocount\**** (an imputation method), which will produce the `FeatureTable[Composition] artifact`.

```bash
qiime composition add-pseudocount \
	--i-table ./qza/filtered-feature-table-ANCOM.qza \
	--o-composition-table ./qza/comp-filtered-feature-table-ANCOM.qza
```

#### Step-3: Calculate differential abundance

```bash
qiime composition ancom \
	--i-table ./qza/comp-filtered-feature-table-ANCOM.qza \
	--m-metadata-file ./others/metadata.tsv \
	--m-metadata-column categorical-time-relative-to-hct \
	--o-visualization ./qzv/ancom-categorical-time-relative-to-hct.qzv
```

### ANCOM-BC

```bash
qiime composition ancombc \
	--i-table ./qza/gg2-taxa_species-feature-table.qza \
	--m-metadata-file ./others/metadata.tsv \
	--p-formula autoFmtGroup \
	--p-prv-cut 0.5 \
	--o-differentials ./qza/autoFmtGroup-ancombc.qza
```

> For this method, we can't use the column names with **`+`** or **`-`** . Because this method use + and - as part of its formula. That's why, during preparing metadata we should avoid using **`-`** as word separator. Best approach is following **camelCase** naming convention.

```bash
qiime composition da-barplot \
	--i-data ./qza/autoFmtGroup-ancombc.qza \
	--p-level-delimiter ';' \
	--p-significance-threshold 1 \
	--o-visualization ./qzv/check-autoFmtGroup-da-barplot.qzv
```

