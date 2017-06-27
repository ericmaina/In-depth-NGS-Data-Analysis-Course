---
title: "ChIPseeker for ChIP peak Annotation, Comparison, and Visualization"
author: "Meeta Mistry"
date: "June 12, 2017"
---

Contributors: Mary Piper and Meeta Mistry

Approximate time: 1.5 hours

## Learning Objectives



## ChIPseeker

Now that we have a set of high confidence peaks for our samples, the next step is to **annotate our peaks to identify relative location relationship information between query peaks and genes/genomic features** to obtain some biological context. 

[ChIPseeker](http://bioconductor.org/packages/release/bioc/vignettes/ChIPseeker/inst/doc/ChIPseeker.html) is an R package for annotating ChIP-seq data analysis. It supports annotating ChIP peaks and provides functions to visualize ChIP peaks coverage over chromosomes and profiles of peaks binding to TSS regions. Comparison of ChIP peak profiles and annotation are also supported, and can be useful to estimate how well biological replications are. Several visualization functions are implemented to visualize the peak annotation and statistical tools for enrichment analyses of functional annotations.


### Setting up 

1. Open up RStudio and open up the `chipseq-project` that we created previously.
2. Open up a new R script ('File' -> 'New File' -> 'Rscript'), and save it as `chipseeker.R`

> **NOTE:** This next section assumes you have the `ChIPseeker` package installed for R 3.3.3. You will also need one additional library for gene annotation. If you haven't done this please run the following lines of code before proceeding.
>
```
source("http://bioconductor.org/biocLite.R")
biocLite("ChIPseeker")
biocLite("TxDb.Hsapiens.UCSC.hg19.knownGene")
```

### Getting data 

As mentioned previously, these donwstream steps should be performed on your high confidence peak calls. While we have a set for our subsetted data, this set is rather small and will not result in anything meaningful in our functional analyses. **We have generated a set of high confidence peak calls using the full dataset.** These were obtained post-IDR analysis, (i.e. concordant peaks between replicates) and are provided in BED format which is optimal input for the ChIPseeker package. 

> **NOTE:** the number of peaks in these bed files are are significantly higher than what we observed with the subsetted data replicate analysis.

We will need to copy over the appropriate files from Orchestra to our laptop. You can do this using `FileZilla` or the `scp` command.

Move over the **BED files from Orchestra(`/groups/hbctraining/chip-seq/full-dataset/idr/*.bed`) to your laptop**. You will want to copy these files into your chipseq-project **into a new folder called `data/idr-bed`.**


Let's start by loading the libraries:

```r
# Load libraries
library(ChIPseeker)
library(TxDb.Hsapiens.UCSC.hg19.knownGene)
library(clusterProfiler)
library(biomaRt)
```

Now let's load all of the data. As input we need to provide the names of our BED files in a list format.

```r
# Load data
samplefiles <- list.files("data/idr-bed", pattern= ".bed", full.names=T)
samplefiles <- as.list(samplefiles)
names(samplefiles) <- c("Nanog", "Pou5f1")

```

We need to assign annotation databases generated from UCSC to a variable:

	txdb <- TxDb.Hsapiens.UCSC.hg19.knownGene

### Visualization

First, let's take a look at peak locations across the genome. The `covplot` function calculates **coverage of peak regions** across the genome and generates a figure to visualize this across chromosomes. We do this for the Nanog peaks and find a considerable number of peaks on all chromosomes.

```
# Assign peak data to variables
nanog <- readPeakFile(samplefiles[[1]])
pou5f1 <- readPeakFile(samplefiles[[2]])

# Plot covplot
covplot(nanog, weightCol="V5")

```

<img src="../img/covplot.png">


Using a window of +/- 1000bp around the TSS of genes we can plot the **density of read count frequency to see where binding is relative to the TSS** or each sample. This is similar to the plot in the ChIPQC report but with the flexibility to customize the plot a bit. We will plot both Nanog and Pou5f1 together to compare the two.

```
# Prepare the promotor regions
promoter <- getPromoters(TxDb=txdb, upstream=1000, downstream=1000)

# Calculate the tag matrix
tagMatrixList <- lapply(as.list(samplefiles), getTagMatrix, windows=promoter)

## Profile plots
plotAvgProf(tagMatrixList, xlim=c(-1000, 1000), conf=0.95,resample=500, facet="row")
```
<img src="../img/density_profileplots.png">

With these plots the confidence interval is estimated by bootstrap method (500 iterations) and is shown in the grey shading that follows each curve. The Nanog peaks exhibit a nice narrow peak at the TSS with small confidence intervals. Whereas the Pou5f1 peaks display a bit wider peak suggesting binding around the TSS with larger confidence intervals.

The **heatmap is another method of visualizing the read count frequency** relative to the TSS.

	# Plot heatmap
	tagHeatmap(tagMatrixList, xlim=c(-1000, 1000), color=NULL)

<img src="../img/Rplot.png" width=500>

### Annotation

ChIPseeker implements the `annotatePeak` function for annotating peaks with nearest gene and genomic region where the peak is located. Many annotation tools calculate the distance of a peak to the nearest TSS and annotates the peak to that gene. This can be misleading as binding sites might be located between two start sites of different genes or hit different genes, which have the same TSS location in the genome. 

<img src="../img/annotate-genes.png" width=800>

The **`annotatePeak` function provides parameters to annotate genes with a max distance cutoff and all genes within this distance will be reported for each peak**. For annotating genomic regions, annotatePeak function reports detail information when genomic region is Exon or Intron. For instance, ‘Exon (uc002sbe.3/9736, exon 69 of 80)’, means that the peak overlaps with the 69th exon of the 80 exons that transcript uc002sbe.3 possess and the corresponding Entrez gene ID is 9736. 

Let's start by retrieving annotations for our Nanog and Pou5f1 peaks calls:

```
peakAnnoList <- lapply(samplefiles, annotatePeak, TxDb=txdb, 
                       tssRegion=c(-1000, 1000), verbose=FALSE)
```

If you take a look at what is stored in `peakAnnoList`, you will see a summary of genomic features for each sample:

```
> peakAnnoList
$Nanog
Annotated peaks generated by ChIPseeker
11023/11035  peaks were annotated
Genomic Annotation Summary:
             Feature  Frequency
9           Promoter 17.1731833
4             5' UTR  0.2358705
3             3' UTR  0.9706976
1           1st Exon  0.5443164
7         Other Exon  1.7781003
2         1st Intron  7.2121927
8       Other Intron 28.2318788
6 Downstream (<=3kb)  0.9434818
5  Distal Intergenic 42.9102785

$Pou5f1
Annotated peaks generated by ChIPseeker
3242/3251  peaks were annotated
Genomic Annotation Summary:
             Feature  Frequency
9           Promoter  3.7939543
4             5' UTR  0.1542258
3             3' UTR  1.0795805
1           1st Exon  1.3263418
7         Other Exon  1.6347933
2         1st Intron  7.4336829
8       Other Intron 30.9685379
6 Downstream (<=3kb)  0.9561999
5  Distal Intergenic 52.6526835
```

To visualize this annotation data ChIPseeker provides several functions. We will demonstrate a few using the Nanog sample only. We will also show how some of the functions can also support comparing across samples.

#### Pie chart of genomic region annotation

```
plotAnnoPie(peakAnnoList[["Nanog"]])
```

<img src="../img/">

#### Vennpie of genomic region annotation

```
vennpie(peakAnnoList[["Nanog"]])
```

<img src="../img/">

#### Barchart (multiple samples for comparison)

**Here, we see that Nanog has a much larger percentage of peaks in promotor regions.**

```
plotAnnoBar(peakAnnoList)

```
<img src="../img/feature-distribution.png">

#### Distribution of TF-binding loci relative to TSS (multiple samples)

```
plotDistToTSS(peakAnnoList, title="Distribution of transcription factor-binding loci \n relative to TSS")
```
<img src="../img/tss-dist.png">



### Functional enrichment analysis







***
*This lesson has been developed by members of the teaching team at the [Harvard Chan Bioinformatics Core (HBC)](http://bioinformatics.sph.harvard.edu/). These are open access materials distributed under the terms of the [Creative Commons Attribution license](https://creativecommons.org/licenses/by/4.0/) (CC BY 4.0), which permits unrestricted use, distribution, and reproduction in any medium, provided the original author and source are credited.*