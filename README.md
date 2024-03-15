# Alignment Extraction

This repository stores scripts for extracting alignments of coding regions and non-coding elements.  Installation of [optparse](https://cran.r-project.org/package=optparse), [RPHAST](https://github.com/CshlSiepelLab/RPHAST) and the `maf_parse` binary file from [PHAST](http://compgen.cshl.edu/phast/) are needed. Note that RPHAST will not build on R 4.3.0 or newer due to changes in the Fortran/BLAS/LAPACK under the hood. See the [R release notes](https://cran.r-project.org/doc/manuals/r-release/NEWS.html) for details.

## Installation

Installation instructions for Ubuntu 22.04 (Jammy) below.

### PHAST

As root:

```sh
# Run everything as root
sudo su
# Prereqs
apt update
apt install -y build-essential f2c gdebi-core libpcre3 libpcre3-dev
# Build CLAPACK
cd /
wget http://www.netlib.org/clapack/clapack.tgz
tar xzf clapack.tgz
cd CLAPACK-3.2.1
cp make.inc.example make.inc && make f2clib && make blaslib && make lib
# Build phast
cd /
git clone https://github.com/CshlSiepelLab/phast
cd phast/src
make CLAPACKPATH=/CLAPACK-3.2.1
make install
```

### RPHAST

In the shell, install R 4.2.3 (4.3.0 or newer won't work):

```sh
export R_VERSION="4.2.3"
sudo curl -O https://cdn.rstudio.com/r/ubuntu-2204/pkgs/r-${R_VERSION}_1_amd64.deb
sudo gdebi -n r-${R_VERSION}_1_amd64.deb
sudo ln -s /opt/R/${R_VERSION}/bin/R /usr/local/bin/R
sudo ln -s /opt/R/${R_VERSION}/bin/Rscript /usr/local/bin/Rscript
echo 'options(repos = c(CRAN = "https://packagemanager.posit.co/cran/__linux__/jammy/latest"))' | sudo tee -a /opt/R/4.2.3/lib/R/etc/Rprofile.site
```

In R:

```r
options(repos = c(CRAN = "https://packagemanager.posit.co/cran/__linux__/jammy/latest"))
install.packages(c("optparse", "remotes"))
remotes::install_github("CshlSiepelLab/RPHAST")
```

## Extraction of *coding region* alignments

Alignment extraction is done in a **chromosome-specific** manner. The starting multiple sequence alignment must be chromosome-speficic and in a MAF format. If alignment file is not chromosome-specific, split the MAF file to chromosome-specific MAFs (example: use [MafFilter](https://jydu.github.io/maffilter/)) *BEFORE* proceeding to the steps below.

The input datasets for this procedure are:

- GTF file for *each* chromosome of interest
- MAF of the chromosome of interest, fragmented into smaller chunks

Example input and output files can be found in folder `data` and `output`.

### Step 1: Split chromosome-specific MAF into smaller fragments

To make the operation computationally feasible, start by splitting the MAF of a chromosome into smaller fragments of about 100kb-300kb, depending on the size of your MAF. Use the `maf_parse` function from PHAST to do this (see documentation [here](http://compgen.cshl.edu/phast/help-pages/maf_parse.txt)).

Below is an example command for splitting the MAF of chrY (`mouse24way_chrY.maf`) into fragments of 300kb. The resulting outputs can be seen in the folder `data/alignment/chrY/`.

<details><summary>Creating the mouse-24way_chrY.maf.gz file (expand).</summary>

----

Starting with the data provided by the upstream in [data/alignment/chrY](data/alignment/chrY). First get the maf-sort.sh script from the [last repo mirror](https://github.com/UCSantaCruzComputationalGenomicsLab/last).

```sh
wget https://raw.githubusercontent.com/UCSantaCruzComputationalGenomicsLab/last/master/scripts/maf-sort.sh
chmod +x maf-sort.sh
```

Cat them all and sort them.

```sh
cat data/alignment/chrY/chrYsplit*.maf | grep -v "^#" | ./maf-sort.sh > mouse24way_chrY.maf
gzip mouse24way_chrY.maf
```

Now get just the chrM alignment from a different alignment, so as to demonstrate splitting the MAF file.

```sh
wget http://hgdownload.cse.ucsc.edu/goldenPath/mm10/multiz60way/maf/chrM.maf.gz
```

Cat them together into a single MAF file.

```sh
cat <(gzip -dc mouse24way_chrY.maf) <(gzip -dc chrM.maf.gz) | ./maf-sort.sh > mouse.24way_chrY_MT.maf
gzip -f mouse.24way_chrY_MT.maf
rm -f chrM.maf.gz
```

If we want to split the MAF file by chromosome, we could use mafSplit from UCSC:

```sh
wget http://hgdownload.cse.ucsc.edu/admin/exe/linux.x86_64/mafSplit
chmod +x mafSplit
mkdir mafsplit
mafSplit -byTarget -useFullSequenceName placeholder.bed mafsplit/ <(gzip -dc mouse.24way_chrY_MT.maf.gz)
# file is mafsplit/chrY.maf
```

----

</details>


```sh
maf_parse <(gzip -dc mouse24way_chrY.maf.gz) --split 300000 --out-root data/alignment/chrY/chrYsplit
```

Output files in [data/alignment/chrY](data/alignment/chrY/) are `chrYsplit*.maf`.

### Step 2: Obtain the coordinates of coding regions (CDS) from GTF file

This step produces a GTF file that only contains coding region coordinates of genes in the chromosome of interest. This step is performed by the function [`getCDSCoordinates.R`](getCDSCoordinates.R), which takes the following arguments:

- `-i, --inputpath`: path to input GTF file
- `-o, --outputpath`: path to output GTF file (CDS only)

The following is an example of how to run the function from the command line:

```sh
Rscript getCDSCoordinates.R -i data/genepred/mm10.ncbiRefSeq.chrY.gtf -o output/CDS-coordinates/mm10.ncbiRefSeq.coding.chrY.gtf
```


### Step 3: Get gene boundaries and CDS coordinates per gene

This step produces data structures necessary for the subsequent extraction of coding region alignments. This step is performed by the function [`getGeneBoundaries.R`](getGeneBoundaries.R), which takes the following arguments:

- `-c, --chromosome`: chromosome of interest
- `-i, --genepred_cds_path`: GTF file path containing the CDS coordinates in the chromosome of interest (output from Step 2)
- `-o, --output_folder`: output folder path

The following is an example of how to run the function from the command line:

```sh
Rscript getGeneBoundaries.R \
    -c chrY \
    -i output/CDS-coordinates/mm10.ncbiRefSeq.coding.chrY.gtf \
    -o output/CDS-information
```

Output files in [output/CDS-information/](output/CDS-information/) (output file names are partially hard-coded, see [`getGeneBoundaries.R`](getGeneBoundaries.R)).

1. `gene_boundaries_information_chrY.RDS`
1. `gene_cds_information_chrY.RDS`

### Step 4: Extract coding region alignments

This step extracts the alignments of coding regions of interest in the FASTA format. This step is performed by the function [`getCodingAlignments.R`](getCodingAlignments.R), which takes the following arguments:

- `-c, --chromosome`: chromosome of interest
- `-r, --refseq`: reference sequence of alignment
- `-i, --cds_info_folder`: CDS information folder (output folder path from Step 3)
- `-o, --output_folder`: output folder for coding region alignments
- `-a, --mafFolderPath`: path to folder containing chromosome MAF fragments
- `-p, --prefix`: prefix of MAF fragments

The following is an example of how to run the function from the command line. Resulting alignments are reverse-complemented when necessary. Example outputs can be seen in the folder [`output/coding-region-alignment/chrY/`](output/coding-region-alignment/chrY).

```sh
Rscript getCodingAlignments.R \
    -c chrY \
    -r mm10 \
    -i output/CDS-information \
    -o output/coding-region-alignment/chrY \
    -a data/alignment/chrY \
    -p chrYsplit
```

Output files in [`output/coding-region-alignment/chrY/`](output/coding-region-alignment/chrY) include FASTA files (`*.fa`) with multiple sequence alignments, each being named according to the gene name (e.g., [`SLY.fa`](output/coding-region-alignment/chrY/SLY.fa), [`SRY.fa`](output/coding-region-alignment/chrY/SRY.fa), etc.).

## Extraction of *non-coding region* alignments

Similar to the coding region extraction, non-coding region alignment extraction is also done in a **chromosome-specific** manner.

The input datasets for this procedure are:

- BED file containing coordinates of non-coding regions for *each* chromosome of interest
- MAF of the chromosome of interest, fragmented into smaller chunks

### Step 1: Split chromosome-specific MAF into smaller fragments

The same as Step 1 for coding regions.

### Step 2: Extract non-coding region alignments

This step extracts the alignments of non-coding regions of interest in the FASTA format. This step is performed by the function [`getNonCodingAlignments.R`](getNonCodingAlignments.R), which takes the following arguments:

- `-c, --chromosome`: chromosome of interest
- `-r, --refseq`: reference sequence of alignment
- `-i, --elementBedPath`: path to BED file containing coordinates of regions of interest, e.g. [`data/CNE-coordinates/mouseCNEs.chrY.bed`](data/CNE-coordinates/mouseCNEs.chrY.bed)
- `-o, --output_folder`: output folder for non-coding region alignments
- `-a, --mafFolderPath`: path to folder containing chromosome MAF fragments
- `-p, --prefix`: prefix of MAF fragments

The following is an example of how to run the function from the command line. Example outputs can be seen in the folder `output/CNE-alignment/`.

```sh
Rscript getNonCodingAlignments.R \
    -c chrY \
    -r mm10 \
    -i data/CNE-coordinates/mouseCNEs.chrY.bed \
    -o output/CNE-alignment \
    -a data/alignment/chrY \
    -p chrYsplit
```

Outputs in [output/CNE-alignment/](output/CNE-alignment/) include FASTA files with multiple sequence alignments (`*.fa`) named according to the region name in the input BED file.