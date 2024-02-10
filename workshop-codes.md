## # Workshop On Microbiome Analysis

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

## Install QIIM2

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

?‘‰?‘‰?‘‰?‘‰?‘‰?‘‰

## Check an artifact

```bash
mkdir -p temp

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
????--output-path ./temp/exported-sequences/
```

## Check available plugins

```bash
qiime --help
```

Check actions under diversity plugin:

```bash
qiime diversity --help
```

?:point_right: :point_right: :point_right: :point_right: :point_right: 

## Importing the data

### Check importable types

```bash
qiime tools list-types  --tsv
```

?“¢ Let's save the importable types list in a tsv file

### Check importable formats

```bash
qiime tools list-formats --importable --tsv
```

:loudspeaker: Let's save the importable format list in a tsv file

:loudspeaker: Let's save the exportable format list in a tsv file

# Redo: Reconstitution of the gut microbiota of antibiotic-treated patients by autologous fecal microbiota transplant

  doi: 10.1126/scitranslmed.aap9489 

```bash
# Create necessary folders
mkdir -p qzv qza others results data
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
' others/sampleids.txt others/compilation-metadata.tsv \
>> ./others/metadata.tsv
```



Validate the metadata

```bash
qiime metadata tabulate \
  --m-input-file ./others/metadata.tsv \
  --o-visualization ./qzv/metadata.qzv
```

Go to https://view.qiime2.org/  and upload the `qvz` file to view it.

Download the data:

```bash
for file in $(cat ./others/SRAids.txt)
do
parallel-fastq-dump --sra-id $file --threads 30 --outdir ./data/ --split-files --gzip 
done
```

:point_right: :point_right: :point_right: :point_right: 

## Prepare Manifest File

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



## Import Data

First check the format (Phred score version) of the data. Run the data with `fastqc` tool. 

> Ensure that conda has `fastqc` installed, prefereabley in different environment. 
> 
> You can install `fastqc` by running `conda install bioconda::fastqc` in the target environment.

Run 

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
  --i-data qza/paired-end-demux.qza \
  --p-n 10000 \
  --o-visualization qzv/demux.qzv
```

Let's view the sequence summary from `demuz.qzv` file.

:point_right: :point_right: :point_right: :point_right:


