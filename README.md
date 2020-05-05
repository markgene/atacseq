## Introduction

The repo markgene/atacseq is an ATAC-seq pipeline configured to be run on internal high performance cluster (HPC) with job scheduler platform Load Sharing Facility (LSF). It is originally forked from [nf-core/atacseq](https://github.com/nf-core/atacseq), a bioinformatics analysis pipeline used for ATAC-seq data. I am working on additional features to the pipeline.

The pipeline is built using [Nextflow](https://www.nextflow.io), a workflow tool to run tasks across multiple compute infrastructures in a very portable manner. It comes with docker containers making installation trivial and results highly reproducible.

## Pipeline summary

1. Raw read QC ([`FastQC`](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/))
2. Adapter trimming ([`Trim Galore!`](https://www.bioinformatics.babraham.ac.uk/projects/trim_galore/))
3. Alignment ([`BWA`](https://sourceforge.net/projects/bio-bwa/files/))
4. Mark duplicates ([`picard`](https://broadinstitute.github.io/picard/))
5. Merge alignments from multiple libraries of the same sample ([`picard`](https://broadinstitute.github.io/picard/))
    1. Re-mark duplicates ([`picard`](https://broadinstitute.github.io/picard/))
    2. Filtering to remove:
        * reads mapping to mitochondrial DNA ([`SAMtools`](https://sourceforge.net/projects/samtools/files/samtools/))
        * reads mapping to blacklisted regions ([`SAMtools`](https://sourceforge.net/projects/samtools/files/samtools/), [`BEDTools`](https://github.com/arq5x/bedtools2/))
        * reads that are marked as duplicates ([`SAMtools`](https://sourceforge.net/projects/samtools/files/samtools/))
        * reads that arent marked as primary alignments ([`SAMtools`](https://sourceforge.net/projects/samtools/files/samtools/))
        * reads that are unmapped ([`SAMtools`](https://sourceforge.net/projects/samtools/files/samtools/))
        * reads that map to multiple locations ([`SAMtools`](https://sourceforge.net/projects/samtools/files/samtools/))
        * reads containing > 4 mismatches ([`BAMTools`](https://github.com/pezmaster31/bamtools))
        * reads that are soft-clipped ([`BAMTools`](https://github.com/pezmaster31/bamtools))
        * reads that have an insert size > 2kb ([`BAMTools`](https://github.com/pezmaster31/bamtools); *paired-end only*)
        * reads that map to different chromosomes ([`Pysam`](http://pysam.readthedocs.io/en/latest/installation.html); *paired-end only*)
        * reads that arent in FR orientation ([`Pysam`](http://pysam.readthedocs.io/en/latest/installation.html); *paired-end only*)
        * reads where only one read of the pair fails the above criteria ([`Pysam`](http://pysam.readthedocs.io/en/latest/installation.html); *paired-end only*)
    3. Alignment-level QC and estimation of library complexity ([`picard`](https://broadinstitute.github.io/picard/), [`Preseq`](http://smithlabresearch.org/software/preseq/))
    4. Create normalised bigWig files scaled to 1 million mapped reads ([`BEDTools`](https://github.com/arq5x/bedtools2/), [`wigToBigWig`](http://hgdownload.soe.ucsc.edu/admin/exe/))
    5. Generate gene-body meta-profile from bigWig files ([`deepTools`](https://deeptools.readthedocs.io/en/develop/content/tools/plotProfile.html))
    6. Calculate genome-wide enrichment ([`deepTools`](https://deeptools.readthedocs.io/en/develop/content/tools/plotFingerprint.html))
    7. Call broad/narrow peaks ([`MACS2`](https://github.com/taoliu/MACS))
    8. Annotate peaks relative to gene features ([`HOMER`](http://homer.ucsd.edu/homer/download.html))
    9. Create consensus peakset across all samples and create tabular file to aid in the filtering of the data ([`BEDTools`](https://github.com/arq5x/bedtools2/))
    10. Count reads in consensus peaks ([`featureCounts`](http://bioinf.wehi.edu.au/featureCounts/))
    11. Differential accessibility analysis, PCA and clustering ([`R`](https://www.r-project.org/), [`DESeq2`](https://bioconductor.org/packages/release/bioc/html/DESeq2.html))
    12. Generate ATAC-seq specific QC html report ([`ataqv`](https://github.com/ParkerLab/ataqv))
6. Merge filtered alignments across replicates ([`picard`](https://broadinstitute.github.io/picard/))
    1. Re-mark duplicates ([`picard`](https://broadinstitute.github.io/picard/))
    2. Remove duplicate reads ([`SAMtools`](https://sourceforge.net/projects/samtools/files/samtools/))
    3. Create normalised bigWig files scaled to 1 million mapped reads ([`BEDTools`](https://github.com/arq5x/bedtools2/), [`wigToBigWig`](http://hgdownload.soe.ucsc.edu/admin/exe/))
    4. Call broad/narrow peaks ([`MACS2`](https://github.com/taoliu/MACS))
    5. Annotate peaks relative to gene features ([`HOMER`](http://homer.ucsd.edu/homer/download.html))
    6. Create consensus peakset across all samples and create tabular file to aid in the filtering of the data ([`BEDTools`](https://github.com/arq5x/bedtools2/))
    7. Count reads in consensus peaks relative to merged library-level alignments ([`featureCounts`](http://bioinf.wehi.edu.au/featureCounts/))
    8. Differential accessibility analysis, PCA and clustering ([`R`](https://www.r-project.org/), [`DESeq2`](https://bioconductor.org/packages/release/bioc/html/DESeq2.html))
7. Create IGV session file containing bigWig tracks, peaks and differential sites for data visualisation ([`IGV`](https://software.broadinstitute.org/software/igv/)).
8. Present QC for raw read, alignment, peak-calling and differential accessibility results ([`ataqv`](https://github.com/ParkerLab/ataqv), [`MultiQC`](http://multiqc.info/), [`R`](https://www.r-project.org/))

## Quick Start

i. Install [`nextflow`](https://nf-co.re/usage/installation)

```sh
# Make sure that Java v8+ is installed:
java -version
# Install Nextflow
curl -fsSL get.nextflow.io | bash
# Add Nextflow binary to your user's PATH:
mv nextflow ~/bin/
# Test with hello-world example
nextflow run hello
```

ii. Install conda environment.

```sh
git clone https://github.com/markgene/atacseq.git
cd atacseq
# The command will create a Conda env named atacseq. Be sure you do not have 
# the evironment of the same name.
conda env create -f environment.yml

# Wait for the installlation and then activate it.
conda activate atacseq
```

iii. Download the pipeline and test it on a minimal dataset with a single command

```bash
## Notice the test is run in local executor.
bsub -P MC -J testLocal -q priority -n 2 -R "rusage[mem=8000]" "nextflow run main.nf -profile test -with-report report.html -with-trace -with-timeline timeline.html -with-dag flowchart.png"
# Test with test_sjlsf
bsub -P MC -J testLSF -q priority -n 1 -R "rusage[mem=8000]" "nextflow run main.nf -profile test_sjlsf -with-report report.html -with-trace -with-timeline timeline.html -with-dag flowchart.png"
```

> Please check [nf-core/configs](https://github.com/nf-core/configs#documentation) to see if a custom config file to run nf-core pipelines already exists for your Institute. If so, you can simply use `-profile institute` in your command. This will enable either `docker` or `singularity` and set the appropriate execution settings for your local compute environment.

iv. Start running your own analysis!

```bash
# DO NOT RUN. This will take a while. 
# It runs four mouse samples with the config specified in `conf/sjlsf.config`.
bsub -P MC -J exampleRun -q priority -n 1 "nextflow run main.nf -profile conf/sjlsf.config -with-report report.html -with-trace -with-timeline timeline.html -with-dag flowchart.png --outdir sjlsf_results -resume"
```

See [usage docs](docs/usage.md) for all of the available options when running the pipeline.

## Documentation

The markgene/atacseq document is found in the `docs/` directory:

1. [Running the pipeline](docs/usage.md)
1. [Output and how to interpret the results](docs/output.md)

The original nf-core/atacseq pipeline comes with documentation:

1. [Installation](https://nf-co.re/usage/installation)
1. Pipeline configuration
    * [Local installation](https://nf-co.re/usage/local_installation)
    * [Adding your own system config](https://nf-co.re/usage/adding_own_config)
    * [Reference genomes](https://nf-co.re/usage/reference_genomes)
1. [Troubleshooting](https://nf-co.re/usage/troubleshooting)

## To-dos

1. Add Irreproducible Discovery Rate.
1. Downstream analysis. I am going to add the following analyses to the pipeline, i) motif enrichment analysis, ii) transcription factor footprinting, and iii) nucleosome positioning. Perhaps other analyses in the future.
1. Generate report. The pipeline organizes the output in a hierarchical structure. It reflects how the workflow goes and sometime the result is stored in a deep manner. I will collect the results I am most concerned to one place, perhaps by creating symbolic links of the different sub-folders or files.

