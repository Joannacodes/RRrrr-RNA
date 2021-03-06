# Charville (2015) RNA-Seq Data Re-analysis Pt.II

Continued from the analysis in part I. After looking at the undelrying trends in the data, can run some stats accordingly.

# Dependencies

```{r}
library(edgeR)
library(preprocessCore)
library(ggplot2)
```

# EdgeR
For DE analysis, use the raw, filtered data as opposed to the (quantile) normalized data, and use EdgeR's TMM function instead. 

``` {r}
pData
head(filt_edata)

# define 3 groups
group <- factor(paste(pData$Type,pData$Tx, sep="."))
pData$group <- group
pData$group <- relevel(pData$group, ref="Act.None")

#design matrix
design <- model.matrix(~Donor+group, data=pData)
design

#create dge object
cds1 <- DGEList(filt_edata)
dim(cds1)
head(cds1$counts)

#estimate disp + normalization
cds1<- estimateDisp(cds1, design=design)
cds1 <- calcNormFactors(cds1)
cds1$samples

#fit a glm
fit <- glmFit(cds1,design)
colnames(fit)
```

With this design matrix, edgeR compares samples in a paired manner (within a single donor).
We are interested in two comparisons: Quiescent vs. Activated, and p38i vs. Activated. 

## DEGs in Quiescent cells compared to activated cells

```{r}
lrt <- glmLRT(fit)
summary(de <- decideTestsDGE(lrt))
detags <- rownames(cds1)[as.logical(de)]
plotSmear(lrt, de.tags=detags, main="DEGs between Q and A cells")
abline(h=c(-1, 1), col="blue")
```

Now to have a look at the CPM of the top DE genes:
```{r}
o <- order(lrt$table$PValue)
cpm(cds1)[o[1:10],1:4]
```

A volcano plot to visualize significant DEGs.

```{r}
lrt.Tbl <- lrt$table 
de.genes <- lrt$table[as.logical(de),]
sum(abs(de.genes$logFC)>2) 
```
This last line gives the number of genes above this cutoff. The table can be saved in a csv file.
BH correction and setting the threshold for significance:

```{r}
lrt.Tbl$PValue_adj = p.adjust(lrt.Tbl$PValue, method='BH')
lrt.Tbl$threshold = as.factor(abs(lrt.Tbl$logFC)>2 & lrt.Tbl$PValue_adj<0.05)
table(lrt.Tbl$threshold)

ggplot(data=lrt.Tbl, aes(x=logFC, y=-log10(PValue), colour=threshold)) + 
  geom_point(alpha=0.4, size=1.75) + 
  xlab("log2 fold change") + 
  ylab("-log10 p-value")+
  ggtitle("DEGs Q vs A")
```

## DEGs in p38 inhibitor treated cells compared to activated cells

```{r}
colnames(fit)
lrt_Tx <- glmLRT(fit, coef=3)

summary(de_Tx <- decideTestsDGE(lrt_Tx))
detags_Tx <- rownames(cds1)[as.logical(de_Tx)]
plotSmear(lrt_Tx, de.tags=detags_Tx, main="DE genes between A and A+p38i cells")
abline(h=c(-1, 1), col="blue")

lrt_Tx.Tbl <- lrt_Tx$table 
de.genes_Tx <- lrt_Tx$table[as.logical(de_Tx),]
sum(abs(de.genes_Tx$logFC)>2)
```

Again, look at CPMs for the top DE genes:
```{r}
o_Tx <- order(lrt_Tx$table$PValue)
cpm(cds1)[o[1:10],]
```

## Expression of the top DEGs for each analysis

Below, we plot a histogram of log concentrations for the top 100 genes for each analysis.
```{r}
glm.DE <- rownames(lrt.Tbl)[as.numeric(lrt.Tbl$threshold)>1 ]
length(glm.DE) #how many genes are significantly DE
length(glm.DE)/nrow(lrt.Tbl) * 100
```
Equal to the percentage of total genes (after filtering), roughly 20% here. 

```{r}
hist(lrt.Tbl[glm.DE[1:1000],"logCPM"], xlab="Log CPM", xlim=c(-2,15),col="red", freq=FALSE, main="Expression of top 100 DE genes - Q vs A")
```
Now for the p38i treatment sample set:

```{r}
lrt_Tx.Tbl$PValue_adj = p.adjust(lrt_Tx.Tbl$PValue, method='BH')
lrt_Tx.Tbl$threshold = as.factor(abs(lrt_Tx.Tbl$logFC)>2 & lrt_Tx.Tbl$PValue_adj<0.05)
glmTx.DE <- rownames(lrt_Tx.Tbl)[as.numeric(lrt_Tx.Tbl$threshold)>1 ]
length(glmTx.DE)/nrow(lrt_Tx.Tbl) * 100
```
A much smaller percentage here, only ~1%. 

```{r}
hist(lrt_Tx.Tbl[glmTx.DE[1:1000],"logCPM"], xlab="Log CPM", xlim=c(-2,15),col="red", freq=FALSE, main="Expression of top 100 DE genes - p38i vs A")

# SessionInfo
```{r echo=FALSE} 
sessionInfo()
```
