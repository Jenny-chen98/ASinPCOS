# ASinPCOS
The scripts and resources for the article `The landscape of alternative splicing in granulosa cells and a potential novel role of YAP1 in PCOS`

`yml` folder contains the packaged conda environment.

`Upstream Analysis of ASinPCOS.md` contains scripts and bash commands used for upstream analysis, including download raw data, quality control, mapping reads to human genome, reads quantification and differential alternative splicing events analysis.

`Downstream-asInPCOS.qmd` contains R scripts with `Quarto` format for all the downstream analysis of  RNA-seq data and plotting code and can be opened in `Rstudio`.

`results/03_isoform/switchPlot` folder contains all the plots analyzed by `IsoformSwitchAnalyzeR` package. A specific gene containing an isoform switch by creating a composite plot visualizing: 1) The isoform structure along with the concatenated annotations (including transcript classification, ORF, Coding poteintial, NMD sensitivity, annotated protein domains); 2) gene and isoform expression and 3) isoform usage-including the result of the isoform switch test.

`results/01_DEG` folder contains the GO enrichment tables derived from the clustering in `Figure 1C` and `Figure S2`.
