##BICF R nano course: RNA-seq analysis with R

[Back to main course page](https://portal.biohpc.swmed.edu/content/training/bioinformatics-nanocourses/rbio/)

####Today we are going to do:
+ Use edgeR to normalize data and visualization
+ Use edgeR to do differential expressed genes
+ Differential expressed genes filtering and draw heatmap
+ Use Ballgown to visualize the transcripts and do differential analysis 


###Run edgeR for gene differential expression analysis
Open your Rstudio and install edgeR
```R
source("http://bioconductor.org/biocLite.R")
biocLite("edgeR",suppressUpdates = T)
```
For pheatmap package, click "package"->"install"
Type 'a' if program asks you if you want to update packages.
Install from:CRAN
Packages: pheatmap
Check "Install dependencies"
Click "Install"

Click "Yes" if prompt window asks you if you want to use a personal library.

Use the same way to install package "VennDiagram".

After everything finished, load the libraries using
```R
library("edgeR")
library("pheatmap")
```

**Load Data**
```R
#Set working directory
#Session-> Set working directory -> choose directory

#Read data matrix and sample file
cfile = read.table("RNAseq_count_table.txt",header=T,row.names=1,sep="\t")
coldata = read.table("RNAseq_design_pe.txt",header=T,sep="\t")
head(cfile)
head(coldata)

#Reorder the counts columns to match the order of sample file
cfile = cfile[coldata$SampleID]

#It is good to set your control group label as the baseline. Especially you are going to use intercept
group = relevel(factor(coldata$SampleGroup),ref="monocytes")
cds = DGEList(cfile,group=group)
```



**Pre-filtering the low-expressed genes.**

The code below means keep a gene if cpm (counts per million) exceeds 1 in at least 4 samples

```R
nrow(cds)
cds = cds[ rowSums(cpm(cds)>=1) >= 4, ,keep.lib.sizes=FALSE]
nrow(cds)
```



#### Explortory analysis and some visulization

Use cpm() function to get log2 transformed normalized counts
```R
rld <- cpm(cds, log=TRUE)

```

**Calculate the distance between sample pairs and do hierarchical clustering**
```R
sampleDists = dist(t(rld))
sampleDists
plot(hclust(sampleDists))
```


**Use a heatmap to show sample correlation **

```R
pheatmap(cor(rld))

```



**Use MDS and PCA plot to check the relationship of replicates**


edgeR provides MDA plot function but not PCA 
```R
#edgeR provides MDA plot function but not PCA
points = c(15,16)
colors = rep(c("red","blue"),4)
plotMDS(cds, col=colors[group], pch=points[group])
legend("topright", legend=levels(group), pch=points, col=colors, ncol=2)
```


** Let's try to generate a PCA plot**

```R
library(ggplot2)
pca <- prcomp(t(rld),center=T, scale=T) #You have to scale/normalize the data first
percentVar = pca$sdev^2/sum(pca$sdev^2)  #calculate the percentage of variance
d <- data.frame(PC1 = pca$x[, 1], PC2 = pca$x[, 2], group = group)
ggplot(data = d, aes_string(x = "PC1", y = "PC2", color = "group")) + geom_point(size = 3) + 
    xlab(paste0("PC1: ", round(percentVar[1] *100), "% variance")) + 
    ylab(paste0("PC2: ", round(percentVar[2] *100), "% variance"))

```



###Find differential expressed genes

**Make a design matrix**
```R
#Make a design matrix
samplegroup <- factor(coldata$SampleGroup)
design<-model.matrix(~samplegroup)
design
```


**Normalize data and estimate dispersion**

```R
#Normalize data and estimate dispersion
cds = calcNormFactors(cds)
cds$samples
cds <- estimateDisp(cds,design)

fit <- glmFit(cds, design)
lrt <- glmLRT(fit, coef=2)
res <- topTags(lrt, n=dim(cfile)[1],sort.by="logFC") #retrive all genes
```



You can also apply some other threshold (smaller p value or bigger logFoldChange value to filter the resutls)


```R
outdf = cbind(gene_name = rownames(res), data.frame(res))
write.table(outdf,"edgeR.res.xls",quote=F,sep="\t",row.names=F)
head(outdf)

deG = outdf[(abs(outdf$logFC)>=1 && outdf$FDR<=0.01),]
write.table(deG,'edgeR.deG.xls',quote=F,sep="\t",row.names=F)
dim(deG)

```

**Draw heatmap on differential expressed genes**

```R
deG_rld = rld[rownames(rld) %in% deG$gene_name,]
pheatmap(deG_rld,scale="row",show_rownames = F)
```



###Try to add interaction term

**Make a design matrix**
```R
#Make a design matrix
race <- factor(coldata$Race)
design_interaction<-model.matrix(~0+samplegroup:race)
design_interaction
```

**Re-calculate the dispersion using the new model**
```R
#Re-calculate the dispersion using the new model
cds<-estimateDisp(cds,design_interaction)
fit <- glmFit(cds, design_interaction)
lrt_w <- glmLRT(fit,contrast=c(0,0,-1,1)) #compare neutrophils vs monocyte of White
res_w <- topTags(lrt_w, n=dim(cfile)[1],sort.by="logFC") #retrive all genes
```


####Try to control for batch effect

** Make new design matrix and redo the analysis**
```R
#Make a design matrix
subjectid <- factor(coldata$SubjectID)
design_batch = model.matrix(~subjectid+samplegroup)
design_batch

#Re-calculate the dispersion using the new model
cds <- estimateDisp(cds,design_batch)
fit <-  glmFit(cds, design_batch)
lrt_b <- glmLRT(fit,coef=5)
res_b <- topTags(lrt_b, n=dim(cfile)[1],sort.by="logFC") #retrive all genes
outdf_b <- cbind(gene_name = rownames(res_b), data.frame(res_b))

```


** Compare the original and adjusted for batch effect results **

```R
library(VennDiagram)
dim(deG)
dim(deG_b)
overlap_count = dim(deG[deG$gene_name %in% deG_b$gene_name,])[1]
dim(overlap_count)

```




###Transcript differential expression analysis using Ballgown


Files output in Ballgown readable format are:

+ e_data.ctab: exon-level expression measurements. One row per exon. Columns are e_id (numeric exon id), chr, strand, start, end (genomic location of the exon), and the following expression measurements for each sample:
  + rcount: reads overlapping the exon
  + ucount: uniquely mapped reads overlapping the exon
  + mrcount: multi-map-corrected number of reads overlapping the exon
  + cov average per-base read coverage
  + cov_sd: standard deviation of per-base read coverage
  + mcov: multi-map-corrected average per-base read coverage
  + mcov_sd: standard deviation of multi-map-corrected per-base coverage
+ i_data.ctab: intron- (i.e., junction-) level expression measurements. One row per intron. Columns are i_id (numeric intron id), chr, strand, start, end (genomic location of the intron), and the following expression measurements for each sample:
  + rcount: number of reads supporting the intron
  + ucount: number of uniquely mapped reads supporting the intron
  + mrcount: multi-map-corrected number of reads supporting the intron
+ t_data.ctab: transcript-level expression measurements. One row per transcript. Columns are:
  + t_id: numeric transcript id
  + chr, strand, start, end: genomic location of the transcript
  + t_name: Cufflinks-generated transcript id
  + num_exons: number of exons comprising the transcript
  + length: transcript length, including both exons and introns
  + gene_id: gene the transcript belongs to
  + gene_name: HUGO gene name for the transcript, if known
  + cov: per-base coverage for the transcript (available for each sample)
  + FPKM: Cufflinks-estimated FPKM for the transcript (available for each sample)
+ e2t.ctab: table with two columns, e_id and t_id, denoting which exons belong to which transcripts. These ids match the ids in the e_data and t_data tables.
+ i2t.ctab: table with two columns, i_id and t_id, denoting which introns belong to which transcripts. These ids match the ids in the i_data and t_data tables.


While the program is running, please download the stringtie folder with pre-runned results in day3 to the folder you made. 
Now let's take a look at the folder structures.
For Ballgown to read the folder, it has to be all the ctab of each sample in an individual sample folder, and all sample folder have similar naming scheme, in our case "SRR155", for Ballgown to identify all the samples.


####Run Ballgown

Install ballgown:
```R
source("http://bioconductor.org/biocLite.R")
biocLite("ballgown")
library("ballgown")
```

Now use the session buttion to change the working directory to "isoform".

** Make ballgown object **
```R
stdir <- 'stringtie'
design <- "RNAseq_design_pe.txt"
samtbl <- read.table(file=design,header=TRUE,sep='\t')
#make bg object
bg <- ballgown(dataDir=stdir, samplePattern='SRR155', meas='all')
```



Browsing transcriptome data 
```R
structure(bg)$exon
structure(bg)$intron
structure(bg)$trans
```

Use expr to get expression data
Basic usage:
```
*expr(ballgown_object_name, <EXPRESSION_MEASUREMENT>)
```
Where * can be a letter represent different genomic features.
+ t: transcript
+ e: exon
+ i: intron
+ g: gene (Sum up all transcripts belonging to that gene, sometimes this takes a long time)

```R
transcript_fpkm = texpr(bg, 'FPKM')
head(transcript_fpkm)
junction_rcount = iexpr(bg)
head(junction_rcount)
whole_intron_table = iexpr(bg, 'all')
head(whole_intron_table)
gene_expression = gexpr(bg) #try 
head(gene_expression)
```
You will see a data frame with the expression value. The row names are the tids.
In order to get the transcript names/gene names, you need to use indexes slot to get the names

Now we are going to make a gene name table for the output file.

The indexes slot of a ballgown object connects the pieces of the assembly and provides other experimental information. indexes(bg) is a list with several components that can be extracted with the $ operator.
There are some components in indexes:
+ pData:  holds a data frame of phenotype information for the samples in the experiment. This must be created manually. It is **very important** that the rows of pData are in the correct order. Each row corresponds to a sample, and the rows of pData should be ordered the same as the tables in the expr slot. You can check that order by running sampleNames(bg).
+ t2g: denotes which transcripts belong to which genes
+ e2t: denotes which exons belong to which transcripts
+ i2t: denotes which introns belong to which transcripts

```R
#make gene-transcript relation tables, this will be used in the future to generate readable results
transcript_gene_table = indexes(bg)$t2g
head(transcript_gene_table)
transcript_name <- transcriptNames(bg)
head(transcript_name)
rownames(transcript_gene_table) <- transcript_name
head(transcript_gene_table)
genes <- unique(cbind(geneNames(bg),geneIDs(bg)))
colnames(genes) <- c('SYMBOL','ENSEMBL')
```
Now let try to add the names to expression data frame

```R
#take a look at the data frame structures again
head(transcript_fpkm)
head(transcript_gene_table)
transcript_fpkm_annotated = merge(transcript_fpkm, transcript_gene_table, by.x="row.names",by.y="t_id",all.x=TRUE)
head(transcript_fpkm_annotated)
dim(transcript_fpkm)
dim(transcript_fpkm_annotated)
```
Now you should know how to use expr and indexes to get annotated expression tables. Let get down to the diffenetial alternative splicing analysis

```R
#make phenotype information
samples <- sampleNames(bg)
mergetbl <- merge(as.data.frame(samples),samtbl,by.x="samples",by.y="SampleID",all.x=TRUE,sort=FALSE)
pData(bg) = data.frame(id=sampleNames(bg), group=as.character(mergetbl$SampleGroup),subj=as.character(mergetbl$SubjectID))
phenotype_table = pData(bg)

#testing
stat_results = stattest(bg, feature='transcript', meas='cov', covariate='group',getFC=TRUE)

#If you want to adjust for individual
stat_results = stattest(bg, feature='transcript', meas='cov', covariate='group',adjustvars='subj',getFC=TRUE)

r1 <- merge(transcript_gene_table,na.omit(stat_results),by.y='id',by.x='t_id',all.y=TRUE,all.x=FALSE)
r2 <- merge(genes,r1,by.y='g_id',by.x='ENSEMBL',all.y=TRUE,all.x=FALSE)
#write.table(r2,file='de_altsplice.bg.txt',quote=FALSE,sep='\t',row.names=FALSE)

#check r2
summary(r2)

#filter the table using different threshlod
res_q01<-r2[r2$qval<=0.01,]
write.table(res_q01,file='de_altsplice.txt',quote=FALSE,sep='\t',row.names=FALSE)

plotMeans('ENSG00000024048.10', bg, groupvar='group', meas='cov', colorby='transcript')
plotTranscripts(gene="ENSG00000024048.10",gown=bg,samples=samples,meas='FPKM', colorby='transcript')

```
