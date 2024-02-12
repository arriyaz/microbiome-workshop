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

## Importing the data

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

First check the format (Phred score version) of the data. Run the data with `fastqc` tool. 

> Ensure that conda has `fastqc` installed, prefereabley in a different environment. 
> 
> You can install `fastqc` by running `conda install bioconda::fastqc` in the target environment.

Run fastqc for some files.

```bash
fastqc ./data/FMT.0093C_46_L001_R2_001.fastq.gz
```

Import the data.

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
F_read_trunc_len=204
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
