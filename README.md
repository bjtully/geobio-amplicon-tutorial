---
title: International Geobiology Course 2021 - Example Amplicon Workflow Analysis
author: "Benjamin Tully - modified from Happy Belly example by Dr. Michael Lee"
---

This tutorial will have 3 parts. These parts include:

* Very short Pre-part 1
* Part 1 - Running DADA2 on amplicon data in R
* Part 2 - Performing data analysis in R

The original data are from this [paper](https://www.frontiersin.org/articles/10.3389/fmicb.2015.01470/full) and are also the dataset used in the Happy Belly Amplicons Analysis [tutorial](https://astrobiomike.github.io/amplicon/)

Launch our Binder from here: [![Binder](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/AstrobioMike/binder-dada2-ex-workflow/v4?urlpath=rstudio)

## Pre-Part 1 -- Download and set-up the data

This will be run using the `Terminal` tab that is available after launching the Binder RStudio environment.

```
cd ~
curl -L -o dada2_amplicon_ex_workflow.tar.gz https://ndownloader.figshare.com/files/23066516
tar -xzvf dada2_amplicon_ex_workflow.tar.gz
rm dada2_amplicon_ex_workflow.tar.gz
cd dada2_amplicon_ex_workflow/
```
This will download 20 samples with forward (R1) and reverse (R2) reads. For a total of 40 fastq files (.fq). In this example we are using simplified names, but it always useful to name these files such that the first component links sample sources, maybe includes additional information (replace `_sub_`), and then details which read of the pair is present. We are going to use this formating to create a file that will help us in just a minute.

```
ls *_R1.fq | cut -f1 -d "_" > samples
```
### Remove primers
Here is an example of how to remove primers with one sample using the tool `cutadapt`:

```
cutadapt --version
cutadapt -a ^GTGCCAGCMGCCGCGGTAA...ATTAGAWACCCBDGTAGTCC \
    -A ^GGACTACHVGGGTWTCTAAT...TTACCGCGGCKGCTGGCAC \
    -m 215 -M 285 --discard-untrimmed \
    -o B1_sub_R1_trimmed.fq -p B1_sub_R2_trimmed.fq \
    B1_sub_R1.fq B1_sub_R2.fq
```
We can check to see how this looks after trimming:

```
head -n 2 B1_sub_R1.fq
head -n 2 B1_sub_R1_trimmed.fq
head -n 2 B1_sub_R2.fq
head -n 2 B1_sub_R2_trimmed.fq
```
That processes one sample, but we need to perform this on all the samples:

```
for sample in $(cat samples)
do

    echo "On sample: $sample"
    
    cutadapt -a ^GTGCCAGCMGCCGCGGTAA...ATTAGAWACCCBDGTAGTCC \
    -A ^GGACTACHVGGGTWTCTAAT...TTACCGCGGCKGCTGGCAC \
    -m 215 -M 285 --discard-untrimmed \
    -o ${sample}_sub_R1_trimmed.fq.gz -p ${sample}_sub_R2_trimmed.fq.gz \
    ${sample}_sub_R1.fq ${sample}_sub_R2.fq \
    >> cutadapt_primer_trimming_stats.txt 2>&1

done
```
We can use Unix to give us a quick summary of the results:

```
paste samples <(grep "passing" cutadapt_primer_trimming_stats.txt | cut -f3 -d "(" | tr -d ")") <(grep "filtered" cutadapt_primer_trimming_stats.txt | cut -f3 -d "(" | tr -d ")")
```

## Part 1 - Implementing DADA2 in R

In the first block of code, we will call the package DADA2 and create a number of variables that contain things like the sample IDs (from `samples`), the existing file names, and new files for the outputs we will generate.

```{r}
library(dada2)
packageVersion("dada2") # 1.11.5 when this was initially put together, though now 1.15.2 in the binder

setwd("~/dada2_amplicon_ex_workflow")

list.files() # make sure what we think is here is actually here

## first we're setting a few variables we're going to use ##
  # one with all sample names, by scanning our "samples" file we made earlier
samples <- scan("samples", what="character")

  # one holding the file names of all the forward reads
forward_reads <- paste0(samples, "_sub_R1_trimmed.fq.gz")
  # and one with the reverse
reverse_reads <- paste0(samples, "_sub_R2_trimmed.fq.gz")

  # and variables holding file names for the forward and reverse
  # filtered reads we're going to generate below
filtered_forward_reads <- paste0(samples, "_sub_R1_filtered.fq.gz")
filtered_reverse_reads <- paste0(samples, "_sub_R2_filtered.fq.gz")
```

### Trim amplicons for quality

DADA2 has a built in plotting function `plotQualityProfile()` which we can use to see the quality of our current reads.

```{r}
plotQualityProfile(forward_reads)
plotQualityProfile(reverse_reads)
  # and just plotting the last 4 samples of the reverse reads
plotQualityProfile(reverse_reads[17:20])
```

The quality scores start to decrease towards the end of the read, especially for the reverse read. We need to balance the amount that we trim from these reads with the amount of data we want to keep for downstream analysis. Having suitably long reads will allow us to overlap those reads in a future step. Non-overlapping reads will eventually be discarded.

Use the DADA2 command `filterAndTrim()`

```{r}
filtered_out <- filterAndTrim(forward_reads, filtered_forward_reads,
                reverse_reads, filtered_reverse_reads, maxEE=c(2,2),
                rm.phix=TRUE, minLen=175, truncLen=c(250,200))
```
Some information about the parameters we set:

 * `maxEE` is the quality filtering threshold being applied based on the expected errors and in this case we are saying we want to throw the read away if it is likely to have more than 2 erroneous base calls (we are specifying for both the forward and reverse reads separately) 
 * `rm.phix` removes any reads that match the PhiX bacteriophage genome, which is typically added to Illumina sequencing runs for quality monitoring
 * `minLen` is setting the minimum length reads we want to keep after trimming
 * We have our `truncLen` parameter setting the minimum size to trim the forward and reverse reads to in order to keep the quality scores roughly above 30 overall (based on our visual assessment.

`filtered_forward_reads` and `filtered_reverse_reads` contain the names of the files we are generating. We can see these files in our current directory with `list.files()`

We can also create an object call `filtered_out`. A matrix holding how many reads went in and how many reads made it out:

```{r}
class(filtered_out) # matrix
dim(filtered_out) # 20 2

filtered_out
```
And we can visualize the impact of this filtering step

```{r}
plotQualityProfile(filtered_forward_reads)
plotQualityProfile(filtered_reverse_reads)
plotQualityProfile(filtered_reverse_reads[17:20])
```

### Create the DADA2 error model

DADA2 creates in silico models of the error rate for each read set. This helps to clean away artefacts that have been added to data during sequencing. It helps when combining samples from different sequencing runs because it determines this error models for each sample separately. This step is ALSO VERY LONG (~30 minutes) - especially in Binder.

```{r}
#err_forward_reads <- learnErrors(filtered_forward_reads)
# err_forward_reads <- learnErrors(filtered_forward_reads, multithread=TRUE)
#err_reverse_reads <- learnErrors(filtered_reverse_reads)
# err_reverse_reads <- learnErrors(filtered_reverse_reads, multithread=TRUE)
load("amplicon_dada2_ex.RData")
```
We can visualize the results:

```{r}
plotErrors(err_forward_reads, nominalQ=TRUE)
plotErrors(err_reverse_reads, nominalQ=TRUE)
```
A brief explanation of what these figures mean:
> The red line is what is expected based on the quality score, the black line represents the estimate, and the black dots represent the observed. This is one of those cases where this isn???t a binary thing like ???yes, things are good??? or ???no, they???re not???. Over time and seeing outputs like this for multiple datasets you get a better feeling of what to expect and what should be more cause for alarm. But generally speaking, you want the observed (black dots) to track well with the estimated (black line). The DADA2 authors notes here that you can try to improve this by increasing the number of bases the function is using (default 100 million).

### Dereplication

Instead of keeping 100 identical sequences and doing all downstream processing to all 100, you can keep/process one of them, and just attach the number 100 to it.

```{r}
derep_forward <- derepFastq(filtered_forward_reads, verbose=TRUE)
names(derep_forward) <- samples # the sample names in these objects are initially the file names of the samples, this sets them to the sample names for the rest of the workflow
derep_reverse <- derepFastq(filtered_reverse_reads, verbose=TRUE)
names(derep_reverse) <- samples
```

### Inferring ASVs

ASVs are the big advancement for DADA2 (compared to OTUs) - with the goal of identifying "true biological sequences." It does this by incorporating the consensus quality profiles and abundances of each unique sequence, and then figuring out if each sequence is more likely to be of biological origin or more likely to be spurious.

[DADA2 manuscript](https://www.nature.com/articles/nmeth.3869#methods)

[DADA2 website](https://benjjneb.github.io/dada2/index.html)

> This step can be run on individual samples, which is the least computationally intensive manner, or on all samples together, which increases the function???s ability to resolve low-abundance ASVs. Imagine Sample A has 10,000 copies of sequence Z, and Sample B has 1 copy of sequence Z. Sequence Z would likely be filtered out of Sample B even though it was a ???true??? singleton among perhaps thousands of spurious singletons we needed to remove. Because running all samples together on large datasets can become impractical computationally, the developers also added a way to try to combine the best of both worlds they refer to as pseudo-pooling. This basically provides a way to tell Sample B that sequence Z is legit. But it???s noted that this is not always the best way to go, and it may depend on your experimental design if this appropriate for your data.

```{r}
dada_forward <- dada(derep_forward, err=err_forward_reads, pool="pseudo")
dada_reverse <- dada(derep_reverse, err=err_reverse_reads, pool="pseudo")
```

### Merge forward and reverse reads

We trimmed the forward reads to 250bp and the reverse to 200bp, and our primers were 515f???806r. After cutting off the primers we???re expecting a typical amplicon size of around 260 bases, so our typical overlap should be up around 190bp.

To allow for biology we set the minimum overlap for this dataset for 170bp. We also set the `trimOverhang` option to `TRUE` in case any of our reads go passed their opposite primers.

```{r}
merged_amplicons <- mergePairs(dada_forward, derep_forward, dada_reverse,
                    derep_reverse, trimOverhang=TRUE, minOverlap=170)

  # this object holds a lot of information that may be the first place you'd want to look if you want to start poking under the hood
class(merged_amplicons) # list
length(merged_amplicons) # 20 elements in this list, one for each of our samples
names(merged_amplicons) # the names() function gives us the name of each element of the list 

class(merged_amplicons$B1) # each element of the list is a dataframe that can be accessed and manipulated like any ordinary dataframe

names(merged_amplicons$B1) # the names() function on a dataframe gives you the column names
# "sequence"  "abundance" "forward"   "reverse"   "nmatch"    "nmismatch" "nindel"    "prefer"    "accept"
```

At this point we've generated ASVs. We now want to start creating the outputs we want to use in our downstream analysis.

### Generate a count table

```{r}
seqtab <- makeSequenceTable(merged_amplicons)
class(seqtab) # matrix
dim(seqtab) # 20 2521
```

### Detect chimeras

Chimeras are detected by aligning each sequence with those that were recovered in greater abundance and then seeing if there are any lower-abundance sequences that can be made exactly by mixing left and right portions of two of the more-abundant ones.

```{r}
seqtab.nochim <- removeBimeraDenovo(seqtab, verbose=T)

  # though we only lost 17 sequences, we don't know if they held a lot in terms of abundance, this is one quick way to look at that
sum(seqtab.nochim)/sum(seqtab)
```

### Create a summary table

This is a quick way to pull out how many reads were dropped at various points of the pipeline. Where we will start if a sample lost a lot of reads in a single step.

```{r}
 # set a little function
getN <- function(x) sum(getUniques(x))

  # making a little table
summary_tab <- data.frame(row.names=samples, dada2_input=filtered_out[,1],
               filtered=filtered_out[,2], dada_f=sapply(dada_forward, getN),
               dada_r=sapply(dada_reverse, getN), merged=sapply(merged_amplicons, getN),
               nonchim=rowSums(seqtab.nochim),
               final_perc_reads_retained=round(rowSums(seqtab.nochim)/filtered_out[,1]*100, 1))

summary_tab
```

We can save this file for later use

```{r}
write.table(summary_tab, "read-count-tracking.tsv", quote=FALSE, sep="\t", col.names=NA)
```

### Assign taxonomy

We are using the DECIPHER package and will be using a DECIPHER-format version of the SILVA v138 database. We will download a special version of this database for this tutorial.

```{r}
## downloading DECIPHER-formatted SILVA v138 reference (v2 R compression used for R versions 1.4-3.5)
download.file(url="https://ndownloader.figshare.com/files/23739737", destfile="SILVA_SSU_r138_2019.v2-R-compresssed.RData")

## loading reference taxonomy object
load("SILVA_SSU_r138_2019.v2-R-compresssed.RData")
```

Assigning taxonomy will take a long time in Binder, so we'll use the output we loaded before from `amplicon_dada2_ex.RData` to progress. Here is how we'd run DECIPHER

```
## loading DECIPHER
# BiocManager::install("DECIPHER")
#library(DECIPHER)
#packageVersion("DECIPHER") # v2.6.0 when this was put together

## creating DNAStringSet object of our ASVs
#dna <- DNAStringSet(getSequences(seqtab.nochim))

## and classifying
#tax_info <- IdTaxa(test=dna, trainingSet=trainingSet, strand="both", processors=NULL)
```

### Store outputs

Here is where we will store files to be accessed later. Maybe taking a look in Excel or using the sequence FASTA files for an analysis

```{r}
  # giving our seq headers more manageable names (ASV_1, ASV_2...)
asv_seqs <- colnames(seqtab.nochim)
asv_headers <- vector(dim(seqtab.nochim)[2], mode="character")

for (i in 1:dim(seqtab.nochim)[2]) {
  asv_headers[i] <- paste(">ASV", i, sep="_")
}

  # making and writing out a fasta of our final ASV seqs:
asv_fasta <- c(rbind(asv_headers, asv_seqs))
write(asv_fasta, "ASVs.fa")

  # count table:
asv_tab <- t(seqtab.nochim)
row.names(asv_tab) <- sub(">", "", asv_headers)
write.table(asv_tab, "ASVs_counts.tsv", sep="\t", quote=F, col.names=NA)

  # tax table:
  # creating table of taxonomy and setting any that are unclassified as "NA"
ranks <- c("domain", "phylum", "class", "order", "family", "genus", "species")
asv_tax <- t(sapply(tax_info, function(x) {
  m <- match(ranks, x$rank)
  taxa <- x$taxon[m]
  taxa[startsWith(taxa, "unclassified_")] <- NA
  taxa
}))
colnames(asv_tax) <- ranks
rownames(asv_tax) <- gsub(pattern=">", replacement="", x=asv_headers)

write.table(asv_tax, "ASVs_taxonomy.tsv", sep = "\t", quote=F, col.names=NA)
```

### Remove likely contaminants

From the folks who brought you DADA2, now introducing [decontam](https://doi.org/10.1186/s40168-018-0605-2) with accompanying [website](https://github.com/benjjneb/decontam)

By detailing that we have samples that represent extraction blanks - amplicon products generated from a DNA extraction without added sample - we can remove ASVs that are sourced from the extraction kit/lab and not from our precious samples!

```{r}
library(decontam)
packageVersion("decontam")

colnames(asv_tab) # our blanks are the first 4 of 20 samples in this case
vector_for_decontam <- c(rep(TRUE, 4), rep(FALSE, 16))

contam_df <- isContaminant(t(asv_tab), neg=vector_for_decontam)

table(contam_df$contaminant) # identified 6 as contaminants

  # getting vector holding the identified contaminant IDs
contam_asvs <- row.names(contam_df[contam_df$contaminant == TRUE, ])

# We only detect 6 contaminants. We see their taxonomic designations to make sure they
# are actually contaminants
asv_tax[row.names(asv_tax) %in% contam_asvs, ]
```

We can then remove them from a outputs that we just generated to have even more perfect sample files!

```{r}
  # making new fasta file
contam_indices <- which(asv_fasta %in% paste0(">", contam_asvs))
dont_want <- sort(c(contam_indices, contam_indices + 1))
asv_fasta_no_contam <- asv_fasta[- dont_want]

  # making new count table
asv_tab_no_contam <- asv_tab[!row.names(asv_tab) %in% contam_asvs, ]

  # making new taxonomy table
asv_tax_no_contam <- asv_tax[!row.names(asv_tax) %in% contam_asvs, ]

  ## and now writing them out to files
write(asv_fasta_no_contam, "ASVs-no-contam.fa")
write.table(asv_tab_no_contam, "ASVs_counts-no-contam.tsv",
            sep="\t", quote=F, col.names=NA)
write.table(asv_tax_no_contam, "ASVs_taxonomy-no-contam.tsv",
            sep="\t", quote=F, col.names=NA)
```

## Part 2 - Analysis in R

Make sure we are still in the right place

```{r}
setwd("~/dada2_amplicon_ex_workflow/")
list.files()
```

And load the libraries needed for this part

```{r}
library("phyloseq")
library("vegan")
library("DESeq2")
library("ggplot2")
library("dendextend")
library("tidyr")
library("viridis")
library("reshape")
```

We going to clear all of the R variables to start fresh and create a new table to contain our count data

```{r}
rm(list=ls())
  
count_tab <- read.table("ASVs_counts-no-contam.tsv", header=T, row.names=1,
             check.names=F, sep="\t")[ , -c(1:4)]
```

> Since we???ve already used decontam to remove likely contaminants, we???re dropping the ???blank??? samples from our count table which in the table we???re reading in are the first 4 columns. That???s what???s being done by the [ , -c(1:4)] part at the end there.

```{r}
tax_tab <- as.matrix(read.table("ASVs_taxonomy-no-contam.tsv", header=T,
           row.names=1, check.names=F, sep="\t"))

sample_info_tab <- read.table("sample_info.tsv", header=T, row.names=1,
                   check.names=F, sep="\t")
  
  # and setting the color column to be of type "character", which helps later
sample_info_tab$color <- as.character(sample_info_tab$color)

sample_info_tab
```

In `sample_info_tab` we have:

* 16 samples as rows
* 4 columns
  * ???temperature??? for the temperature of the venting water was where collected
  * ???type??? indicating if that sample is a blank, water sample, rock, or biofilm
  * a characteristics column called ???char??? that just serves to distinguish between the main types of rocks (glassy, altered, or carbonate)
  * ???color???, which has different R colors we can use later for plotting

This looks a lot like most tables you've seen in Excel or elsewhere - that's because you can make this table in Excel and import it into R

### Beta Diversity

Beta Diversity provides a measure of how similar samples are to each other, based on the underlying ASVs counts

To do this we calculate metrics such as distances or dissimilarities based on pairwise comparisons of samples. In this instance we'll use a Euclidean distance on normalized data. 

```{r}
  # first we need to make a DESeq2 object
deseq_counts <- DESeqDataSetFromMatrix(count_tab, colData = sample_info_tab, design = ~type) 
    # we have to include the "colData" and "design" arguments because they are 
    # required, as they are needed for further downstream processing by DESeq2, 
    # but for our purposes of simply transforming the data right now, they don't 
    # matter
  
deseq_counts_vst <- varianceStabilizingTransformation(deseq_counts)
    # NOTE: If you get this error here with your dataset: "Error in
    # estimateSizeFactorsForMatrix(counts(object), locfunc =locfunc, : every
    # gene contains at least one zero, cannot compute log geometric means", that
    # can be because the count table is sparse with many zeroes, which is common
    # with marker-gene surveys. In that case you'd need to use a specific
    # function first that is equipped to deal with that. You could run:
      # deseq_counts <- estimateSizeFactors(deseq_counts, type = "poscounts")
    # now followed by the transformation function:
      # deseq_counts_vst <- varianceStabilizingTransformation(deseq_counts)

  # and here is pulling out our transformed table
vst_trans_count_tab <- assay(deseq_counts_vst)

  # and calculating our Euclidean distance matrix
euc_dist <- dist(t(vst_trans_count_tab))
``` 

From this matrix we can visualize the relationship in a dendrogram

```{r}
euc_clust <- hclust(euc_dist, method="ward.D2")

  # hclust objects like this can be plotted with the generic plot() function
plot(euc_clust) 
    # but i like to change them to dendrograms for two reasons:
      # 1) it's easier to color the dendrogram plot by groups
      # 2) if wanted you can rotate clusters with the rotate() 
      #    function of the dendextend package

euc_dend <- as.dendrogram(euc_clust, hang=0.1)
dend_cols <- as.character(sample_info_tab$color[order.dendrogram(euc_dend)])
labels_colors(euc_dend) <- dend_cols

plot(euc_dend, ylab="VST Euc. dist.")
```
> The broadest clusters separate the biofilm, carbonate, and water samples from the basalt rocks, which are the black and brown labels. And those form two distinct clusters, with samples R8???R11 separate from the others (R1-R6, and R12). R8-R11 (black) were all of the glassier type of basalt with thin (~1-2 mm), smooth exteriors, while the rest (R1-R6, and R12; brown) had more highly altered, thick (>1 cm) outer rinds (excluding the oddball carbonate which isn???t a basalt, R7).

We can also visualize this relationship with an Ordination. This requires taking a complex matrix and reducing the number dimension so that we can visualize in on a flat surface. Principle coordinates analysis (PCoA) is a type of multidimensional scaling that operates on dissimilarities or distances. We???re still doing beta diversity here, we want to use our transformed table. So we???re going to make a `phyloseq` object with our DESeq2-transformed table and generate the PCoA from that.

```{r}
 # making our phyloseq object with transformed table
vst_count_phy <- otu_table(vst_trans_count_tab, taxa_are_rows=T)
sample_info_tab_phy <- sample_data(sample_info_tab)
vst_physeq <- phyloseq(vst_count_phy, sample_info_tab_phy)

  # generating and visualizing the PCoA with phyloseq
vst_pcoa <- ordinate(vst_physeq, method="MDS", distance="euclidean")
eigen_vals <- vst_pcoa$values$Eigenvalues # allows us to scale the axes according to their magnitude of separating apart the samples

plot_ordination(vst_physeq, vst_pcoa, color="char") + 
  geom_point(size=1) + labs(col="type") + 
  geom_text(aes(label=rownames(sample_info_tab), hjust=0.3, vjust=-0.4)) + 
  coord_fixed(sqrt(eigen_vals[2]/eigen_vals[1])) + ggtitle("PCoA") + 
  scale_color_manual(values=unique(sample_info_tab$color[order(sample_info_tab$char)])) + 
  theme(legend.position="none")
```
> This is just providing us with a different overview of how our samples relate to each other. Inferences that are consistent with the hierarchical clustering above can be considered a bit more robust if the same general trends emerge from both approaches. It???s important to remember that these are exploratory visualizations and do not say anything statistically about our samples.

### Alpha Diversity

Alpha diversity provides summary metrics that describes an individual sample. Commonly this can be thought of as "richness" - a metric that summarizes how many "species" are present and how samples look.

> Be cautious with estimators and extrapolations. These typically don???t work the same (i.e. are not statistically robust, and not biologically meaningful) with our current understanding of microbiolgy in comparison to macrobiology

#### Rarefaction curves
> It is not okay to use rarefaction curves to estimate total richness of a sample, or to extrapolate anything from them. But they can still be useful

Here we will use `vegan` to make a rarefaction curve

```{r}
rarecurve(t(count_tab), step=100, col=sample_info_tab$color, lwd=2, ylab="ASVs", label=F)

  # and adding a vertical line at the fewest seqs in any sample
abline(v=(min(rowSums(t(count_tab)))))
```

> In this plot, samples are colored the same way as above, and the black vertical line represents the sampling depth of the sample with the least amount of sequences (a bottom water sample, BW1, in this case). This view suggests that the rock samples (all of them) are more diverse and have a greater richness than the water samples or the biofilm sample ??? based on where they all cross the vertical line of lowest sampling depth, which is not necessarily predictive of where they???d end up had they been sampled to greater depth. And again, just focusing on the brown and black lines for the two types of basalts we have, they seem to show similar trends within their respective groups that suggest the more highly altered basalts (brown lines) may host more diverse microbial communities than the glassier basalts (black lines).

We???re going to plot Chao1 richness esimates and Shannon diversity values:

 * Chao1 is a richness estimator, ???richness??? being the total number of distinct units in your sample, ???distinct units??? being ASVs
 * Shannon???s diversity index is a metric of diversity. The term diversity includes ???richness??? (the total number of your distinct units) and ???evenness??? (the relative proportions of all of your distinct units).
 * Should not be considered ???true??? values of anything or be compared across studies

For these plots we'll use `phyloseq` again

```{r}
  # first we need to create a phyloseq object using our un-transformed count table
count_tab_phy <- otu_table(count_tab, taxa_are_rows=T)
tax_tab_phy <- tax_table(tax_tab)

ASV_physeq <- phyloseq(count_tab_phy, tax_tab_phy, sample_info_tab_phy)

  # and now we can call the plot_richness() function on our phyloseq object
plot_richness(ASV_physeq, color="char", measures=c("Chao1", "Shannon")) + 
  scale_color_manual(values=unique(sample_info_tab$color[order(sample_info_tab$char)])) +
  theme(legend.title = element_blank())
```

> Need to stress that these are not interpretable as ???real??? numbers of anything (due to the nature of amplicon data), but they can be useful as relative metrics of comparison. For example, we see from this that the more highly altered basalts seem to host communities that are more diverse and have a higher richness than the other rocks, and that the water and biofilm samples are less diverse than the rocks.

`phyloseq` is powerful in that it can help us easily visualize this data in other ways. For example grouping by sample type

```{r}
plot_richness(ASV_physeq, x="type", color="char", measures=c("Chao1", "Shannon")) + 
  scale_color_manual(values=unique(sample_info_tab$color[order(sample_info_tab$char)])) +
  theme(legend.title = element_blank())
```

Reminder that Chao1 is richness and Shannon is diversity. 
> Take for example the biofilm sample (green) above. It seems to have a higher estimated richness than the two water samples, but a lower Shannon diversity than both water samples. This suggests that the water samples likely have a greater ???evenness???; or to put it another way, even though the biofilm may have more biological units (our ASVs here), it may be largely dominated by only a few of them.

### Taxonomic summaries

We???ll make some broad-level summarization figures. Again using `phyloseq`, to start, we need to parse our count matrix by taxonomy. How to do this for other samples down will depend on your data and your question.

```{r}
  # using phyloseq to make a count table that has summed all ASVs
    # that were in the same phylum
phyla_counts_tab <- otu_table(tax_glom(ASV_physeq, taxrank="phylum")) 

  # making a vector of phyla names to set as row names
phyla_tax_vec <- as.vector(tax_table(tax_glom(ASV_physeq, taxrank="phylum"))[,2]) 
rownames(phyla_counts_tab) <- as.vector(phyla_tax_vec)

  # we also have to account for sequences that weren't assigned any
    # taxonomy even at the phylum level 
    # these came into R as 'NAs' in the taxonomy table, but their counts are
    # still in the count table
    # so we can get that value for each sample by substracting the column sums
    # of this new table (that has everything that had a phylum assigned to it)
    # from the column sums of the starting count table (that has all
    # representative sequences)
unclassified_tax_counts <- colSums(count_tab) - colSums(phyla_counts_tab)
  # and we'll add this row to our phylum count table:
phyla_and_unidentified_counts_tab <- rbind(phyla_counts_tab, "Unclassified"=unclassified_tax_counts)

  # now we'll remove the Proteobacteria, so we can next add them back in
    # broken down by class
temp_major_taxa_counts_tab <- phyla_and_unidentified_counts_tab[!row.names(phyla_and_unidentified_counts_tab) %in% "Proteobacteria", ]

  # making count table broken down by class (contains classes beyond the
    # Proteobacteria too at this point)
class_counts_tab <- otu_table(tax_glom(ASV_physeq, taxrank="class")) 

  # making a table that holds the phylum and class level info
class_tax_phy_tab <- tax_table(tax_glom(ASV_physeq, taxrank="class")) 

phy_tmp_vec <- class_tax_phy_tab[,2]
class_tmp_vec <- class_tax_phy_tab[,3]
rows_tmp <- row.names(class_tax_phy_tab)
class_tax_tab <- data.frame("phylum"=phy_tmp_vec, "class"=class_tmp_vec, row.names = rows_tmp)

  # making a vector of just the Proteobacteria classes
proteo_classes_vec <- as.vector(class_tax_tab[class_tax_tab$phylum == "Proteobacteria", "class"])

  # changing the row names like above so that they correspond to the taxonomy,
    # rather than an ASV identifier
rownames(class_counts_tab) <- as.vector(class_tax_tab$class) 

  # making a table of the counts of the Proteobacterial classes
proteo_class_counts_tab <- class_counts_tab[row.names(class_counts_tab) %in% proteo_classes_vec, ] 

  # there are also possibly some some sequences that were resolved to the level
    # of Proteobacteria, but not any further, and therefore would be missing from
    # our class table
    # we can find the sum of them by subtracting the proteo class count table
    # from just the Proteobacteria row from the original phylum-level count table
proteo_no_class_annotated_counts <- phyla_and_unidentified_counts_tab[row.names(phyla_and_unidentified_counts_tab) %in% "Proteobacteria", ] - colSums(proteo_class_counts_tab)

  # now combining the tables:
major_taxa_counts_tab <- rbind(temp_major_taxa_counts_tab, proteo_class_counts_tab, "Unresolved_Proteobacteria"=proteo_no_class_annotated_counts)

  # and to check we didn't miss any other sequences, we can compare the column
    # sums to see if they are the same
    # if "TRUE", we know nothing fell through the cracks
identical(colSums(major_taxa_counts_tab), colSums(count_tab)) 

  # now we'll generate a proportions table for summarizing:
major_taxa_proportions_tab <- apply(major_taxa_counts_tab, 2, function(x) x/sum(x)*100)

  # if we check the dimensions of this table at this point
dim(major_taxa_proportions_tab)
  # we see there are currently 42 rows, which might be a little busy for a
    # summary figure
    # many of these taxa make up a very small percentage, so we're going to
    # filter some out
    # this is a completely arbitrary decision solely to ease visualization and
    # intepretation, entirely up to your data and you
    # here, we'll only keep rows (taxa) that make up greater than 5% in any
    # individual sample
temp_filt_major_taxa_proportions_tab <- data.frame(major_taxa_proportions_tab[apply(major_taxa_proportions_tab, 1, max) > 5, ])
  # checking how many we have that were above this threshold
dim(temp_filt_major_taxa_proportions_tab) 
    # now we have 12, much more manageable for an overview figure

  # though each of the filtered taxa made up less than 5% alone, together they
    # may add up and should still be included in the overall summary
    # so we're going to add a row called "Other" that keeps track of how much we
    # filtered out (which will also keep our totals at 100%)
filtered_proportions <- colSums(major_taxa_proportions_tab) - colSums(temp_filt_major_taxa_proportions_tab)
filt_major_taxa_proportions_tab <- rbind(temp_filt_major_taxa_proportions_tab, "Other"=filtered_proportions)
```

Lots of R to manipulate the data is a way that is logical for analysis. Conceptually, we can have manipulated this table in Excel. Combined columns and names. Do some addition and subtraction, but the goal of code like this is the replicability. Now, if I had to go back and make a change to the ASV calculation step, I could regenerate this data without having to do manual work in Excel.

We???ll make some stacked bar charts, boxplots, and some pie charts. We???ll use `ggplot2` to do this, and for these types of plots it seems to be easiest to work with tables in narrow format. We???ll see what that means, how to transform the table, and then add some information for the samples to help with plotting.

```{r}
  # first let's make a copy of our table that's safe for manipulating
filt_major_taxa_proportions_tab_for_plot <- filt_major_taxa_proportions_tab

  # and add a column of the taxa names so that it is within the table, rather
  # than just as row names (this makes working with ggplot easier)
filt_major_taxa_proportions_tab_for_plot$Major_Taxa <- row.names(filt_major_taxa_proportions_tab_for_plot)

  # now we'll transform the table into narrow, or long, format (also makes
  # plotting easier)
filt_major_taxa_proportions_tab_for_plot.g <- gather(filt_major_taxa_proportions_tab_for_plot, Sample, Proportion, -Major_Taxa)

  # take a look at the new table and compare it with the old one
head(filt_major_taxa_proportions_tab_for_plot.g)
head(filt_major_taxa_proportions_tab_for_plot)
    # manipulating tables like this is something you may need to do frequently in R

  # now we want a table with "color" and "characteristics" of each sample to
    # merge into our plotting table so we can use that more easily in our plotting
    # function
    # here we're making a new table by pulling what we want from the sample
    # information table
sample_info_for_merge<-data.frame("Sample"=row.names(sample_info_tab), "char"=sample_info_tab$char, "color"=sample_info_tab$color, stringsAsFactors=F)

  # and here we are merging this table with the plotting table we just made
    # (this is an awesome function!)
filt_major_taxa_proportions_tab_for_plot.g2 <- merge(filt_major_taxa_proportions_tab_for_plot.g, sample_info_for_merge)

  # and now we're ready to make some summary figures with our wonderfully
    # constructed table

  ## a good color scheme can be hard to find, i included the viridis package
    ## here because it's color-blind friendly and sometimes it's been really
    ## helpful for me, though this is not demonstrated in all of the following :/ 

  # one common way to look at this is with stacked bar charts for each taxon per sample:
ggplot(filt_major_taxa_proportions_tab_for_plot.g2, aes(x=Sample, y=Proportion, fill=Major_Taxa)) +
  geom_bar(width=0.6, stat="identity") +
  theme_bw() +
  theme(axis.text.x=element_text(angle=90, vjust=0.4, hjust=1), legend.title=element_blank()) +
  labs(x="Sample", y="% of 16S rRNA gene copies recovered", title="All samples")
```

Another way to look would be using boxplots where each box is a major taxon, with each point being colored based on its sample type.

```{r}
ggplot(filt_major_taxa_proportions_tab_for_plot.g2, aes(Major_Taxa, Proportion)) +
  geom_jitter(aes(color=factor(char)), size=2, width=0.15, height=0) +
  scale_color_manual(values=unique(filt_major_taxa_proportions_tab_for_plot.g2$color[order(filt_major_taxa_proportions_tab_for_plot.g2$char)])) +
  geom_boxplot(fill=NA, outlier.color=NA) +
  theme(axis.text.x=element_text(angle=90, vjust=0.4, hjust=1), legend.title=element_blank()) +
  labs(x="Major Taxa", y="% of 16S rRNA gene copies recovered", title="All samples")
```

> This is a very coarse level of resolution as we are using taxonomic classifications at the phylum and class ranks. This is why things may look more similar between the rocks and water samples than you might expect, and why when looking at the ASV level ??? like we did with the exploratory visualizations above ??? we can see more clearly that these do in fact host distinct communities.

Another way to look at this would be to plot the water and rock samples separately, which might help tighten up some taxa boxplots if they have a different distribution between the two sample types.

```{r}
  # let's set some helpful variables first:
bw_sample_IDs <- row.names(sample_info_tab)[sample_info_tab$type == "water"]
rock_sample_IDs <- row.names(sample_info_tab)[sample_info_tab$type == "rock"]

  # first we need to subset our plotting table to include just the rock samples to plot
filt_major_taxa_proportions_rocks_only_tab_for_plot.g <- filt_major_taxa_proportions_tab_for_plot.g2[filt_major_taxa_proportions_tab_for_plot.g2$Sample %in% rock_sample_IDs, ]
  # and then just the water samples
filt_major_taxa_proportions_water_samples_only_tab_for_plot.g <- filt_major_taxa_proportions_tab_for_plot.g2[filt_major_taxa_proportions_tab_for_plot.g2$Sample %in% bw_sample_IDs, ]

  # and now we can use the same code as above just with whatever minor alterations we want
  # rock samples
ggplot(filt_major_taxa_proportions_rocks_only_tab_for_plot.g, aes(Major_Taxa, Proportion)) +
  scale_y_continuous(limits=c(0,50)) + # adding a setting for the y axis range so the rock and water plots are on the same scale
  geom_jitter(aes(color=factor(char)), size=2, width=0.15, height=0) +
  scale_color_manual(values=unique(filt_major_taxa_proportions_rocks_only_tab_for_plot.g$color[order(filt_major_taxa_proportions_rocks_only_tab_for_plot.g$char)])) +
  geom_boxplot(fill=NA, outlier.color=NA) +
  theme(axis.text.x=element_text(angle=90, vjust=0.4, hjust=1), legend.position="top", legend.title=element_blank()) + # moved legend to top 
  labs(x="Major Taxa", y="% of 16S rRNA gene copies recovered", title="Rock samples only")

  # water samples
ggplot(filt_major_taxa_proportions_water_samples_only_tab_for_plot.g, aes(Major_Taxa, Proportion)) +
  scale_y_continuous(limits=c(0,50)) + # adding a setting for the y axis range so the rock and water plots are on the same scale
  geom_jitter(aes(color=factor(char)), size=2, width=0.15, height=0) +
  scale_color_manual(values=unique(filt_major_taxa_proportions_water_samples_only_tab_for_plot.g$color[order(filt_major_taxa_proportions_water_samples_only_tab_for_plot.g$char)])) +
  geom_boxplot(fill=NA, outlier.color=NA) +
  theme(axis.text.x=element_text(angle=90, vjust=0.4, hjust=1), legend.position="top", legend.title=element_blank()) + # moved legend to top 
  labs(x="Major Taxa", y="% of 16S rRNA gene copies recovered", title="Bottom water samples only")
```

### Betadisper and permutational ANOVA

> Here we are going to test if there is a statistically signficant difference between our sample types. One way to do this is with the `betadisper` and `adonis` functions from the `vegan` package. adonis can tell us if there is a statistical difference between groups, but it has an assumption that must be met that we first need to check with `betadisper`, and that is that there is a sufficient level of homogeneity of dispersion within groups. If there is not, then `adonis` can be unreliable.

```{r}
anova(betadisper(euc_dist, sample_info_tab$type))
```

We are looking for signficance (<0.05). This tells us that there is a difference between group dispersions, which means that we can???t trust the results of an `adonis` (permutational ANOVA) test on this, because the assumption of homogenous within-group disperions is not met.

> But a more interesting and specific question is ???Do the rocks differ based on their level of exterior alteration???? So let???s try this looking at just the basalt rocks, based on their characteristics of glassy and altered.

```{r}
  # first we'll need to go back to our transformed table, and generate a
    # distance matrix only incorporating the basalt samples
      # and to help with that I'm making a variable that holds all basalt rock
      # names (just removing the single calcium carbonate sample, R7)
basalt_sample_IDs <- rock_sample_IDs[!rock_sample_IDs %in% "R7"]

  # new distance matrix of only basalts
basalt_euc_dist <- dist(t(vst_trans_count_tab[ , colnames(vst_trans_count_tab) %in% basalt_sample_IDs]))

  # and now making a sample info table with just the basalts
basalt_sample_info_tab <- sample_info_tab[row.names(sample_info_tab) %in% basalt_sample_IDs, ]

  # running betadisper on just these based on level of alteration as shown in the images above:
anova(betadisper(basalt_euc_dist, basalt_sample_info_tab$char))
```

We do not find a significant difference between their within-group dispersions (0.7). So we can now test if the groups host statistically different communities based on adonis, having met this assumption.

```{r}
adonis(basalt_euc_dist~basalt_sample_info_tab$char)
```

This gives us our first statistical evidence that there is actually a difference in microbial communities hosted by the more highly altered basalts as compared to the glassier less altered basalts

We can visualize these results

```{r}
 # making our phyloseq object with transformed table
basalt_vst_count_phy <- otu_table(vst_trans_count_tab[, colnames(vst_trans_count_tab) %in% basalt_sample_IDs], taxa_are_rows=T)
basalt_sample_info_tab_phy <- sample_data(basalt_sample_info_tab)
basalt_vst_physeq <- phyloseq(basalt_vst_count_phy, basalt_sample_info_tab_phy)

  # generating and visualizing the PCoA with phyloseq
basalt_vst_pcoa <- ordinate(basalt_vst_physeq, method="MDS", distance="euclidean")
basalt_eigen_vals <- basalt_vst_pcoa$values$Eigenvalues # allows us to scale the axes according to their magnitude of separating apart the samples

  # and making our new ordination of just basalts with our adonis statistic
plot_ordination(basalt_vst_physeq, basalt_vst_pcoa, color="char") + 
  labs(col="type") + geom_point(size=1) + 
  geom_text(aes(label=rownames(basalt_sample_info_tab), hjust=0.3, vjust=-0.4)) + 
  annotate("text", x=25, y=73, label="Highly altered vs glassy") +
  annotate("text", x=25, y=67, label="Permutational ANOVA = 0.005") + 
  coord_fixed(sqrt(basalt_eigen_vals[2]/basalt_eigen_vals[1])) + ggtitle("PCoA - basalts only") + 
  scale_color_manual(values=unique(basalt_sample_info_tab$color[order(basalt_sample_info_tab$char)])) + 
  theme(legend.position="none")
```

### Differential abundance analysis with DESeq2

> Recovered 16S rRNA gene copy numbers do not equal organism abundance. Recovered 16S rRNA gene copy numbers do represent numbers of recovered 16S rRNA gene copies.

We can perform differential abundance testing to test for which representative sequences have significantly different copy-number counts between samples ??? which can be useful information and guide the generation of hypotheses. We'll use `DESeq2`.

Now that we???ve found a statistical difference between our two rock samples, this is one way we can try to find out which ASVs (and possibly which taxa) are contributing to that difference.

```{r}
# first making a basalt-only phyloseq object of non-transformed values (as that is what DESeq2 operates on
basalt_count_phy <- otu_table(count_tab[, colnames(count_tab) %in% basalt_sample_IDs], taxa_are_rows=T)
basalt_count_physeq <- phyloseq(basalt_count_phy, basalt_sample_info_tab_phy)
  
  # now converting our phyloseq object to a deseq object
basalt_deseq <- phyloseq_to_deseq2(basalt_count_physeq, ~char)

  # and running deseq standard analysis:
basalt_deseq <- DESeq(basalt_deseq)
```

```{r}
 # pulling out our results table, we specify the object, the p-value we are going to use to filter our results, and what contrast we want to consider by first naming the column, then the two groups we care about
deseq_res_altered_vs_glassy <- results(basalt_deseq, alpha=0.01, contrast=c("char", "altered", "glassy"))

  # we can get a glimpse at what this table currently holds with the summary command
summary(deseq_res_altered_vs_glassy) 
    # this tells us out of ~1,800 ASVs, with adj-p < 0.01, there are 7 increased when comparing altered basalts to glassy basalts, and about 6 decreased
    # "decreased" in this case means at a lower count abundance in the altered basalts than in the glassy basalts, and "increased" means greater proportion in altered than in glassy
    # remember, this is done with a drastically reduced dataset, which can be hindering the capabilities here

  # let's subset this table to only include these that pass our specified significance level
sigtab_res_deseq_altered_vs_glassy <- deseq_res_altered_vs_glassy[which(deseq_res_altered_vs_glassy$padj < 0.01), ]

  # now we can see this table only contains those we consider significantly differentially abundant
summary(sigtab_res_deseq_altered_vs_glassy) 

  # next let's stitch that together with these ASV's taxonomic annotations for a quick look at both together
sigtab_deseq_altered_vs_glassy_with_tax <- cbind(as(sigtab_res_deseq_altered_vs_glassy, "data.frame"), as(tax_table(ASV_physeq)[row.names(sigtab_res_deseq_altered_vs_glassy), ], "matrix"))

  # and now let's sort that table by the baseMean column
sigtab_deseq_altered_vs_glassy_with_tax[order(sigtab_deseq_altered_vs_glassy_with_tax$baseMean, decreasing=T), ]

  # this puts sequence derived from an Actinobacteria at the top that was detected in ~10 log2fold greater abundance in the glassy basalts than in the more highly altered basalts
```

> If you glance through the taxonomy of our significant table here, you???ll see several have the same designations. It???s possible this is one of those cases where the single-nucleotide resolution approach more inhibits your cause than helps it. Another way to look at this would be to sum the ASVs by the same genus designations, or to go back and cluster them into some form of OTU (after identifying ASVs) ??? in which case you???d still be using the ASV units, but then clustering them at some arbitrary level to see if that level of resolution is more revealing for the system you???re looking at.