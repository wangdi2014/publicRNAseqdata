Walkthrough with all data analyses that comes with the manuscript
Comparing published FPKM/RPKM values and reprocessed FPKM values.
========================================================

Prepare by loading packages etc.

```r
library(pheatmap)
library(reshape)
library(gplots)
```

```
## KernSmooth 2.23 loaded
## Copyright M. P. Wand 1997-2009
## 
## Attaching package: 'gplots'
## 
## The following object is masked from 'package:stats':
## 
##     lowess
```

```r
library(ops)
```

```
## 
## Attaching package: 'ops'
## 
## The following object is masked from 'package:stats':
## 
##     filter
```

```r
library(calibrate)
```

```
## Loading required package: MASS
```

```r
library(biomaRt)
library(sva)
```

```
## Loading required package: corpcor
## Loading required package: mgcv
## Loading required package: nlme
## This is mgcv 1.8-3. For overview type 'help("mgcv-package")'.
```

Data from four different public sources are downloaded and brain, heart and kidney samples are extracted. 

Start with Human Protein Atlas (HPA):


```r
#temp <- tempfile()
#download.file(url="http://www.proteinatlas.org/download/rna.csv.zip",destfile=temp)
#hpa <- read.csv(unz(temp, "rna.csv"))
#unlink(temp)

#hpa.heart <- hpa[hpa$Sample=="heart muscle", c("Gene", "Value")]
#hpa.brain <- hpa[hpa$Sample=="cerebral cortex", c("Gene", "Value")]
#hpa.kidney <- hpa[hpa$Sample=="kidney", c("Gene", "Value")]

#hpa.fpkms <- merge(hpa.heart, hpa.brain, by="Gene")
#hpa.fpkms <- merge(hpa.fpkms, hpa.kidney, by="Gene")
#colnames(hpa.fpkms) <- c("ENSG_ID", "HPA_heart", "HPA_brain", "HPA_kidney")
```

Check if the identifiers are unique and write table to file.


```r
#length(hpa.fpkms[,1])
#length(unique(hpa.fpkms[,1]))

#write.table(hpa.fpkms,file="hpa_fpkms.txt",quote=F,sep="\t")
```

Next dataset is from the article "Alternative isoform regulation in human tissue transcriptomes.." by Wang et.al
AltIso:


```r
#temp <- tempfile()
#download.file(url="http://genes.mit.edu/burgelab/Supplementary/wang_sandberg08/hg18.ensGene.CEs.rpkm.txt",destfile=temp)
#altiso <- read.delim(temp, sep="\t")
#unlink(temp)
```

There is no kidney sample here, so just use heart and brain


```r
#altiso.fpkms <- altiso[,c("X.Gene","heart","brain")]
#colnames(altiso.fpkms) <- c("ENSG_ID", "AltIso_heart", "AltIso_brain")
```

Check uniqueness of IDs.


```r
#length(altiso.fpkms[,1])
#length(unique(altiso.fpkms[,1]))

#write.table(altiso.fpkms,file="altiso_fpkms.txt",quote=F,sep="\t")
```

Next dataset is derived from "GTEx": Genotype-Tissue Expression

This is a big download: 337.8 Mb (as of 2014-02-04)
We also add some code to randomly select one sample from each tissue type; there are many biological replicates in this data set.


```r
#temp <- tempfile()
#download.file(url="http://www.gtexportal.org/home/rest/file/download?portalFileId=175729&forDownload=true",destfile=temp)
#header_lines <- readLines(temp, n=2)
#gtex <- read.delim(temp, skip=2, sep="\t")
#unlink(temp)

#write.table(gtex, file="gtex_all.txt",   quote=F, sep="\t")

#download.file(url="http://www.gtexportal.org/home/rest/file/download?portalFileId=175707&forDownload=true",destfile="GTEx_description.txt")

#metadata <- read.delim("GTEx_description.txt", sep="\t")
```

The metadata table seems to contain entries that are not in the RPKM table.


```r
#samp.id <- gsub('-','.',metadata$SAMPID)
#eligible.samples <- which(samp.id %in% colnames(gtex))
#metadata <- metadata[eligible.samples,]
```

Select random heart, kidney and brain samples.


```r
#random.heart <- sample(which(metadata$SMTS=="Heart"), size=1)
#random.heart.samplename <- gsub('-','.',metadata[random.heart, "SAMPID"])
#gtex.heart.fpkm <- as.numeric(gtex[,random.heart.samplename])

#random.brain <- sample(which(metadata$SMTS=="Brain"), size=1)
#random.brain.samplename <- gsub('-','.',metadata[random.brain, "SAMPID"])
#gtex.brain.fpkm <- as.numeric(gtex[,random.brain.samplename])

#random.kidney <- sample(which(metadata$SMTS=="Kidney"), size=1)
#random.kidney.samplename <- gsub('-','.',metadata[random.kidney, "SAMPID"])
#gtex.kidney.fpkm <- as.numeric(gtex[,random.kidney.samplename])
```

Get gene IDs on same format as the other data sets by removing the part after the dot; check ID uniqueness and write to file.


```r
#gtex.names <- gtex[,"Name"]
#temp_list <- strsplit(as.character(gtex.names), split="\\.")
#gtex.names.nodot <- unlist(temp_list)[2*(1:length(gtex.names))-1]

#gtex.fpkms <- data.frame(ENSG_ID=gtex.names.nodot, GTEx_heart=gtex.heart.fpkm, GTEx_brain=gtex.brain.fpkm,GTEx_kidney=gtex.kidney.fpkm)

#length(gtex.fpkms[,1])
#length(unique(gtex.fpkms[,1]))

#write.table(gtex.fpkms,file="gtex_fpkms.txt",quote=F,sep="\t")
```

*RNA-seq Atlas*


```r
#temp <- tempfile()
#download.file(url="http://medicalgenomics.org/rna_seq_atlas/download?download_revision1=1",destfile=temp)
#atlas <- read.delim(temp, sep="\t")
#unlink(temp)

#atlas.fpkms <- atlas[,c("ensembl_gene_id","heart","hypothalamus","kidney")]
#colnames(atlas.fpkms) <- c("ENSG_ID","Atlas_heart","Atlas_brain","Atlas_kidney")
#write.table(atlas.fpkms,file="atlas_fpkms.txt",quote=F,sep="\t")
```

Combining F/RPKM values from public data sets
---------------------------------------------

We will join the data sets on ENSEMBL ID:s, losing a lot of data in the process - but joining on gene symbols or something else would lead to an even worse loss. 


```r
library(org.Hs.eg.db) # for transferring gene identifiers
```

```
## Loading required package: AnnotationDbi
## Loading required package: BiocGenerics
## Loading required package: parallel
## 
## Attaching package: 'BiocGenerics'
## 
## The following objects are masked from 'package:parallel':
## 
##     clusterApply, clusterApplyLB, clusterCall, clusterEvalQ,
##     clusterExport, clusterMap, parApply, parCapply, parLapply,
##     parLapplyLB, parRapply, parSapply, parSapplyLB
## 
## The following object is masked from 'package:stats':
## 
##     xtabs
## 
## The following objects are masked from 'package:base':
## 
##     anyDuplicated, append, as.data.frame, as.vector, cbind,
##     colnames, do.call, duplicated, eval, evalq, Filter, Find, get,
##     intersect, is.unsorted, lapply, Map, mapply, match, mget,
##     order, paste, pmax, pmax.int, pmin, pmin.int, Position, rank,
##     rbind, Reduce, rep.int, rownames, sapply, setdiff, sort,
##     table, tapply, union, unique, unlist
## 
## Loading required package: Biobase
## Welcome to Bioconductor
## 
##     Vignettes contain introductory material; view with
##     'browseVignettes()'. To cite Bioconductor, see
##     'citation("Biobase")', and for packages 'citation("pkgname")'.
## 
## Loading required package: GenomeInfoDb
## 
## Attaching package: 'AnnotationDbi'
## 
## The following object is masked from 'package:MASS':
## 
##     select
## 
## Loading required package: DBI
```

```r
library(data.table) # for collapsing transcript RPKMs
library(pheatmap) # for nicer visualization
library(edgeR) # for TMM normalization
```

```
<<<<<<< HEAD
## Loading required package: limma
## 
## Attaching package: 'limma'
## 
## The following object is masked from 'package:BiocGenerics':
## 
##     plotMA
=======
## Error: there is no package called 'edgeR'
>>>>>>> a26a6006b42f51efac74e0a6bbc6f403cbf4a1b0
```

```r
#hpa.fpkms <- read.delim("hpa_fpkms.txt")
#altiso.fpkms <- read.delim("altiso_fpkms.txt")
#gtex.fpkms <- read.delim("gtex_fpkms.txt")
#atlas.fpkms <- read.delim("atlas_fpkms.txt")
```

The RNA-seq Atlas data set uses many different identifiers, while the other all use ENSG as the primary identifier

Approach 1: Merge on ENSEMBL genes (ENSG) as given in RNA-seq Atlas. Note that there are repeated ENSG ID:s in RNA-seq Atlas, as opposed to the other data sets, so we need to do something about that. In this case, we just sum the transcripts that belong to each ENSG gene. We use data.table for this.


```r
#data.dt <- data.table(atlas.fpkms)
#setkey(data.dt, ENSG_ID)
#temp <- data.dt[, lapply(.SD, sum), by=ENSG_ID]
#collapsed <- as.data.frame(temp)
#atlas.fpkms.summed <- collapsed[,2:ncol(collapsed)] 
#rownames(atlas.fpkms.summed) <- collapsed[,1]

#atlas.fpkms.summed <- atlas.fpkms.summed[2:nrow(atlas.fpkms.summed),]
```

Finally, combine all the data sets into a data frame.


```r
#fpkms <- merge(hpa.fpkms, altiso.fpkms, by="ENSG_ID")
#fpkms <- merge(fpkms, gtex.fpkms, by="ENSG_ID")
#fpkms <- merge(fpkms, atlas.fpkms.summed, by.x="ENSG_ID", by.y=0)
#gene_id <- fpkms[,1]
#f <- fpkms[,2:ncol(fpkms)]
#rownames(f) <- gene_id
```

Check how many ENSG IDs we have left.


```r
#dim(f)
```

Approach 2: Try to map Entrez symbols to ENSEMBL to recover more ENSG IDs than already present in the table. 


```r
#m <- org.Hs.egENSEMBL
#mapped_genes <- mappedkeys(m)
#ensg.for.entrez <- as.list(m[mapped_genes])
#remapped.ensg <- ensg.for.entrez[as.character(atlas$entrez_gene_id)]

#atlas.fpkms$remapped_ensg <- as.character(remapped.ensg)

# And add expression values
#data.dt <- data.table(atlas.fpkms[,2:ncol(atlas.fpkms)])
#setkey(data.dt, remapped_ensg)
#temp <- data.dt[, lapply(.SD, sum), by=remapped_ensg]
#collapsed <- as.data.frame(temp)
#atlas.fpkms.summed <- collapsed[,2:ncol(collapsed)] 
#rownames(atlas.fpkms.summed) <- collapsed[,1]
```

Combine data sets again


```r
#fpkms <- merge(hpa.fpkms, altiso.fpkms, by="ENSG_ID")
#fpkms <- merge(fpkms, gtex.fpkms, by="ENSG_ID")
#fpkms <- merge(fpkms, atlas.fpkms.summed, by.x="ENSG_ID", by.y=0)
#gene_id <- fpkms[,1]
#f <- fpkms[,2:ncol(fpkms)]
#rownames(f) <- gene_id
#write.table(f, file = 'published_rpkms.txt', quote=F)

#instead of downloading everytime:
```
Download the data from local file:


```r
published <- read.delim("published_rpkms.txt",sep=" ")

sampleinfo_published <- read.table("sample_info_published.txt",header=TRUE)
```

The published FPKM values are first filtered by removing all lines where FPKM is less or equal to 0.01 in all samples:


```r
published.nozero <- published[-which(rowSums(published[,])<=0.01),]
```

Heatmap of Spearman correlations between published expression profiles (# genes = 13,323):


```r
pheatmap(cor(published.nozero, method="spearman")) 
```

![plot of chunk :published heatmap spearman](figure/:published heatmap spearman.png) 

Alternatively, one could use Pearson correlation (not shown in paper):


```r
pheatmap(cor(published.nozero))
```

![plot of chunk :published heatmap pearson](figure/:published heatmap pearson.png) 

PCA analysis of published FPKM values


```r
colors <- c("indianred", "dodgerblue", "forestgreen",
            "indianred", "dodgerblue",
            "indianred", "dodgerblue", "forestgreen", 
            "indianred", "dodgerblue", "forestgreen")
  
p <- prcomp(t(published.nozero))
rownames(p$x) <- c(rep("HPA",3),rep("AltIso",2),rep("GTEx",3),rep("Atlas",3))

plot(p$x[,1],p$x[,2],pch=20,cex=1.5,col=colors,xlab=paste("PC1 58% of variance"),ylab=paste("PC2 13% of variance"),main="Published FPKM/RPKM values \n n=13,323")
legend("bottomright",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
```

![plot of chunk :published PCA](figure/:published PCA1.png) 

```r
#plotting PC2 vs PC3 (not shown in paper):
plot(p$x[,2],p$x[,3],pch=20,cex=1.5,col=colors,xlab=paste("PC2"),ylab=paste("PC3"),main="Published FPKM/RPKM values \n n=13,323")
legend("bottomright",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
```

![plot of chunk :published PCA](figure/:published PCA2.png) 

We can plot all pairwise combinations of principal components 1 to 5. (not shown in paper):


```r
par(mfrow=c(4,4))
for (i in 1:6){
  for(j in 1:6){
  	if (i<j){ 
		plot(p$x[,i],p$x[,j],pch=20,col=colors,xlab=paste("PC",i),ylab=paste("PC",j),main="Published FPKM values \n n=13,323")
		}
	}
}
```

![plot of chunk :published pairwise PCA](figure/:published pairwise PCA.png) 

Look a bit closer at PCs 1-3 in prcomp:


```r
      load.pc1 <- p$rotation[,1][order(p$rotation[,1])]
      extreme.pc1 <- c(tail(load.pc1), head(load.pc1))

      extreme.pc1.ensg <- names(extreme.pc1)
      ensembl = useMart("ensembl", dataset = "hsapiens_gene_ensembl") #select the ensembl database
      extreme.pc1.symbols <- getBM(attributes=c("ensembl_gene_id", "hgnc_symbol"), 
                           filters = "ensembl_gene_id",
                           values=extreme.pc1.ensg,
                           mart=ensembl)
      
      barplot(extreme.pc1, names.arg=extreme.pc1.symbols[,2], las=2, main="Genes w highest absolute loadings in PC1 (raw RPKM)")
```

![plot of chunk :published PC 1-3](figure/:published PC 1-31.png) 

```r
      load.pc2 <- p$rotation[,2][order(p$rotation[,2])]
      extreme.pc2 <- c(tail(load.pc2), head(load.pc2))
      
      extreme.pc2.ensg <- names(extreme.pc2)
      extreme.pc2.symbols <- getBM(attributes=c("ensembl_gene_id", "hgnc_symbol"), 
                           filters = "ensembl_gene_id",
                           values=extreme.pc2.ensg,
                           mart=ensembl)
      
      barplot(extreme.pc2, names.arg=extreme.pc2.symbols[,2], las=2, main="Genes w highest absolute loadings in PC2 (raw RPKM)")
```

![plot of chunk :published PC 1-3](figure/:published PC 1-32.png) 

```r
      load.pc3 <- p$rotation[,3][order(p$rotation[,3])]
      extreme.pc3 <- c(tail(load.pc3), head(load.pc3))
      
      extreme.pc3.ensg <- names(extreme.pc3)
      extreme.pc3.symbols <- getBM(attributes=c("ensembl_gene_id", "hgnc_symbol"), 
                           filters = "ensembl_gene_id",
                           values=extreme.pc3.ensg,
                           mart=ensembl)
      
      
      barplot(extreme.pc3, names.arg=extreme.pc3.symbols[,2], las=2, main="Genes w highest absolute loadings in PC3 (raw RPKM)")
```

![plot of chunk :published PC 1-3](figure/:published PC 1-33.png) 

PC1:
ensembl_gene_id hgnc_symbol HPA_heart HPA_brain HPA_kidney AltIso_heart AltIso_brain   GTEx_heart   GTEx_brain  GTEx_kidney Atlas_heart Atlas_brain   Atlas_kidney
ENSG00000087086         FTL     436.5     487.2     1523.1       463.86       407.35 5.737621e+02 2099.7219238 5310.5092773      16.930       7.700     40.892
ENSG00000092054        MYH7    1802.4       2.1        0.8      4117.17         0.83 4.713973e+03    6.4869833    0.6886783    2137.510       0.635     0.354
ENSG00000111245        MYL2    5291.5       0.2        2.6     16780.36         1.03 1.576518e+04   10.2006025    2.3142705    1064.500       0.000     0.358
ENSG00000118194       TNNT2    2693.8       7.0       20.8      8074.39         9.37 2.027960e+03    1.6754625   11.8667431   10930.800       8.713     43.633
ENSG00000118785        SPP1       8.3     368.2     1251.1         7.74       309.26 5.712032e-01  317.7869873 2806.6496582      16.293     121.874     878.535
ENSG00000120885         CLU     336.9    3545.9      631.6        58.72      1682.03 2.138420e+01  307.3468933  365.1710815      29.357     394.746     47.363
ENSG00000123560        PLP1      11.9    1295.2        0.7         2.56      1544.93 2.431595e+00 1164.7164307    0.4162828       5.086     779.755     0.513
ENSG00000131095        GFAP       3.0    1372.0        1.4         1.70       834.13 1.671327e+00 3633.8173828    0.8537604       0.549     691.908     0.135
ENSG00000136872       ALDOB       0.4       2.4     1824.1         0.00         0.49 1.629837e-01    0.2298469 1927.8311768       1.452       0.125     788.910
ENSG00000159251       ACTC1    2914.4       1.6        0.4      3457.43         0.42 5.319639e+03    3.6530383    1.0412433     310.068       0.421     0.311
ENSG00000175084         DES    3403.6       2.4       10.2      3160.05         2.65 5.524179e+03   13.1029377    4.9041762    1675.890       4.500     6.252
ENSG00000198125          MB    3937.3       0.9        2.3      7065.54         1.35 2.705914e+03    2.6915243    0.6978353    1415.554       0.032     0.000
                

PC2:
ensembl_gene_id   hgnc_symbol HPA_heart HPA_brain HPA_kidney AltIso_heart AltIso_brain   GTEx_heart   GTEx_brain  GTEx_kidney Atlas_heart Atlas_brain   Atlas_kidney
ENSG00000075624        ACTB     423.6     786.5      521.8       377.87      3277.02   283.548279  782.8235474  653.1090088      67.058     140.560     69.191
ENSG00000087086         FTL     436.5     487.2     1523.1       463.86       407.35   573.762085 2099.7219238 5310.5092773      16.930       7.700     40.892
ENSG00000106631        MYL7    4606.5       0.2        0.1       896.01         0.00   214.277832    1.5850240    0.5706197     115.287       0.000     0.000
ENSG00000111245        MYL2    5291.5       0.2        2.6     16780.36         1.03 15765.178711   10.2006025    2.3142705    1064.500       0.000     0.358
ENSG00000118194       TNNT2    2693.8       7.0       20.8      8074.39         9.37  2027.959717    1.6754625   11.8667431   10930.800       8.713     43.633
ENSG00000125971     DYNLRB1     207.8     164.7      170.0      1532.98      2293.04    21.458061   42.5225830   32.5898590      20.874      42.315     23.413
ENSG00000140416        TPM1    4937.5      61.9      128.6      4379.09        68.39  1132.400513   17.7401543   31.1860542    4228.464     105.073     189.505
ENSG00000148677      ANKRD1    2451.5       0.2        0.3      1088.94         0.10  1471.471680    2.0739977    0.5601565    3863.610       0.403     0.263
ENSG00000160808        MYL3    1213.4       3.2       24.9      3909.81         0.84  3996.971191    5.8288269   19.8492756      80.668       0.204     2.634
ENSG00000167996        FTH1     363.8     366.9      498.9      3563.32      9086.14   337.275970 1151.6777344 1828.0887451      43.503      53.459     66.455
ENSG00000175206        NPPA    6693.0       8.1        0.1       193.69         1.74   137.694275    0.8982058    2.7191663     311.399       0.445     0.131
ENSG00000188257     PLA2G2A      51.2       0.8        2.3       210.68         0.00     2.313501    0.4622447    0.4419056    2351.218       3.057     24.728
                
       
PC3:
ensembl_gene_id hgnc_symbol HPA_heart HPA_brain HPA_kidney AltIso_heart AltIso_brain  GTEx_heart   GTEx_brain  GTEx_kidney Atlas_heart Atlas_brain    Atlas_kidney
ENSG00000071082       RPL31     632.0     348.9      561.3      2239.86      1678.90    38.41791   99.8054581   51.5868454     378.917     484.379    259.057
ENSG00000075624        ACTB     423.6     786.5      521.8       377.87      3277.02   283.54828  782.8235474  653.1090088      67.058     140.560    69.191
ENSG00000101608      MYL12A    2033.5      32.2      214.9       528.97        28.70  1968.66089   54.9993858  141.2810822     300.410      15.476    24.862
ENSG00000106211       HSPB1     521.6      37.7      196.6      3971.37       643.46   333.51608   86.1186981  201.8283081      61.190       6.808    17.175
ENSG00000111245        MYL2    5291.5       0.2        2.6     16780.36         1.03 15765.17871   10.2006025    2.3142705    1064.500       0.000    0.358
ENSG00000118194       TNNT2    2693.8       7.0       20.8      8074.39         9.37  2027.95972    1.6754625   11.8667431   10930.800       8.713    43.633
ENSG00000125971     DYNLRB1     207.8     164.7      170.0      1532.98      2293.04    21.45806   42.5225830   32.5898590      20.874      42.315    23.413
ENSG00000129991       TNNI3    2157.5       0.5        0.1      2698.36         0.47  5751.82275    4.0958185    0.6090932     999.559       0.675    0.132
ENSG00000159251       ACTC1    2914.4       1.6        0.4      3457.43         0.42  5319.63867    3.6530383    1.0412433     310.068       0.421    0.311
ENSG00000167996        FTH1     363.8     366.9      498.9      3563.32      9086.14   337.27597 1151.6777344 1828.0887451      43.503      53.459    66.455
ENSG00000175084         DES    3403.6       2.4       10.2      3160.05         2.65  5524.17871   13.1029377    4.9041762    1675.890       4.500    6.252
ENSG00000175206        NPPA    6693.0       8.1        0.1       193.69         1.74   137.69427    0.8982058    2.7191663     311.399       0.445    0.131
                

Try Anova on a "melted" expression matrix with some metadata:


```r
m <- melt(published.nozero)
```

```
## Using  as id variables
```

```r
colnames(m) <- c("sample_ID","RPKM")

meta <- sampleinfo_published[,c("Study","Tissue","Preparation","NumberRaw","Numbermapped","Readtype")]
rownames(meta) <- colnames(published.nozero)
tissue <- rep(meta$Tissue, each=nrow(published.nozero))
study <- rep(meta$Study, each=nrow(published.nozero))
prep <- rep(meta$Preparation, each=nrow(published.nozero))
layout <- rep(meta$Readtype, each=nrow(published.nozero))
raw <- rep(meta$NumberRaw, each=nrow(published.nozero))
mapped <- rep(meta$Numbermapped, each=nrow(published.nozero))
data <- data.frame(m, tissue=tissue, study=study, prep=prep, layout=layout,nraw=raw,nmapped=mapped)
fit <- lm(RPKM ~ layout + prep + nraw + study + tissue, data=data)
a <- anova(fit)
maxval = 100

barplot(100*a$"Sum Sq"[1:5]/sum(a$"Sum Sq"[1:5]),names.arg=rownames(a[1:5,]),main="Anova, published FPKM/RPKM values",ylim=c(0,maxval))
```

![plot of chunk :published anova](figure/:published anova.png) 

Try log2 tranform the published FPKM values:


```r
pseudo <- 1
published.log <- log2(published.nozero + pseudo)
```

Heatmap of Spearman correlations between published expression profiles with log2 values:


```r
pheatmap(cor(published.log),method="spearman")
```

![plot of chunk :published log heatmap spearman](figure/:published log heatmap spearman.png) 

PCA analysis of log2 published FPKM values:


```r
colors <- c("indianred", "dodgerblue", "forestgreen",
            "indianred", "dodgerblue",
            "indianred", "dodgerblue", "forestgreen", 
            "indianred", "dodgerblue", "forestgreen")

p.log <- prcomp(t(published.log))

plot(p.log$x[,1],p.log$x[,2],pch=20,col=colors,xlab=paste("PC1 31% of variance"),ylab=paste("PC2 27% of variance"),main="log2 Published FPKM/RPKM values \n n=13,323")
legend("bottomright",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
```

![plot of chunk :published log PCA](figure/:published log PCA1.png) 

```r
plot(p.log$x[,2],p.log$x[,3],pch=20,col=colors,xlab=paste("PC2 27% of variance"),ylab=paste("PC3 19% of variance"),main="log2 Published FPKM values \n n=13,323")
legend("bottomright",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
```

![plot of chunk :published log PCA](figure/:published log PCA2.png) 

We can plot all pairwise combinations of principal components 1 to 5. (not shown in paper):


```r
par(mfrow=c(4,4))
for (i in 1:6){
  for(j in 1:6){
    if (i<j){ 
		plot(p.log$x[,i],p.log$x[,j],pch=20,col=colors,xlab=paste("PC",i),ylab=paste("PC",j),main="log2 Published FPKM values \n n=13323")
		}
	}
}
```

![plot of chunk :published log PCA pairwise](figure/:published log PCA pairwise.png) 

Look a bit closer at PCs 1-3 in prcomp:


```r
     load.pc1 <- p.log$rotation[,1][order(p.log$rotation[,1])]
     extreme.pc1 <- c(tail(load.pc1), head(load.pc1))

     extreme.pc1.ensg <- names(extreme.pc1)
     extreme.pc1.symbols <- getBM(attributes=c("ensembl_gene_id", "hgnc_symbol"), 
                           filters = "ensembl_gene_id",
                           values=extreme.pc1.ensg,
                           mart=ensembl)
      
      barplot(extreme.pc1, names.arg=extreme.pc1.symbols[,2], las=2, main="Genes w highest absolute loadings in PC1 (log2RPKM)")
```

![plot of chunk :published log PC 1-3](figure/:published log PC 1-31.png) 

```r
      load.pc2 <- p.log$rotation[,2][order(p.log$rotation[,2])]
      extreme.pc2 <- c(tail(load.pc2), head(load.pc2))
      
      extreme.pc2.ensg <- names(extreme.pc2)
      extreme.pc2.symbols <- getBM(attributes=c("ensembl_gene_id", "hgnc_symbol"), 
                           filters = "ensembl_gene_id",
                           values=extreme.pc2.ensg,
                           mart=ensembl)
     
      barplot(extreme.pc2, names.arg=extreme.pc2.symbols[,2], las=2, main="Genes w highest absolute loadings in PC2 (log2RPKM)")
```

![plot of chunk :published log PC 1-3](figure/:published log PC 1-32.png) 

```r
      load.pc3 <- p.log$rotation[,3][order(p$rotation[,3])]
      extreme.pc3 <- c(tail(load.pc3), head(load.pc3))
      
      extreme.pc3.ensg <- names(extreme.pc3)
      extreme.pc3.symbols <- getBM(attributes=c("ensembl_gene_id", "hgnc_symbol"), 
                           filters = "ensembl_gene_id",
                           values=extreme.pc3.ensg,
                           mart=ensembl)
      
      barplot(extreme.pc3, names.arg=extreme.pc3.symbols[,2], las=2, main="Genes w highest absolute loadings in PC3 (log2RPKM)")
```

![plot of chunk :published log PC 1-3](figure/:published log PC 1-33.png) 

PC1:
ensembl_gene_id hgnc_symbol HPA_heart HPA_brain HPA_kidney AltIso_heart AltIso_brain   GTEx_heart   GTEx_brain  GTEx_kidney Atlas_heart Atlas_brain    Atlas_kidney
ENSG00000057593          F7       0.2       0.3        0.3         0.00         0.30 6.088877e-03    0.1806296    0.4762675      26.652      15.712    19.155
ENSG00000063177       RPL18     455.9     237.8      458.0       205.67       284.33 1.003603e+02  159.0169830  134.4763184       3.984       5.143    2.603
ENSG00000087086         FTL     436.5     487.2     1523.1       463.86       407.35 5.737621e+02 2099.7219238 5310.5092773      16.930       7.700    40.892
ENSG00000100097      LGALS1     498.7     113.2       77.2       389.82       161.11 1.837938e+02  216.8468323   58.2166481       2.138       1.154    0.332
ENSG00000105372       RPS19     419.2     268.3      453.6       118.30       140.77 1.180695e+02  324.9812317  306.6752319       3.360       2.945    3.123
ENSG00000105583     WDR83OS     132.7      52.6       99.0        23.17        26.35 6.529290e+01   62.2843895   80.6908875       0.411       0.352    0.452
ENSG00000105640      RPL18A      45.8      31.3       68.1       505.42       422.37 4.883201e+01   55.6919632   60.6585007       0.000       0.000    0.000  
ENSG00000118194       TNNT2    2693.8       7.0       20.8      8074.39         9.37 2.027960e+03    1.6754625   11.8667431   10930.800       8.713    43.633
ENSG00000135218        CD36     499.4       2.6        8.0       132.46         0.08 1.551205e+02    0.5913891    0.3047371     894.983       8.763    17.456
ENSG00000148677      ANKRD1    2451.5       0.2        0.3      1088.94         0.10 1.471472e+03    2.0739977    0.5601565    3863.610       0.403    0.263
ENSG00000171560         FGA       0.0       0.0        0.7         0.00         0.00 0.000000e+00    0.0000000    6.4727707      63.271       0.000    25.045
ENSG00000188257     PLA2G2A      51.2       0.8        2.3       210.68         0.00 2.313501e+00    0.4622447    0.4419056    2351.218       3.057    24.728
                 
PC2:
ensembl_gene_id hgnc_symbol HPA_heart HPA_brain HPA_kidney AltIso_heart AltIso_brain   GTEx_heart  GTEx_brain GTEx_kidney Atlas_heart Atlas_brain   Atlas_kidney
ENSG00000104435       STMN2       0.1     235.9        0.2         0.00       176.88 1.790114e-02  217.035278  0.15557937       0.105     139.464   0.000
ENSG00000104833      TUBB4A       0.3     235.1        1.5         2.94      2038.44 5.334400e-01  387.557159  1.91307497       0.408      82.655   0.389
ENSG00000104879         CKM    1172.0       0.1        2.6      3321.37         0.26 2.774733e+03    4.498085  1.39116168      57.889       0.000   0.060
ENSG00000111245        MYL2    5291.5       0.2        2.6     16780.36         1.03 1.576518e+04   10.200603  2.31427050    1064.500       0.000   0.358
ENSG00000114854       TNNC1    3087.9       0.3       14.0      4100.03         0.69 2.618310e+03    3.013112  5.23761368     316.135       0.763   1.014
ENSG00000123560        PLP1      11.9    1295.2        0.7         2.56      1544.93 2.431595e+00 1164.716431  0.41628277       5.086     779.755   0.513
ENSG00000131095        GFAP       3.0    1372.0        1.4         1.70       834.13 1.671327e+00 3633.817383  0.85376036       0.549     691.908   0.135
ENSG00000132639      SNAP25       1.2     802.9        1.7         0.15       535.77 3.757800e-02  117.302628  0.65681189       0.346     284.680   0.180
ENSG00000159251       ACTC1    2914.4       1.6        0.4      3457.43         0.42 5.319639e+03    3.653038  1.04124331     310.068       0.421   0.311
ENSG00000160808        MYL3    1213.4       3.2       24.9      3909.81         0.84 3.996971e+03    5.828827 19.84927559      80.668       0.204   2.634
ENSG00000168314        MOBP       0.0     138.0        0.1         0.00       100.74 6.864151e-03  486.668854  0.02783972       0.176     153.127   0.139
ENSG00000198125          MB    3937.3       0.9        2.3      7065.54         1.35 2.705914e+03    2.691524  0.69783533    1415.554       0.032   0.000

       
PC3:
ensembl_gene_id hgnc_symbol HPA_heart HPA_brain HPA_kidney AltIso_heart AltIso_brain  GTEx_heart   GTEx_brain  GTEx_kidney Atlas_heart Atlas_brain    Atlas_kidney
ENSG00000071082       RPL31     632.0     348.9      561.3      2239.86      1678.90    38.41791   99.8054581   51.5868454     378.917     484.379    259.057
ENSG00000075624        ACTB     423.6     786.5      521.8       377.87      3277.02   283.54828  782.8235474  653.1090088      67.058     140.560    69.191
ENSG00000101608      MYL12A    2033.5      32.2      214.9       528.97        28.70  1968.66089   54.9993858  141.2810822     300.410      15.476    24.862
ENSG00000106211       HSPB1     521.6      37.7      196.6      3971.37       643.46   333.51608   86.1186981  201.8283081      61.190       6.808    17.175
ENSG00000111245        MYL2    5291.5       0.2        2.6     16780.36         1.03 15765.17871   10.2006025    2.3142705    1064.500       0.000    0.358
ENSG00000118194       TNNT2    2693.8       7.0       20.8      8074.39         9.37  2027.95972    1.6754625   11.8667431   10930.800       8.713    43.633
ENSG00000125971     DYNLRB1     207.8     164.7      170.0      1532.98      2293.04    21.45806   42.5225830   32.5898590      20.874      42.315    23.413
ENSG00000129991       TNNI3    2157.5       0.5        0.1      2698.36         0.47  5751.82275    4.0958185    0.6090932     999.559       0.675    0.132
ENSG00000159251       ACTC1    2914.4       1.6        0.4      3457.43         0.42  5319.63867    3.6530383    1.0412433     310.068       0.421    0.311
ENSG00000167996        FTH1     363.8     366.9      498.9      3563.32      9086.14   337.27597 1151.6777344 1828.0887451      43.503      53.459    66.455
ENSG00000175084         DES    3403.6       2.4       10.2      3160.05         2.65  5524.17871   13.1029377    4.9041762    1675.890       4.500    6.252
ENSG00000175206        NPPA    6693.0       8.1        0.1       193.69         1.74   137.69427    0.8982058    2.7191663     311.399       0.445    0.131


To further validate the above results, indicating that tissue specificity appears mainly in PC 2 and 3, we will extract the 500 genes with highest loadings in each component and plot the corresponding published FPKM values in a heatmap:


```r
     load.pc1 <- abs(p.log$rotation[,1])[order(abs(p.log$rotation[,1]),decreasing=TRUE)]
     top.pc1 <- names(load.pc1[1:500]) 

     load.pc2 <- abs(p.log$rotation[,2])[order(abs(p.log$rotation[,2]),decreasing=TRUE)]
     top.pc2 <- names(load.pc2[1:500])

     load.pc3 <- abs(p.log$rotation[,3])[order(abs(p.log$rotation[,3]),decreasing=TRUE)]
     top.pc3 <- names(load.pc3[1:500])

     pheatmap(cor(published[top.pc1,]),method="spearman")
```

![plot of chunk :published log top 500 loadings heatmap](figure/:published log top 500 loadings heatmap1.png) 

```r
     pheatmap(cor(published[top.pc2,]),method="spearman")
```

![plot of chunk :published log top 500 loadings heatmap](figure/:published log top 500 loadings heatmap2.png) 

```r
     pheatmap(cor(published[top.pc3,]),method="spearman")
```

![plot of chunk :published log top 500 loadings heatmap](figure/:published log top 500 loadings heatmap3.png) 

Try Anova on a "melted" expression matrix with logged values and some metadata:


```r
n <- melt(published.log)
```

```
## Using  as id variables
```

```r
colnames(n) <- c("sample_ID","RPKM")
meta <- sampleinfo_published[,c("Study","Tissue","Preparation","NumberRaw","Numbermapped","Readtype")]
rownames(meta) <- colnames(published.log)
tissue <- rep(meta$Tissue, each=nrow(published.log))
study <- rep(meta$Study, each=nrow(published.log))
prep <- rep(meta$Preparation, each=nrow(published.log))
layout <- rep(meta$Readtype, each=nrow(published.log))
raw <- rep(meta$NumberRaw, each=nrow(published.log))
mapped <- rep(meta$Numbermapped, each=nrow(published.log))
data <- data.frame(n, tissue=tissue, study=study, prep=prep, layout=layout,nraw=raw,nmapped=mapped)
fit <- lm(RPKM ~ layout + prep + nraw + study + tissue, data=data)
b <- anova(fit)
maxval = 100

barplot(100*b$"Sum Sq"[1:5]/sum(b$"Sum Sq"[1:5]),names.arg=rownames(a[1:5,]),main="Anova, log2 published FPKM/RPKM values",ylim=c(0,maxval))
```

![plot of chunk :published log anova](figure/:published log anova.png) 

Combat analysis is performed on log2 values (n=13,323):


```r
meta <- data.frame(study=c(rep("HPA",3),rep("AltIso",2),rep("GTex",3),rep("Atlas",3)),tissue=c("Heart","Brain","Kidney","Heart","Brain","Heart","Brain","Kidney","Heart","Brain","Kidney"))

batch <- meta$study
design <- model.matrix(~as.factor(tissue),data=meta)

combat <- ComBat(dat=published.log,batch=batch,mod=design,numCovs=NULL,par.prior=TRUE)
```

```
## Found 4 batches
## Found 2  categorical covariate(s)
## Standardizing Data across genes
## Fitting L/S model and finding priors
## Finding parametric adjustments
## Adjusting the Data
```

```r
write.table(combat, file="published_rpkms_combat_log2.txt", quote=F)
```

Heatmap of Spearman correlations between published expression profiles after combat run (# genes = 13,323):


```r
pheatmap(cor(combat, method="spearman")) 
```

![plot of chunk :published log combat heatmap spearman](figure/:published log combat heatmap spearman.png) 

Alternatively, one could use Pearson correlation (not shown in paper):


```r
pheatmap(cor(combat))
```

![plot of chunk :published log combat heatmap pearson](figure/:published log combat heatmap pearson.png) 

PCA analysis of published FPKM values after combat run:


```r
colors <- c("indianred", "dodgerblue", "forestgreen",
            "indianred", "dodgerblue",
            "indianred", "dodgerblue", "forestgreen", 
            "indianred", "dodgerblue", "forestgreen")

p.combat <- prcomp(t(combat))

plot(p.combat$x[,1],p.combat$x[,2],pch=20,col=colors,xlab=paste("PC1 54% of variance"),ylab=paste("PC2 38% of variance"),main="Published FPKM values \n COMBAT \n n=13,323")
legend("bottomright",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
```

![plot of chunk :published log combat PCA](figure/:published log combat PCA1.png) 

```r
plot(p.combat$x[,2],p.combat$x[,3],pch=20,col=colors,xlab=paste("PC2"),ylab=paste("PC3"),main="Published FPKM values \n COMBAT \n n=13323")
```

![plot of chunk :published log combat PCA](figure/:published log combat PCA2.png) 

We can plot all pairwise combinations of principal components 1 to 5. (not shown in paper):


```r
par(mfrow=c(4,4))
for (i in 1:6){
  for(j in 1:6){
    if (i<j){ 
		plot(p.combat$x[,i],p.combat$x[,j],pch=20,col=colors,xlab=paste("PC",i),ylab=paste("PC",j),main="Published FPKM values \n COMBAT \ n=13323")
		}
	}
}
```

![plot of chunk :published log combat PCA pairwise](figure/:published log combat PCA pairwise.png) 

Look a bit closer at PCs 1-3 in prcomp:


```r
     load.pc1 <- p.combat$rotation[,1][order(p.combat$rotation[,1])]
     extreme.pc1 <- c(tail(load.pc1), head(load.pc1))

     extreme.pc1.ensg <- names(extreme.pc1)
     extreme.pc1.symbols <- getBM(attributes=c("ensembl_gene_id", "hgnc_symbol"), 
                           filters = "ensembl_gene_id",
                           values=extreme.pc1.ensg,
                           mart=ensembl)
      
      barplot(extreme.pc1, names.arg=extreme.pc1.symbols[,2], las=2, main="Genes w highest absolute loadings in PC1 (Combat log2RPKM)")
```

![plot of chunk :published log combat PC 1-3](figure/:published log combat PC 1-31.png) 

```r
      load.pc2 <- p.combat$rotation[,2][order(p.combat$rotation[,2])]
      extreme.pc2 <- c(tail(load.pc2), head(load.pc2))

      extreme.pc2.ensg <- names(extreme.pc2)
      extreme.pc2.symbols <- getBM(attributes=c("ensembl_gene_id", "hgnc_symbol"), 
                           filters = "ensembl_gene_id",
                           values=extreme.pc2.ensg,
                           mart=ensembl)
      
      barplot(extreme.pc2, names.arg=extreme.pc2.symbols[,2], las=2, main="Genes w highest absolute loadings in PC2 (Combat log2RPKM)")
```

![plot of chunk :published log combat PC 1-3](figure/:published log combat PC 1-32.png) 

```r
      load.pc3 <- p.combat$rotation[,3][order(p.combat$rotation[,3])]
      extreme.pc3 <- c(tail(load.pc3), head(load.pc3))

      extreme.pc3.ensg <- names(extreme.pc3)
      extreme.pc3.symbols <- getBM(attributes=c("ensembl_gene_id", "hgnc_symbol"), 
                           filters = "ensembl_gene_id",
                           values=extreme.pc3.ensg,
                           mart=ensembl)
      
      barplot(extreme.pc3, names.arg=extreme.pc3.symbols[,2], las=2, main="Genes w highest absolute loadings in PC3 (Combat log2RPKM)")
```

![plot of chunk :published log combat PC 1-3](figure/:published log combat PC 1-33.png) 

Revisit Anova with combated values.


```r
o <- melt(combat)
```

```
## Using  as id variables
```

```r
colnames(o) <- c("sample_ID","combat")
meta <- sampleinfo_published[,c("Study","Tissue","Preparation","NumberRaw","Numbermapped","Readtype")]
rownames(meta) <- colnames(combat)
tissue <- rep(meta$Tissue, each=nrow(combat))
study <- rep(meta$Study, each=nrow(combat))
prep <- rep(meta$Preparation, each=nrow(combat))
layout <- rep(meta$Readtype, each=nrow(combat))
raw <- rep(meta$NumberRaw, each=nrow(combat))
mapped <- rep(meta$Numbermapped, each=nrow(combat))
data <- data.frame(o, tissue=tissue, study=study, prep=prep, layout=layout,nraw=raw,nmapped=mapped)
fit <- lm(combat ~ layout + prep + nraw + study + tissue, data=data)
c <- anova(fit)
maxval = 100

barplot(100*c$"Sum Sq"[1:5]/sum(c$"Sum Sq"[1:5]),names.arg=rownames(c[1:5,]),main="Anova Combat",ylim=c(0,maxval))
```

![plot of chunk :published log combat anova](figure/:published log combat anova.png) 

Comparing FPKMs for FASTQ files reprocessed with TopHat and Cufflinks:


```r
cufflinks <- read.delim("fpkm_table_tophat.txt")

sampleinfo_cufflinks <- read.delim("sample_info_reprocessed.txt")
```

First, we will restrict the data set to only include protein coding genes using the ensembl based R package biomaRt:


```r
gene_ids <- as.vector(cufflinks[,1])

ensembl = useMart("ensembl", dataset = "hsapiens_gene_ensembl") #select the ensembl database

gene_type <- getBM(attributes=c("ensembl_gene_id", "gene_biotype"), 
                   filters = "ensembl_gene_id",
                   values=gene_ids,
                   mart=ensembl)

pc <- subset(gene_type[,1],gene_type[,2]=="protein_coding")

cufflinks_pc <- cufflinks[match(pc,cufflinks[,1]),]
```

Let's remove all lines where FPKM is close to zero in all samples before we proceed with this version of the data set:


```r
cufflinks_pc_nozero <- cufflinks_pc[-which(rowSums(cufflinks_pc[,3:16])<=0.01),]
```

Heatmap of Spearman correlations between reprocessed expression profiles (# genes = 19,475):


```r
pheatmap(cor(cufflinks_pc_nozero[,3:16], method="spearman")) 
```

![plot of chunk :cufflinks heatmap spearman](figure/:cufflinks heatmap spearman.png) 

Alternatively, one could use Pearson correlation (not shown in paper):


```r
pheatmap(cor(cufflinks_pc_nozero[,3:16])) 
```

![plot of chunk :cufflinks heatmap pearson](figure/:cufflinks heatmap pearson.png) 

Let's look at a few PCA plots:


```r
cufflinks_fpkms <- cufflinks_pc_nozero[,3:16]
rownames(cufflinks_fpkms) <- cufflinks_pc_nozero[,1]

p.cufflinks <- prcomp(t(cufflinks_fpkms))

colors <- c("dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred")          

p.cufflinks <- prcomp(t(cufflinks_fpkms))

colors <- c("dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred")          

plot(p.cufflinks$x[,1],p.cufflinks$x[,2],pch=20,col=colors,xlab=paste("PC1 87% of variance"),ylab=paste("PC2 7.7% of variance"),main="Reprocessed FPKM values \n n=19,475")
legend("bottomright",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
```

![plot of chunk :cufflinks PCA](figure/:cufflinks PCA1.png) 

```r
plot(p.cufflinks$x[,2],p.cufflinks$x[,2],pch=20,col=colors,xlab=paste("PC2"),ylab=paste("PC3"),main="Reprocessed FPKM values \n n=19475")
```

![plot of chunk :cufflinks PCA](figure/:cufflinks PCA2.png) 

We can plot all pairwise combinations of principal components 1 to 5 (not shown in paper):


```r
colors <- c("dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred")

par(mfrow=c(4,4))
for (i in 1:6){
  for(j in 1:6){
    if (i<j){ 
      plot(p.cufflinks$x[,i],p.cufflinks$x[,j],pch=20,col=colors,xlab=paste("PC",i),ylab=paste("PC",j),main="Cufflinks FPKM values \n n=19475")
		}
	}
}
```

![plot of chunk :cufflinks PCA pairwise](figure/:cufflinks PCA pairwise.png) 

Look at PCA loadings for PC1-3:


```r
      load.pc1 <- p.cufflinks$rotation[,1][order(p.cufflinks$rotation[,1])]
      extreme.pc1 <- c(tail(load.pc1), head(load.pc1))
      
      extreme.pc1.ensg <- names(extreme.pc1)
      extreme.pc1.symbols <- getBM(attributes=c("ensembl_gene_id", "hgnc_symbol"), 
                           filters = "ensembl_gene_id",
                           values=extreme.pc1.ensg,
                           mart=ensembl)
      
      barplot(extreme.pc1, names.arg=extreme.pc1.symbols[,2], las=2, main="Genes w highest absolute loadings in PC1 (raw Cufflinks    FPKM)")
```

![plot of chunk :cufflinks PC 1-3](figure/:cufflinks PC 1-31.png) 

```r
      load.pc2 <- p.cufflinks$rotation[,2][order(p.cufflinks$rotation[,2])]
      extreme.pc2 <- c(tail(load.pc2), head(load.pc2))
      
      extreme.pc2.ensg <- names(extreme.pc2)
      extreme.pc2.symbols <- getBM(attributes=c("ensembl_gene_id", "hgnc_symbol"), 
                           filters = "ensembl_gene_id",
                           values=extreme.pc2.ensg,
                           mart=ensembl)
      
      barplot(extreme.pc2, names.arg=extreme.pc2.symbols[,2], las=2, main="Genes w highest absolute loadings in PC2 (raw Cufflinks    FPKM)")
```

![plot of chunk :cufflinks PC 1-3](figure/:cufflinks PC 1-32.png) 

```r
      load.pc3 <- p.cufflinks$rotation[,3][order(p.cufflinks$rotation[,3])]
      extreme.pc3 <- c(tail(load.pc3), head(load.pc3))
      
      extreme.pc3.ensg <- names(extreme.pc3)
      extreme.pc3.symbols <- getBM(attributes=c("ensembl_gene_id", "hgnc_symbol"), 
                           filters = "ensembl_gene_id",
                           values=extreme.pc3.ensg,
                           mart=ensembl)
      
      barplot(extreme.pc3, names.arg=extreme.pc3.symbols[,2], las=2, main="Genes w highest absolute loadings in PC3 (raw Cufflinks    FPKM)")
```

![plot of chunk :cufflinks PC 1-3](figure/:cufflinks PC 1-33.png) 

PC1:
ensembl_gene_id     hgnc_symbol  EoGE_brain  EoGE_heart   EoGE_kidney Atlas_brain Atlas_heart Atlas_kidney BodyMap_brain BodyMap_heart BodyMap_kidney   HPA_brain
ENSG00000060138        YBX3      9.07246e+00 2.25311e+02     243.229 2266.430000  29731.30000     2030.770   4.95210e+01   3.20109e+02        123.268    1.76349e+01
ENSG00000109971       HSPA8      6.27816e+02 4.84668e+02     508.538 2205.090000  2103.80000     1276.960   4.20237e+02   2.98899e+02        605.504    6.47549e+02
ENSG00000130203        APOE      8.03945e+02 6.57937e+01    7062.530   97.184500     9.90770      131.672   2.36936e+02   7.50127e+00        318.506    7.02494e+02
ENSG00000136872       ALDOB      2.32646e-01 2.62751e-01    3923.870    0.141965     1.94897      839.917   4.26264e-01   2.48512e-01        492.240    2.34255e-01
ENSG00000166598     HSP90B1      1.02055e+02 6.23045e+01     197.881 1934.760000  5957.27000     2593.600   1.05416e+02   9.51998e+01        415.542    1.44129e+02
ENSG00000198804      MT-CO1      9.52453e+03 3.87410e+04    3772.820 4331.870000 12564.70000    12148.200   1.41602e+04   4.93494e+04      30835.800    1.12354e+04
ENSG00000198840      MT-ND3      1.11546e+04 1.12741e+04   16370.800 4281.000000 11015.10000    11389.900   2.17279e+04   3.45605e+04      32774.300    5.55203e+03
ENSG00000198888      MT-ND1      9.78313e+03 1.29704e+04   12927.400 6609.070000 11043.30000     9043.730   1.26450e+04   2.88022e+04      12167.300    3.65390e+03
ENSG00000198899     MT-ATP6      1.45439e+04 4.36996e+04   27038.300 8168.860000 16937.60000    16023.400   2.62130e+04   4.91401e+04      33007.100    8.50714e+03
ENSG00000198938      MT-CO3      1.03779e+04 2.79864e+04   13574.300 2072.480000  3861.35000     4634.460   1.24644e+04   3.89022e+04      29411.400    7.53832e+03
ENSG00000211445        GPX3      3.09235e+01 5.42119e+02   18989.300   11.297300   392.65900     2265.160   2.87217e+01   3.66150e+02       3920.100    8.35414e+01
ENSG00000228253     MT-ATP8      3.59405e+04 1.25506e+04   29657.800 5519.300000 12604.10000    11548.200   2.71870e+05   5.17674e+04     172497.000    1.55311e+04
                  
                  HPA_heart     HPA_kidney    AltIso_brain    AltIso_heart
                  2.08790e+02    41.4174      2.45855e+01  3.08346e+02
                  2.97209e+02    716.9350     2.93391e+02  1.23311e+02
                  1.75460e+01    1933.3900    2.60673e+02  2.13628e+01
                  2.95117e-01    3730.3700    2.18647e-01  1.61570e-01
                  1.00780e+02    192.4100     6.63249e+01  6.66813e+01
                  6.49431e+04    18088.5000   1.69366e+04  3.74794e+04
                  4.18535e+04    12978.9000   6.82735e+04  1.23074e+05
                  2.35580e+04    6011.1400    2.61690e+04  3.01527e+04
                  5.36826e+04    16901.4000   3.76973e+04  6.11711e+04
                  5.67093e+04    15282.4000   2.02301e+04  3.17234e+04
                  7.29493e+02    8203.3100    2.39292e+01  4.08317e+02
                  1.42989e+05    34584.4000   1.93776e+05  3.27884e+05
 
 
All mitochondrially encoded genes have relatively high expresseion levels, FPKM values of several thousands.

Anova analysis of different batch factors:


```r
p <- melt(cufflinks_fpkms)
```

```
## Using  as id variables
```

```r
colnames(p) <- c("sample_ID","Cuff_FPKM")
meta <- sampleinfo_cufflinks[,c("Study","Tissue","Preparation","NumberRaw","Numbermapped","Readtype")]
rownames(meta) <- colnames(cufflinks_fpkms)
tissue <- rep(meta$Tissue, each=nrow(cufflinks_fpkms))
study <- rep(meta$Study, each=nrow(cufflinks_fpkms))
prep <- rep(meta$Preparation, each=nrow(cufflinks_fpkms))
layout <- rep(meta$Readtype, each=nrow(cufflinks_fpkms))
nraw <- rep(meta$NumberRaw, each=nrow(cufflinks_fpkms))
data <- data.frame(p, tissue=tissue, study=study, prep=prep, layout=layout, nraw=nraw)
fit <- lm(Cuff_FPKM ~ layout + prep + nraw + study + tissue, data=data)
d <- anova(fit)
maxval = 100

barplot(100*d$"Sum Sq"[1:5]/sum(d$"Sum Sq"[1:5]),names.arg=rownames(d[1:5,]),main="Anova, Cufflinks FPKM",ylim=c(0,maxval))
```

![plot of chunk :cufflinks anova](figure/:cufflinks anova.png) 

Try log2 tranform the reprocessed FPKM values:


```r
pseudo <- 1
cufflinks_log <- log2(cufflinks_fpkms + pseudo)
```

Heatmap of Spearman correlations between log2 reprocessed cufflinks FPKM values:


```r
pheatmap(cor(cufflinks_log) ,method="spearman")
```

![plot of chunk :cufflinks log heatmap spearman](figure/:cufflinks log heatmap spearman.png) 

PCA analysis of log2 reprocessed cufflinks FPKM values:


```r
colors <- c("dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred")          

p.log.cufflinks <- prcomp(t(cufflinks_log))

plot(p.log.cufflinks$x[,1],p.log.cufflinks$x[,2],pch=20,col=colors,xlab=paste("PC1 33% of variance"),ylab=paste("PC2 25% of variance"),main="log2 reprocessed cufflinks FPKM values \n n=19,475")
legend("bottomright",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
```

![plot of chunk :cufflinks log PCA](figure/:cufflinks log PCA1.png) 

```r
plot(p.log.cufflinks$x[,2],p.log.cufflinks$x[,3],pch=20,col=colors,xlab=paste("PC2 25% of variance"),ylab=paste("PC3 22% of variance"),main="log2 reprocessed cufflinks FPKM values \n n=19,475")
legend("bottomright",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
```

![plot of chunk :cufflinks log PCA](figure/:cufflinks log PCA2.png) 

We can plot all pairwise combinations of principal components 1 to 5. (not shown in paper):


```r
par(mfrow=c(4,4))
for (i in 1:6){
  for(j in 1:6){
    if (i<j){ 
  	plot(p.log.cufflinks$x[,i],p.log.cufflinks$x[,j],pch=20,col=colors,xlab=paste("PC",i),ylab=paste("PC",j),main="log2 reprocessed FPKM values \n n=19475")
		}
	}
}
```

![plot of chunk :cufflinks log PCA pairwise](figure/:cufflinks log PCA pairwise.png) 

Look a bit closer at PCs 1-3 in prcomp for the logged FPKM values from cufflinks:


```r
      load.pc1 <- p.log.cufflinks$rotation[,1][order(p.log.cufflinks$rotation[,1])]
      extreme.pc1 <- c(tail(load.pc1), head(load.pc1))
      
      extreme.pc1.ensg <- names(extreme.pc1)
      extreme.pc1.symbols <- getBM(attributes=c("ensembl_gene_id", "hgnc_symbol"), 
                           filters = "ensembl_gene_id",
                           values=extreme.pc1.ensg,
                           mart=ensembl)
      
      barplot(extreme.pc1, names.arg=extreme.pc1.symbols[,2], las=2, main="Genes w highest absolute loadings in PC1 (log2Cufflinks FPKM)")
```

![plot of chunk :cufflinks log PC 1-3](figure/:cufflinks log PC 1-31.png) 

```r
      load.pc2 <- p.log.cufflinks$rotation[,2][order(p.log.cufflinks$rotation[,2])]
      extreme.pc2 <- c(tail(load.pc2), head(load.pc2))
      
      extreme.pc2.ensg <- names(extreme.pc2)
      extreme.pc2.symbols <- getBM(attributes=c("ensembl_gene_id", "hgnc_symbol"), 
                           filters = "ensembl_gene_id",
                           values=extreme.pc2.ensg,
                           mart=ensembl)
      
      barplot(extreme.pc2, names.arg=extreme.pc2.symbols[,2], las=2, main="Genes w highest absolute loadings in PC2 (log2Cufflinks FPKM)")
```

![plot of chunk :cufflinks log PC 1-3](figure/:cufflinks log PC 1-32.png) 

```r
      load.pc3 <- p.log.cufflinks$rotation[,3][order(p.log.cufflinks$rotation[,3])]
      extreme.pc3 <- c(tail(load.pc3), head(load.pc3))
      
      extreme.pc3.ensg <- names(extreme.pc3)
      extreme.pc3.symbols <- getBM(attributes=c("ensembl_gene_id", "hgnc_symbol"), 
                           filters = "ensembl_gene_id",
                           values=extreme.pc3.ensg,
                           mart=ensembl)
      
      barplot(extreme.pc3, names.arg=extreme.pc3.symbols[,2], las=2, main="Genes w highest absolute loadings in PC3 (log2Cufflinks FPKM)")
```

![plot of chunk :cufflinks log PC 1-3](figure/:cufflinks log PC 1-33.png) 

Seems to yield a heart vs. brain separation

PC1:
ensembl_gene_id hgnc_symbol  EoGE_brain EoGE_heart EoGE_kidney Atlas_brain Atlas_heart Atlas_kidney BodyMap_brain BodyMap_heart BodyMap_kidney HPA_brain
ENSG00000091513          TF    9.072460  225.31100 2.43229e+02  2266.43000 29731.30000  2030.770000     49.521000     320.10900     123.268000   17.6349
ENSG00000111245        MYL2  649.714000  401.25200 8.33284e+03    10.07880    26.61130    75.759900    482.139000     369.82500    2650.190000  524.2920
ENSG00000114854       TNNC1    0.693233 4082.36000 2.47695e-01     0.00000   317.58200     0.000000      0.187680     462.30200       0.000000    0.0000
ENSG00000123560        PLP1 1371.960000 5534.68000 7.19101e+03   141.29900  1082.37000   383.309000   1244.870000    2252.56000    2233.280000 1888.7900
ENSG00000125462     C1orf61  325.661000    0.90371 5.74676e+03   164.96600    14.18270   871.652000    541.821000       9.02186    5605.570000  190.9560
ENSG00000131095        GFAP 5495.280000  275.90600 2.84596e+03  3538.52000   170.58400   330.082000   2412.680000      94.74010     558.195000 5895.8000
ENSG00000132639      SNAP25 2829.730000    8.54714 0.00000e+00   474.81000     2.27603     0.328358   1993.670000       3.17969       0.266083 1789.8700
ENSG00000148677      ANKRD1  803.945000   65.79370 7.06253e+03    97.18450     9.90770   131.672000    236.936000       7.50127     318.506000  702.4940
ENSG00000159251       ACTC1 1848.190000    9.74111 5.94327e+00   934.18000     0.96437     5.572650   2412.650000       1.40515       0.452636 2752.7200
ENSG00000175084         DES    2.841210  451.67200 3.93243e-01     0.69976   425.58600     0.160330      0.912246     262.60000       0.306539   11.0686
ENSG00000197971         MBP 6099.710000    9.18006 2.85108e+01  2304.66000    12.39860    23.004400   3361.860000       6.55905      12.862700 2097.8600
ENSG00000198125          MB   30.923500  542.11900 1.89893e+04    11.29730   392.65900  2265.160000     28.721700     366.15000    3920.100000   83.5414
                  HPA_heart  HPA_kidney AltIso_brain AltIso_heart
ENSG00000060138   208.79000   41.417400     24.58550    308.34600
ENSG00000087086   790.22800 4210.010000    598.34800    483.14100
ENSG00000106631 10010.90000    0.000000      0.00000    851.30800
ENSG00000109846  2283.73000  359.345000   2399.48000   4851.69000
ENSG00000118785     4.53991 1718.990000    225.45600     15.52530
ENSG00000120885   214.37800  729.142000   6333.52000    206.25600
ENSG00000123560    13.69040    0.236948   2404.65000      3.63968
ENSG00000130203    17.54600 1933.390000    260.67300     21.36280
ENSG00000131095     2.58878    0.572929   1472.64000     11.55800
ENSG00000175206 25526.30000    0.000000      3.20302    737.39000
ENSG00000197971    10.46300   19.881500  15257.10000     11.95440
ENSG00000211445   729.49300 8203.310000     23.92920    408.31700

PC2:
ensembl_gene_id hgnc_symbol EoGE_brain  EoGE_heart EoGE_kidney Atlas_brain Atlas_heart Atlas_kidney BodyMap_brain BodyMap_heart BodyMap_kidney  HPA_brain
ENSG00000077522       ACTN2  51.106500 1275.910000    0.000000 3.80910e+01 1.42103e+03     2.250640    12.1376000   4.90980e+02       0.417171 15.6188000
ESG00000092054        MYH7   1.119220 7826.230000    0.124365 9.55006e-01 3.06060e+03     0.445822     0.4532170   2.77221e+03       0.054512  4.2841700
ENSG00000095932    C19orf77   6.958530    0.000000 1019.650000 6.48434e-02 0.00000e+00   237.905000     5.0188200   0.00000e+00     178.100000  0.9530750
ENSG00000129991       TNNI3   1.061300 3832.320000    0.000000 1.88834e+00 3.11226e+03     0.425951     0.4234730   1.48068e+03       0.000000  0.1386610
ENSG00000136872       ALDOB   0.232646    0.262751 3923.870000 1.41965e-01 1.94897e+00   839.917000     0.4262640   2.48512e-01     492.240000  0.2342550
ENSG00000137731       FXYD2   0.085333    1.057280 2744.210000 0.00000e+00 3.14842e-02    83.444000     0.0788030   2.42817e-01    1244.340000  0.2491600
ENSG00000148677      ANKRD1   0.000000  400.980000    0.000000 4.71658e-01 4.32044e+03     0.438835     0.0464207   7.95391e+02       0.278846  0.0494568
ENSG00000162366    PDZK1IP1   0.000000    0.338415 4259.220000 0.00000e+00 1.76553e-01    34.162000     0.2206780   6.88281e-02     525.810000  0.0000000
ENSG00000164825       DEFB1   0.000000    0.000000 1469.920000 1.44730e-01 0.00000e+00   300.820000     0.4429480   9.55738e-01     289.458000  0.0000000
ENSG00000169344        UMOD   0.000000    0.000000 1380.650000 0.00000e+00 0.00000e+00   701.195000     0.0000000   0.00000e+00    1357.400000  0.0000000
ENSG00000196233        LCOR   1.771630    2.036680    1.675120 1.41164e+03 4.79727e+02   604.976000     5.2570900   1.99915e+00       3.941860  4.2944100
ENSG00000224867               0.000000    0.000000    0.000000 1.65713e+02 2.19459e+02   305.302000    48.0801000   0.00000e+00       1.643140  0.0000000
                  HPA_heart  HPA_kidney AltIso_brain AltIso_heart
ENSG00000077522  601.310000 3.15824e-01   15.5911000   559.541000
ENSG00000092054  328.643000 7.33492e-02    0.6425810  2699.210000
ENSG00000095932    0.000000 6.26060e+02    5.2137000     0.579565
ENSG00000129991 1103.600000 0.00000e+00    1.0688300  3405.670000
ENSG00000136872    0.295117 3.73037e+03    0.2186470     0.161570
ENSG00000137731    0.738869 1.50256e+03    0.0176443     0.148702
ENSG00000148677 1857.880000 7.97934e-02    0.3143330   793.048000
ENSG00000162366    0.217821 1.30491e+03    0.0000000     0.282131
ENSG00000164825    0.000000 9.01979e+02    0.2545150     1.097130
ENSG00000169344    0.000000 2.09624e+03    0.0000000     0.000000
ENSG00000196233    2.325930 6.51514e+00    1.8125400     1.759250
ENSG00000224867    0.000000 0.00000e+00    3.8624100     5.694370

PC3:
ensembl_gene_id hgnc_symbol  EoGE_brain  EoGE_heart EoGE_kidney Atlas_brain Atlas_heart Atlas_kidney BodyMap_brain BodyMap_heart BodyMap_kidney   HPA_brain
ENSG00000092054        MYH7   1.1192200 7826.230000    0.124365   0.9550060  3060.60000  4.45822e-01     0.4532170   2.77221e+03    5.45120e-02   4.2841700
ENSG00000104879         CKM   0.0800316 3766.780000    0.921574   0.0000000    69.11510  2.89714e-02     0.0474386   1.18818e+03    5.96727e-02   0.0302300
ENSG00000106631        MYL7   0.6932330 4082.360000    0.247695   0.0000000   317.58200  0.00000e+00     0.1876800   4.62302e+02    0.00000e+00   0.0000000
ENSG00000118785        SPP1 325.6610000    0.903710 5746.760000 164.9660000    14.18270  8.71652e+02   541.8210000   9.02186e+00    5.60557e+03 190.9560000
ENSG00000136872       ALDOB   0.2326460    0.262751 3923.870000   0.1419650     1.94897  8.39917e+02     0.4262640   2.48512e-01    4.92240e+02   0.2342550
ENSG00000145692        BHMT   1.2324900    0.227735  524.560000   2.1022100     2.41740  1.42418e+03     0.2495350   5.49817e-02    5.09868e+01   0.0872756
ENSG00000159251       ACTC1   0.7813530 8115.270000    2.472130   0.4595020   306.74400  3.62615e-01     0.4032380   2.51572e+03    4.92426e+00   0.2033750
ENSG00000163586       FABP1   0.0000000    0.523537  911.884000   0.0000000     2.37512  3.56154e+02     0.1256400   0.00000e+00    6.95123e+01   0.0000000
ENSG00000164825       DEFB1   0.0000000    0.000000 1469.920000   0.1447300     0.00000  3.00820e+02     0.4429480   9.55738e-01    2.89458e+02   0.0000000
ENSG00000169344        UMOD   0.0000000    0.000000 1380.650000   0.0000000     0.00000  7.01195e+02     0.0000000   0.00000e+00    1.35740e+03   0.0000000
ENSG00000173991        TCAP   3.5296600 5591.530000    0.980538   0.2285410   552.90400  2.19431e-01     0.5623330   6.06384e+02    4.70557e-01   1.5549800
ENSG00000198125          MB   0.4223660 6146.640000    1.012190   0.0895293   645.03900  0.00000e+00     0.2452290   4.56240e+03    2.97123e-01   0.7569730
                  HPA_heart  HPA_kidney AltIso_brain AltIso_heart
ENSG00000092054 3.28643e+02 7.33492e-02     0.642581  2699.210000
ENSG00000104879 1.22132e+03 2.91014e-01     0.217256  2304.880000
ENSG00000106631 1.00109e+04 0.00000e+00     0.000000   851.308000
ENSG00000118785 4.53991e+00 1.71899e+03   225.456000    15.525300
ENSG00000136872 2.95117e-01 3.73037e+03     0.218647     0.161570
ENSG00000145692 6.10857e-02 7.70503e+02     1.009370     0.350104
ENSG00000159251 4.20522e+03 4.57615e-01     1.214480  1725.760000
ENSG00000163586 0.00000e+00 4.37508e+02     0.204203     0.000000
ENSG00000164825 0.00000e+00 9.01979e+02     0.254515     1.097130
ENSG00000169344 0.00000e+00 2.09624e+03     0.000000     0.000000
ENSG00000173991 7.63777e+02 1.70633e-01     1.213970  2399.890000
ENSG00000198125 6.00091e+03 6.13148e-01     0.809898  7280.090000

To further validate the above results, indicating that tissue specificity appears mainly in PC 3, we will extract the 500 genes with highest loadings in each component and plot the corresponding cufflinks FPKM values in a heatmap:


```r
     cufflinks_values <- cufflinks_pc_nozero[,3:16]
     rownames(cufflinks_values) <- cufflinks_pc_nozero[,1]

     load.pc1 <- abs(p.log.cufflinks$rotation[,1])[order(abs(p.log.cufflinks$rotation[,1]),decreasing=TRUE)]
     top.pc1 <- names(load.pc1[1:500])
     load.pc2 <- abs(p.log.cufflinks$rotation[,2])[order(abs(p.log.cufflinks$rotation[,2]),decreasing=TRUE)]
     top.pc2 <- names(load.pc2[1:500])
     load.pc3 <- abs(p.log.cufflinks$rotation[,3])[order(abs(p.log.cufflinks$rotation[,3]),decreasing=TRUE)]
     top.pc3 <- names(load.pc3[1:500])
     
     pheatmap(cor(cufflinks_values[top.pc1,]),method="spearman")
```

![plot of chunk :cufflinks log top 500 loadings heatmap](figure/:cufflinks log top 500 loadings heatmap1.png) 

```r
     pheatmap(cor(cufflinks_values[top.pc2,]),method="spearman")
```

![plot of chunk :cufflinks log top 500 loadings heatmap](figure/:cufflinks log top 500 loadings heatmap2.png) 

```r
     pheatmap(cor(cufflinks_values[top.pc3,]),method="spearman")
```

![plot of chunk :cufflinks log top 500 loadings heatmap](figure/:cufflinks log top 500 loadings heatmap3.png) 
    
Try Anova on a "melted" expression matrix with logged cufflinks values and some metadata:


```r
q <- melt(cufflinks_log[,])
```

```
## Using  as id variables
```

```r
colnames(q) <- c("sample_ID","logFPKM")
meta <- sampleinfo_cufflinks[,c("Study","Tissue","Preparation","NumberRaw","Numbermapped","Readtype")]
rownames(meta) <- colnames(cufflinks_log)
tissue <- rep(meta$Tissue, each=nrow(cufflinks_log))
study <- rep(meta$Study, each=nrow(cufflinks_log))
prep <- rep(meta$Preparation, each=nrow(cufflinks_log))
layout <- rep(meta$Readtype, each=nrow(cufflinks_log))
data <- data.frame(q, tissue=tissue, study=study, prep=prep, layout=layout)
fit <- lm(Cuff_FPKM ~ layout + prep + nraw + study + tissue, data=data)
```

```
## Error: object 'Cuff_FPKM' not found
```

```r
e <- anova(fit)
maxval = 100

barplot(100*e$"Sum Sq"[1:5]/sum(e$"Sum Sq"[1:5]),names.arg=rownames(e[1:5,]),main="Anova, Cufflinks log2 FPKM",ylim=c(0,maxval))
```

![plot of chunk :cufflinks log anova](figure/:cufflinks log anova.png) 

Combat analysis for removal of batch effects (n=19,475):


```r
meta <- data.frame(study=c(rep("EoGE",3),rep("Atlas",3),rep("BodyMap",3),rep("HPA",3),rep("AltIso",2)),tissue=c("Brain","Heart","Kidney","Brain","Heart","Kidney","Brain","Heart","Kidney","Brain","Heart","Kidney","Brain","Heart"),prep=c(rep("poly-A",3),rep("rRNA-depl",3),rep("poly-A",8)),layout=c(rep("PE",3),rep("SE",3),rep("PE",6),rep("SE",2)))

batch <- meta$study
design <- model.matrix(~as.factor(tissue),data=meta)

combat.cufflinks <- ComBat(dat=cufflinks_log,batch=batch,mod=design,numCovs=NULL,par.prior=TRUE)
```

```
## Found 5 batches
## Found 2  categorical covariate(s)
## Standardizing Data across genes
## Fitting L/S model and finding priors
## Finding parametric adjustments
## Adjusting the Data
```

```r
rownames(combat.cufflinks) <- rownames(cufflinks_log)

#write.table(combat, file="reprocessed_rpkms_combat_log2.txt", quote=F)
```

Let's see how the correlation heatmap and PCA plots look after correction for batch effects with combat:


```r
pheatmap(cor(combat.cufflinks),method="spearman")
```

![plot of chunk :cufflinks log combat heatmap spearman](figure/:cufflinks log combat heatmap spearman.png) 

PCA analysis on reprocessed cufflinks FPKM values after ComBat run:


```r
p.combat.cufflinks <- prcomp(t(combat.cufflinks))

plot(p.combat.cufflinks$x[,1],p.combat.cufflinks$x[,2],pch=20,col=colors,xlab=paste("PC1 54% of variance"),ylab=paste("PC2 37% of variance"),main="Cufflinks FPKM values \n COMBAT \n n=19,475")
legend("bottomright",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
```

![plot of chunk :cufflinks log combat PCA](figure/:cufflinks log combat PCA1.png) 

```r
plot(p.combat.cufflinks$x[,2],p.combat.cufflinks$x[,3],pch=20,col=colors,xlab=paste("PC2 37% of variance"),ylab=paste("PC3 2% of variance"),main="Cufflinks FPKM values \n COMBAT \n n=19,475")
legend("bottomright",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
```

![plot of chunk :cufflinks log combat PCA](figure/:cufflinks log combat PCA2.png) 

We can plot all pairwise combinations of principal components 1 to 5. (not shown in paper):


```r
par(mfrow=c(4,4))
for (i in 1:6){
  for(j in 1:6){
    if (i<j){ 
  	plot(p.combat.cufflinks$x[,i],p.combat.cufflinks$x[,j],pch=20,col=colors,xlab=paste("PC",i),ylab=paste("PC",j),main="Cufflinks FPKM values \n COMBAT \ n=19,475")
		}
	}
}
```

![plot of chunk :cufflinks log combat PCA pairwise](figure/:cufflinks log combat PCA pairwise.png) 

Look a bit closer at PCs 1-3 in prcomp:


```r
      load.pc1 <- p.combat.cufflinks$rotation[,1][order(p.combat.cufflinks$rotation[,1])]
      extreme.pc1 <- c(tail(load.pc1), head(load.pc1))
      
      extreme.pc1.ensg <- names(extreme.pc1)
      extreme.pc1.symbols <- getBM(attributes=c("ensembl_gene_id", "hgnc_symbol"), 
                           filters = "ensembl_gene_id",
                           values=extreme.pc1.ensg,
                           mart=ensembl)
      
      barplot(extreme.pc1, names.arg=extreme.pc1.symbols[,2], las=2, main="Genes w highest absolute loadings in PC1 (COMBAT Cufflinks FPKM)")
```

![plot of chunk :cufflinks log combat PC 1-3](figure/:cufflinks log combat PC 1-31.png) 

```r
      load.pc2 <- p.combat.cufflinks$rotation[,2][order(p.combat.cufflinks$rotation[,2])]
      extreme.pc2 <- c(tail(load.pc2), head(load.pc2))
      
      extreme.pc2.ensg <- names(extreme.pc2)
      extreme.pc2.symbols <- getBM(attributes=c("ensembl_gene_id", "hgnc_symbol"), 
                           filters = "ensembl_gene_id",
                           values=extreme.pc2.ensg,
                           mart=ensembl)
      
      
      barplot(extreme.pc2, names.arg=extreme.pc2.symbols[,2], las=2, main="Genes w highest absolute loadings in PC2 (Combat Cufflinks FPKM)")
```

![plot of chunk :cufflinks log combat PC 1-3](figure/:cufflinks log combat PC 1-32.png) 

```r
      load.pc3 <- p.combat.cufflinks$rotation[,3][order(p.combat.cufflinks$rotation[,3])]
      extreme.pc3 <- c(tail(load.pc3), head(load.pc3))
      
      extreme.pc3.ensg <- names(extreme.pc3)
      extreme.pc3.symbols <- getBM(attributes=c("ensembl_gene_id", "hgnc_symbol"), 
                           filters = "ensembl_gene_id",
                           values=extreme.pc3.ensg,
                           mart=ensembl)
      
      
      barplot(extreme.pc3, names.arg=extreme.pc3.symbols[,2], las=2, main="Genes w highest absolute loadings in PC3 (Combat Cufflinks FPKM)")
```

![plot of chunk :cufflinks log combat PC 1-3](figure/:cufflinks log combat PC 1-33.png) 
Let's have a look at the genes with highest loadings:

PC1:
ENSG00000173641 - HSPB7 - heat shock 27kDa protein family, member 7 - heart specific
ENSG00000175084 - DES- desmin - heart specific
ENSG00000159251 - ACTC1 - actin, alpha, cardiac muscle 1 - heart specific
ENSG00000111245 - MYL2 - myosin, light chain 2, regulatory, cardiac, slow - heart specific
ENSG00000198125 - MB - myoglobin - heart specific
ENSG00000114854 - TNNC1 - troponin C type 1 - heart specific
ENSG00000132639 - SNAP25 - synaptosomal-associated protein, 25kDa - brain specific
ENSG00000123560 - PLP1 - proteolipid protein 1 - brain specific
ENSG00000131095 - GFAP - glial fibrillary acidic protein - brain specific
ENSG00000197971 - MBP - myelin basic protein - brain specific
ENSG00000091513 - TF - transferrin - brain specific
ENSG00000125462 - C1orf61 - chromosome 1 open reading frame 61 - brain specific

PC2:
ENSG00000148677 - ANKRD1 - ankyrin repeat domain 1 - heart specific
ENSG00000111245 - MYL2 - myosin, light chain 2, regulatory - heart specific
ENSG00000106631 - MYL7 - myosin, light chain 7, regulatory - heart specific
ENSG00000129991 - TNNI3 - troponin I type 3 - heart specific
ENSG00000092054 - MYH7 - myosin, heavy chain 7 - heart specific
ENSG00000198125 - MB - myoglobin - heart specific
ENSG00000169344 - UMOD - uromodulin - kidney specific (FPKM 0 in all others)
ENSG00000136872 - ALDOB - aldolase B, fructose-bisphosphate - kidney specific
ENSG00000137731 - FXYD2 - FXYD domain containing ion transport regulator 2 - kidney specific
ENSG00000162366 - PDZK1IP1 - PDZK1 interacting protein 1 - kidney specific
ENSG00000164825 - DEFB1 - defensin, beta 1 - kidney specific
ENSG00000095932 - C19orf77 - small integral membrane protein 24 - kidney specific

PC3:
ENSG00000090932 - DLL3 - delta-like 3 (Drosophila) - brain (but relatively low expressed)
ENSG00000088882 - CPXM1 - carboxypeptidase X (M14 family), member 1 - brain
ENSG00000023228 - NDUFS1 - NADH dehydrogenase (ubiquinone) Fe-S protein 1, 75kDa - quite similar expression
ENSG00000157150 - TIMP4 - TIMP metallopeptidase inhibitor 4 - heart
ENSG00000114942 - EEF1B2 - eukaryotic translation elongation factor 1 beta 2 - quite the same
ENSG00000266956 - No name
ENSG00000118271 - TTR - transthyretin - brain
ENSG00000268332 - No name - brain
ENSG00000010438 - RPSS3 - protease, serine, 3 - brain
ENSG00000105697 - HAMP - hepcidin antimicrobial peptide - brain
ENSG00000129824 - RPS4Y1 - ribosomal protein S4, Y-linked 1 - no clear tissue specificity
ENSG00000205116 - TMEM88B - transmembrane protein 88B - brain

Revisit ANOVA analysis on reprocessed Cufflinks FPKM values after ComBat run:


```r
q <- melt(combat.cufflinks)
```

```
## Using  as id variables
```

```r
colnames(q) <- c("sample_ID","combat")
meta <- sampleinfo_cufflinks[,c("Study","Tissue","Preparation","NumberRaw","Numbermapped","Readtype")]
rownames(meta) <- colnames(combat.cufflinks)
tissue <- rep(meta$Tissue, each=nrow(combat.cufflinks))
study <- rep(meta$Study, each=nrow(combat.cufflinks))
prep <- rep(meta$Preparation, each=nrow(combat.cufflinks))
layout <- rep(meta$Readtype, each=nrow(combat.cufflinks))
raw <- rep(meta$NumberRaw, each=nrow(combat.cufflinks))
mapped <- rep(meta$Numbermapped, each=nrow(combat.cufflinks))
data <- data.frame(q, tissue=tissue, study=study, prep=prep, layout=layout,nraw=raw,nmapped=mapped)
fit <- lm(combat ~ prep + nraw + layout + study + tissue, data=data)
f <- anova(fit)
maxval = 100
barplot(100*f$"Sum Sq"/sum(f$"Sum Sq"),names.arg=rownames(f),main="Anova Cufflinks Combat",ylim=c(0,maxval))
```

![plot of chunk :cufflinks log combat anova](figure/:cufflinks log combat anova.png) 