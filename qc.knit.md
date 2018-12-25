---
title: Elaborating on cell-based quality control in single-cell RNA-seq data
author: 
- name: Aaron T. L. Lun
  affiliation: &CRUK Cancer Research UK Cambridge Institute, Li Ka Shing Centre, Robinson Way, Cambridge CB2 0RE, United Kingdom
- name: Davis J. McCarthy
  affiliation: 
  - &EMBL EMBL European Bioinformatics Institute, Wellcome Genome Campus, Hinxton, Cambridge CB10 1SD, United Kingdom
  - St Vincent's Institute of Medical Research, 41 Victoria Parade, Fitzroy, Victoria 3065, Australia
- name: John C. Marioni
  affiliation: 
  - *CRUK
  - *EMBL
  - Wellcome Trust Sanger Institute, Wellcome Genome Campus, Hinxton, Cambridge CB10 1SA, United Kingdom
date: "2018-11-16"
vignette: >
  %\VignetteIndexEntry{06. Quality control details}
  %\VignetteEngine{knitr::rmarkdown}
  %\VignetteEncoding{UTF-8}    
output: 
  BiocStyle::html_document:
    titlecaps: false
    toc_float: true
bibliography: ref.bib
---



# Overview

Low-quality cells can often yield misleading results in downstream analyses, by:

- forming their own distinct cluster(s), complicating interpretation of the results.
This can be most obviously driven by increased mitochondrial proportions or enrichment for nuclear RNAs after cell damage.
However, very small libraries can also form their own clusters due to shifts in the mean upon log-transformation.
- containing genes that appear to be strongly "upregulated" due to the presence of very small size factors.
This is most problematic if some transcripts are present at constant levels in the ambient solution for all cells (i.e., wells or droplets).
Small counts will then be greatly inflated upon normalization with these size factors.
- containing genes that appear to be strongly "downregulated" due to the loss of RNA upon cell damage.
This seems most pronounced with ribosomal protein genes, though other cytoplasmic transcripts are likely to be affected.
- distorting the characterization of population heterogeneity during variance estimation or principal components analysis.
The first few principal components will capture differences in quality rather than biology, reducing the effectiveness of dimensionality reduction.
Similarly, genes with the largest variances will be driven by differences between low- and high-quality cells.

As such, we need to remove these cells at the start of the analysis.
Recall that we were defining low-quality cells as those with outlier values for various quality control (QC) metrics,
using the `isOutlier()` and `calculateQCMetrics()` functions from the *[scater](https://bioconductor.org/packages/3.9/scater)* package [@mccarthy2017scater].
Here, we will examine some of the reasoning behind the outlier-based QC in more detail.

# Assumptions of outlier identification 

An outlier-based definition for low-quality cells assumes that most cells are of high quality.
This is usually reasonable and can be experimentally supported in some situations by visually checking that the cells are intact, e.g., on the microwell plate.
Another assumption is that the QC metrics are independent on the biological state of each cell.
This ensures that any outlier values for these metrics are driven by technical factors rather than biological processes.
Thus, removing cells based on the metrics will not misrepresent the biology in downstream analyses.

The second assumption is most likely to be violated in highly heterogeneous cell populations.
For example, some cell types may naturally have less RNA or express fewer genes than other cell types.
Such cell types are more likely to be considered outliers and removed, even if they are of high quality.
The use of the MAD mitigates this problem by accounting for biological variability in the QC metrics.
A heterogeneous population should have higher variability in the metrics among high-quality cells, increasing the MAD and reducing the chance of incorrectly removing particular cell types (at the cost of reducing power to remove low-quality cells).
Nonetheless, filtering based on outliers may not be appropriate in extreme cases where one cell type is very different from the others.

Systematic differences in the QC metrics can be handled to some extent using the `batch=` argument in the `isOutlier()` function.
For example, setting `batch` to the plate of origin will identify outliers within each level of `batch`, using plate-specific median and MAD estimates.
This is obviously useful for accommodating known differences in experimental processing, e.g., sequencing at different depth or different amounts of added spike-in RNA. 
We can also include biological factors in `batch`, if those factors could result in systematically fewer expressed genes or lower RNA content.
However, this is not applicable in experiments where the factors are not known in advance.

# Checking for discarded cell types

## In the 416B data set 

We can diagnose loss of distinct cell types during QC by looking for differences in gene expression between the discarded and retained cells.
To demonstrate, we compute the average count across the discarded and retained pools in the 416B data set.


```r
library(SingleCellExperiment)
sce.full.416b <- readRDS("416B_preQC.rds")

library(scater)
lost <- calcAverage(counts(sce.full.416b)[,!sce.full.416b$PassQC])
kept <- calcAverage(counts(sce.full.416b)[,sce.full.416b$PassQC])
```

If the discarded pool is enriched for a certain cell type, we should observe increased expression of the corresponding marker genes.
No systematic upregulation of genes is apparent in the discarded pool in Figure \@ref(fig:discardplot416b), 
indicating that the QC step did not inadvertently filter out a cell type in the 416B dataset.


```r
# Avoid loss of points where either average is zero.
capped.lost <- pmax(lost, min(lost[lost>0]))
capped.kept <- pmax(kept, min(kept[kept>0]))

plot(capped.lost, capped.kept, xlab="Average count (discarded)", 
    ylab="Average count (retained)", log="xy", pch=16)
is.spike <- isSpike(sce.full.416b)
points(capped.lost[is.spike], capped.kept[is.spike], col="red", pch=16)
is.mito <- rowData(sce.full.416b)$is_feature_control_Mt
points(capped.lost[is.mito], capped.kept[is.mito], col="dodgerblue", pch=16)
```

<div class="figure">
<img src="/home/cri.camres.org/lun01/AaronDocs/Research/simpleSingleCell/results/xtra-1-qc_files/figure-html/discardplot416b-1.png" alt="Average counts across all discarded and retained cells in the 416B dataset. Each point represents a gene, with spike-in and mitochondrial transcripts in red and blue respectively." width="100%" />
<p class="caption">(\#fig:discardplot416b)Average counts across all discarded and retained cells in the 416B dataset. Each point represents a gene, with spike-in and mitochondrial transcripts in red and blue respectively.</p>
</div>

We examine this more closely by computing log-fold changes between the average counts of the two pools.
The `predFC` function stabilizes the log-fold change estimates by adding a prior count to the average of each pool.
We only examine the log-fold changes rather than formally testing for differential expression, as we are not interested in penalizing intra-pool heterogeneity.


```r
library(edgeR)
coefs <- predFC(cbind(lost, kept), design=cbind(1, c(1, 0)))[,2]
info <- data.frame(logFC=coefs, Lost=lost, Kept=kept, 
    row.names=rownames(sce.full.416b))
head(info[order(info$logFC, decreasing=TRUE),], 20)
```

```
##                       logFC      Lost        Kept
## ENSMUSG00000104647 6.844237  7.515034 0.000000000
## Nmur1              6.500909  5.909533 0.000000000
## Retn               6.250501 10.333172 0.196931018
## Fut9               6.096356  4.696897 0.010132032
## ENSMUSG00000102352 6.077614  9.393793 0.206637368
## ENSMUSG00000102379 6.029758  4.244690 0.000000000
## 1700101I11Rik      5.828821  4.483094 0.039199404
## Gm4952             5.698380  6.580862 0.172999108
## ENSMUSG00000106680 5.670156  3.389236 0.005234611
## ENSMUSG00000107955 5.554616  5.268508 0.132532601
## Gramd1c            5.446975  4.435342 0.103783669
## Jph3               5.361082  4.696897 0.139188080
## ENSMUSG00000092418 5.324462  3.395752 0.056931488
## 1700029I15Rik      5.316226  8.199510 0.394588776
## Pih1h3b            5.307439  2.546814 0.000000000
## ENSMUSG00000097176 5.275459  2.541927 0.003772842
## Olfr456            5.093383  2.186536 0.000000000
## ENSMUSG00000103731 5.017303  3.315016 0.107148909
## Klhdc8b            4.933215 19.861036 1.635081878
## ENSMUSG00000082449 4.881422  1.878759 0.000000000
```

Again, no obvious cell type markers are present in the top set of genes upregulated in the discarded pool.
The magnitude of the log-fold changes is less important, attributable to imprecision with few cells in the discarded pool.
Large log-fold changes can also be driven by enrichment or depletion of mitochondrial, ribosomal protein or nuclear genes upon cell damage.

## In the PBMC data set

For comparison, we consider the PBMC data set in which we previously identified a platelet population
(see the [previous workflow](https://bioconductor.org/packages/3.9/simpleSingleCell/vignettes/work-3-tenx.html#marker-gene-detection)).
Recall that we relied on the use of the `emptyDrops()` method from the *[DropletUtils](https://bioconductor.org/packages/3.9/DropletUtils)* package to retain the platelets.
In contrast, if we had used a naive threshold on the total unique molecular identifier (UMI) count, we would have removed this population during the cell calling step.


```r
sce.pbmc <- readRDS("pbmc_data.rds")
wrong.keep <- sce.pbmc$total_counts >= 1000

lost <- calcAverage(counts(sce.pbmc)[,!wrong.keep])
kept <- calcAverage(counts(sce.pbmc)[,wrong.keep])
```

The presence of a distinct population in the discarded pool manifests in Figure \@ref(fig:discardplotpbmc) as a shift to the bottom-right for a number of genes.
This includes _PF4_, _PPBP_ and _SDPR_ that are strongly upregulated in the platelets.


```r
# Avoid loss of points where either average is zero.
capped.lost <- pmax(lost, min(lost[lost>0]))
capped.kept <- pmax(kept, min(kept[kept>0]))

plot(capped.lost, capped.kept, xlab="Average count (discarded)", 
    ylab="Average count (retained)", log="xy", pch=16)
platelet <- c("PF4", "PPBP", "SDPR")
points(capped.lost[platelet], capped.kept[platelet], col="orange", pch=16)
```

<div class="figure">
<img src="/home/cri.camres.org/lun01/AaronDocs/Research/simpleSingleCell/results/xtra-1-qc_files/figure-html/discardplotpbmc-1.png" alt="Average counts across all discarded and retained cells in the PBMC dataset, after using a more stringent filter on the total UMI count. Each point represents a gene, with platelet-related genes highlighted in orange." width="100%" />
<p class="caption">(\#fig:discardplotpbmc)Average counts across all discarded and retained cells in the PBMC dataset, after using a more stringent filter on the total UMI count. Each point represents a gene, with platelet-related genes highlighted in orange.</p>
</div>

These platelet-specific genes are also present among the top set of positive log-fold changes. 


```r
coefs <- predFC(cbind(lost, kept), design=cbind(1, c(1, 0)))[,2]
info <- data.frame(logFC=coefs, Lost=lost, Kept=kept, 
    row.names=rownames(sce.pbmc))
head(info[order(info$logFC, decreasing=TRUE),], 20)
```

```
##              logFC      Lost       Kept
## PF4       6.260123 3.1515268 0.16839563
## PPBP      6.193032 3.7362778 0.26331806
## HIST1H2AC 5.937488 2.3556043 0.14358496
## GNG11     5.829776 1.9102028 0.09765613
## SDPR      5.639884 1.6587397 0.09518196
## TUBB1     5.352245 1.3204857 0.08722296
## CLU       5.185106 1.0455393 0.05373447
## ACRBP     4.960898 0.8893300 0.05303073
## NRGN      4.757928 0.9803744 0.12727449
## RGS18     4.638017 1.2305668 0.25363327
## MAP3K7CL  4.526863 0.7415279 0.08998410
## SPARC     4.282294 0.4888558 0.02470512
## MMD       4.250060 0.5552700 0.06362420
## PGRMC1    4.142610 0.5466903 0.08141040
## CMTM5     3.811447 0.3331150 0.01588417
## TSC22D1   3.769813 0.3821595 0.05722460
## HRAT92    3.761280 0.3140556 0.01106262
## GP9       3.739236 0.3436135 0.03580096
## ITGA2B    3.703923 0.3090122 0.01700070
## CTSA      3.635665 0.6181223 0.26569357
```

## Avoiding loss of cell types

If cell types are being incorrectly discarded, the most direct solution is to relax the QC filters by increasing `nmads=` in the `isOutlier()` calls.
We can also avoid filtering on metrics that are associated with genuine biological differences between cell types.
The most extreme approach would be to not perform any QC filtering at all, thus guaranteeing that all cell types in the data are retained.
However, this obviously comes with an increased risk of retaining more low-quality damaged cells.
Such cells will cause problems in downstream analyses as discussed above, which motivates the use of a more strict filter (at least on the first pass) in our workflows.

As an aside, it is worth mentioning that the true technical quality of a cell may be correlated with its type.
(This differs from a correlation between the cell type and the QC metrics, as the latter are our imperfect proxies for quality.)
This can arise if some cell types are not amenable to dissociation or microfluidics handling during the scRNA-seq protocol.
In such cases, it is possible to correctly discard an entire cell type during QC if all of its members are damaged.
Indeed, concerns over the computational removal of cell types during QC are probably minor compared to losses in the experimental protocol.

# Alternative approaches to quality control

## Using fixed thresholds

One alternative strategy is to set pre-defined thresholds on each QC metric.
For example, we might remove all cells with library sizes below 100000 and numbers of expressed genes below 4000.
This avoids any assumptions associated with the use of outliers to identify low-quality cells.
However, it generally requires considerable experience to determine appropriate thresholds for each experimental protocol and biological system.
For example, thresholds for read count-based data are simply not applicable for UMI-based data, and vice versa.
Indeed, even with the same protocol and system, the appropriate threshold can vary from run to run due to the vagaries of RNA capture and sequencing.

## Using PCA-based outliers

Another strategy is to perform a principal components analysis (PCA) based on the quality metrics for each cell, e.g., the total number of reads, the total number of features and the proportion of mitochondrial or spike-in reads.
Outliers on a PCA plot may be indicative of low-quality cells that have aberrant technical properties compared to the (presumed) majority of high-quality cells.
This is demonstrated below on a brain cell dataset from @tasic2016adult, using functions from *[scater](https://bioconductor.org/packages/3.9/scater)*.


```r
# Obtaining the dataset.
library(scRNAseq)
data(allen)

# Setting up the data.
sce.allen <- as(allen, "SingleCellExperiment")
assayNames(sce.allen) <- "counts"
isSpike(sce.allen, "ERCC") <- grep("ERCC", rownames(sce.allen))

# Computing the QC metrics and running PCA.
library(scater)
sce.allen <- calculateQCMetrics(sce.allen)
sce.allen <- runPCA(sce.allen, use_coldata=TRUE, detect_outliers=TRUE)
table(sce.allen$outlier)
```

```
## 
## FALSE  TRUE 
##   374     5
```

Methods like PCA-based outlier detection and support vector machines can provide more power to distinguish low-quality cells from high-quality counterparts [@ilicic2016classification].
This is because they are able to detect subtle patterns across many quality metrics simultaneously. 
However, this comes at some cost to interpretability, as the reason for removing a given cell may not always be obvious.
Users interested in the more sophisticated approaches are referred to the *[scater](https://bioconductor.org/packages/3.9/scater)* and *[cellity](https://bioconductor.org/packages/3.9/cellity)* packages.

## Using the gene expression profiles

For completeness, we note that outliers can also be identified from the gene expression profiles, rather than QC metrics.
We consider this to be a risky strategy as it can remove high-quality cells in rare populations.
Even if subpopulations are explicitly captured with a mixture model, removal of outlier cells will simply reinforce the existing model.
This may be misleading if it understates the biological heterogeneity in each population.

# Concluding remarks 

All software packages used in this workflow are publicly available from the Comprehensive R Archive Network (https://cran.r-project.org) or the Bioconductor project (http://bioconductor.org).
The specific version numbers of the packages used are shown below, along with the version of the R installation.


```r
sessionInfo()
```

```
## R Under development (unstable) (2018-11-02 r75535)
## Platform: x86_64-pc-linux-gnu (64-bit)
## Running under: Ubuntu 16.04.5 LTS
## 
## Matrix products: default
## BLAS: /home/cri.camres.org/lun01/Software/R/trunk/lib/libRblas.so
## LAPACK: /home/cri.camres.org/lun01/Software/R/trunk/lib/libRlapack.so
## 
## locale:
##  [1] LC_CTYPE=en_GB.UTF-8       LC_NUMERIC=C              
##  [3] LC_TIME=en_GB.UTF-8        LC_COLLATE=en_GB.UTF-8    
##  [5] LC_MONETARY=en_GB.UTF-8    LC_MESSAGES=en_GB.UTF-8   
##  [7] LC_PAPER=en_GB.UTF-8       LC_NAME=C                 
##  [9] LC_ADDRESS=C               LC_TELEPHONE=C            
## [11] LC_MEASUREMENT=en_GB.UTF-8 LC_IDENTIFICATION=C       
## 
## attached base packages:
## [1] parallel  stats4    stats     graphics  grDevices utils     datasets 
## [8] methods   base     
## 
## other attached packages:
##  [1] scRNAseq_1.9.0                        
##  [2] edgeR_3.25.0                          
##  [3] Matrix_1.2-15                         
##  [4] org.Hs.eg.db_3.7.0                    
##  [5] EnsDb.Hsapiens.v86_2.99.0             
##  [6] ensembldb_2.7.2                       
##  [7] AnnotationFilter_1.7.0                
##  [8] DropletUtils_1.3.1                    
##  [9] pheatmap_1.0.10                       
## [10] cluster_2.0.7-1                       
## [11] dynamicTreeCut_1.63-1                 
## [12] limma_3.39.1                          
## [13] scran_1.11.4                          
## [14] scater_1.11.2                         
## [15] ggplot2_3.1.0                         
## [16] TxDb.Mmusculus.UCSC.mm10.ensGene_3.4.0
## [17] GenomicFeatures_1.35.1                
## [18] org.Mm.eg.db_3.7.0                    
## [19] AnnotationDbi_1.45.0                  
## [20] SingleCellExperiment_1.5.0            
## [21] SummarizedExperiment_1.13.0           
## [22] DelayedArray_0.9.0                    
## [23] BiocParallel_1.17.1                   
## [24] matrixStats_0.54.0                    
## [25] Biobase_2.43.0                        
## [26] GenomicRanges_1.35.1                  
## [27] GenomeInfoDb_1.19.1                   
## [28] IRanges_2.17.1                        
## [29] S4Vectors_0.21.4                      
## [30] BiocGenerics_0.29.1                   
## [31] bindrcpp_0.2.2                        
## [32] BiocFileCache_1.7.0                   
## [33] dbplyr_1.2.2                          
## [34] knitr_1.20                            
## [35] BiocStyle_2.11.0                      
## 
## loaded via a namespace (and not attached):
##   [1] readxl_1.1.0             backports_1.1.2         
##   [3] plyr_1.8.4               igraph_1.2.2            
##   [5] lazyeval_0.2.1           splines_3.6.0           
##   [7] sp_1.3-1                 digest_0.6.18           
##   [9] htmltools_0.3.6          viridis_0.5.1           
##  [11] magrittr_1.5             memoise_1.1.0           
##  [13] openxlsx_4.1.0           Biostrings_2.51.1       
##  [15] prettyunits_1.0.2        colorspace_1.3-2        
##  [17] blob_1.1.1               rappdirs_0.3.1          
##  [19] rrcov_1.4-4              haven_1.1.2             
##  [21] xfun_0.4                 dplyr_0.7.8             
##  [23] crayon_1.3.4             RCurl_1.95-4.11         
##  [25] bindr_0.1.1              survival_2.43-1         
##  [27] zoo_1.8-4                glue_1.3.0              
##  [29] gtable_0.2.0             zlibbioc_1.29.0         
##  [31] XVector_0.23.0           kernlab_0.9-27          
##  [33] car_3.0-2                Rhdf5lib_1.5.0          
##  [35] prabclus_2.2-6           DEoptimR_1.0-8          
##  [37] HDF5Array_1.11.0         abind_1.4-5             
##  [39] VIM_4.7.0                scales_1.0.0            
##  [41] sgeostat_1.0-27          mvtnorm_1.0-8           
##  [43] DBI_1.0.0                GGally_1.4.0            
##  [45] sROC_0.1-2               Rcpp_1.0.0              
##  [47] laeken_0.4.6             viridisLite_0.3.0       
##  [49] progress_1.2.0           foreign_0.8-71          
##  [51] bit_1.1-14               mclust_5.4.1            
##  [53] truncnorm_1.0-8          vcd_1.4-4               
##  [55] httr_1.3.1               fpc_2.1-11.1            
##  [57] RColorBrewer_1.1-2       modeltools_0.2-22       
##  [59] NADA_1.6-1               flexmix_2.3-14          
##  [61] pkgconfig_2.0.2          reshape_0.8.8           
##  [63] XML_3.98-1.16            nnet_7.3-12             
##  [65] locfit_1.5-9.1           tidyselect_0.2.5        
##  [67] labeling_0.3             rlang_0.3.0.1           
##  [69] reshape2_1.4.3           cellranger_1.1.0        
##  [71] munsell_0.5.0            tools_3.6.0             
##  [73] RSQLite_2.1.1            pls_2.7-0               
##  [75] cvTools_0.3.2            evaluate_0.12           
##  [77] stringr_1.3.1            yaml_2.2.0              
##  [79] bit64_0.9-7              zip_1.0.0               
##  [81] robustbase_0.93-3        purrr_0.2.5             
##  [83] biomaRt_2.39.2           compiler_3.6.0          
##  [85] beeswarm_0.2.3           curl_3.2                
##  [87] e1071_1.7-0              zCompositions_1.1.2     
##  [89] tibble_1.4.2             statmod_1.4.30          
##  [91] robCompositions_2.0.9    pcaPP_1.9-73            
##  [93] stringi_1.2.4            highr_0.7               
##  [95] forcats_0.3.0            trimcluster_0.1-2.1     
##  [97] lattice_0.20-38          ProtGenerics_1.15.0     
##  [99] pillar_1.3.0             BiocManager_1.30.4      
## [101] lmtest_0.9-36            BiocNeighbors_1.1.1     
## [103] data.table_1.11.8        cowplot_0.9.3           
## [105] bitops_1.0-6             irlba_2.3.3             
## [107] rtracklayer_1.43.0       R6_2.3.0                
## [109] bookdown_0.7             KernSmooth_2.23-15      
## [111] gridExtra_2.3            rio_0.5.10              
## [113] vipor_0.4.5              boot_1.3-20             
## [115] MASS_7.3-51.1            assertthat_0.2.0        
## [117] rhdf5_2.27.1             rprojroot_1.3-2         
## [119] withr_2.1.2              GenomicAlignments_1.19.0
## [121] Rsamtools_1.35.0         GenomeInfoDbData_1.2.0  
## [123] diptest_0.75-7           hms_0.4.2               
## [125] grid_3.6.0               class_7.3-14            
## [127] rmarkdown_1.10           DelayedMatrixStats_1.5.0
## [129] carData_3.0-2            mvoutlier_2.0.9         
## [131] Rtsne_0.15               ggbeeswarm_0.6.0
```

# References
