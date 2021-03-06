Sesion 1 | Part 2 : Gene expression analysis
================
Jonathan Maldonado
5/22/2020

Following the former pipeline, we were able to obtain a table with read
counts per gene of Arabidopsis. That table was generated using the
reduced version of the original sequencing files. For this pipeline, we
will use a similar file, containing gene counts obtained from the analysis of the original .fastq files.

Open R Studio (or just R) and load the following libraries.

``` r
library(edgeR)
library(knitr)
library(dplyr)
library(pvclust)
library(ggplot2)
require(gridExtra)
library(RColorBrewer)
library(mixOmics)
library(reshape2)
library(gplots)
library(mclust)
library(matrixStats)
```

Don’t forget to set the working directory right were your downloaded
files are.

``` r
setwd("~/iBio/workshop2020-iBio/S1")
```

## 1\. Importing and formatting data

Start importing counts table and metadata associated to the samples
(previously downloaded from
[Data](https://github.com/ibioChile/Transcriptomics-R-Workshop/tree/master/Session1-Temporal_Analysis/Data)
folder).

``` r
counts <- read.table("fc0.counts_original.txt")
metadata <- read.table("metadata.txt", header=TRUE)
kable(head(metadata))
```

| Sample     | Tissue | Treatment | Replicate | Time |
| :--------- | :----- | :-------- | --------: | ---: |
| SRR5440784 | Shoot  | None      |         1 |    0 |
| SRR5440785 | Shoot  | None      |         2 |    0 |
| SRR5440786 | Shoot  | None      |         3 |    0 |
| SRR5440814 | Shoot  | None      |         1 |    5 |
| SRR5440815 | Shoot  | None      |         1 |   10 |
| SRR5440816 | Shoot  | None      |         1 |   15 |

We will fix the header of counts leaving only the sample ID.

``` r
colnames(counts) <- sapply(strsplit(colnames(counts),".",fixed=TRUE), `[`, 1)
kable(head(counts))
```

|           | SRR5440784 | SRR5440785 | SRR5440786 | SRR5440814 | SRR5440815 | SRR5440816 | SRR5440817 | SRR5440818 | SRR5440819 | SRR5440820 | SRR5440821 | SRR5440822 | SRR5440823 | SRR5440824 | SRR5440825 | SRR5440826 | SRR5440827 | SRR5440828 | SRR5440829 | SRR5440830 | SRR5440831 | SRR5440832 | SRR5440833 | SRR5440834 | SRR5440835 | SRR5440836 | SRR5440837 | SRR5440838 | SRR5440839 | SRR5440840 |
| --------- | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: |
| AT1G01010 |       2074 |       3004 |       2310 |       1033 |        750 |        876 |        553 |        766 |        717 |        838 |        441 |        526 |        671 |        804 |       1181 |        785 |        732 |        524 |        611 |        751 |        562 |        731 |        591 |       1170 |        514 |        762 |        525 |        797 |        397 |        971 |
| AT1G01020 |        750 |       1420 |       1340 |        405 |        444 |        419 |        573 |        477 |        443 |        409 |        642 |        514 |        513 |        390 |        723 |        424 |        333 |        335 |        350 |        674 |        547 |        445 |        402 |        556 |        202 |        438 |        476 |        523 |        399 |        728 |
| AT1G03987 |          2 |          0 |          8 |          0 |          2 |          0 |          1 |          0 |          0 |          0 |          0 |          0 |          0 |          0 |          2 |          2 |          0 |          2 |          0 |          0 |          0 |          0 |          2 |          0 |          0 |          0 |          0 |          0 |          0 |          0 |
| AT1G01030 |        232 |        389 |        352 |         99 |        119 |        108 |         96 |        110 |         74 |        118 |        129 |         88 |        153 |        110 |        154 |         64 |         81 |         62 |         64 |        114 |         97 |        168 |        119 |        153 |         99 |         96 |         72 |         85 |         46 |        122 |
| AT1G01040 |       7811 |      11842 |      10856 |       2798 |       2955 |       2800 |       2766 |       3526 |       3079 |       3955 |       2624 |       2787 |       3336 |       2702 |       4821 |       2695 |       2739 |       2692 |       2473 |       3187 |       2885 |       3427 |       2719 |       4415 |       1932 |       3198 |       2637 |       3589 |       2320 |       4172 |
| AT1G03993 |        629 |       1058 |        970 |        180 |        176 |        156 |         69 |        200 |        198 |        290 |        124 |        154 |        211 |        208 |        345 |        138 |        120 |        223 |        121 |        183 |        188 |        202 |        129 |        279 |        170 |        204 |        128 |        269 |        149 |        365 |

This is a time-series data so, it would be good that samples order
follows the time-series.

``` r
metadata <- metadata %>% arrange(Time)
counts <- counts[,metadata$Sample]
dim(counts)
```

    ## [1] 32833    30

> This table has 30 samples and 32,833 genes.

## 2\. Creating the edgeR object

To create the edgeR object we need the counts. Optionally we can group
samples by a factor. We will group samples by time.

``` r
dgList <- DGEList(counts=counts, genes=rownames(counts), group=metadata$Time)
```

Take a look to the created object. It contains data for $counts,
$samples and $genes.

``` r
dgList
```

    ## An object of class "DGEList"
    ## $counts
    ##           SRR5440784 SRR5440785 SRR5440786 SRR5440814 SRR5440823 SRR5440832
    ## AT1G01010       2074       3004       2310       1033        671        731
    ## AT1G01020        750       1420       1340        405        513        445
    ## AT1G03987          2          0          8          0          0          0
    ## AT1G01030        232        389        352         99        153        168
    ## AT1G01040       7811      11842      10856       2798       3336       3427
    ##           SRR5440815 SRR5440824 SRR5440833 SRR5440816 SRR5440825 SRR5440834
    ## AT1G01010        750        804        591        876       1181       1170
    ## AT1G01020        444        390        402        419        723        556
    ## AT1G03987          2          0          2          0          2          0
    ## AT1G01030        119        110        119        108        154        153
    ## AT1G01040       2955       2702       2719       2800       4821       4415
    ##           SRR5440817 SRR5440826 SRR5440835 SRR5440818 SRR5440827 SRR5440836
    ## AT1G01010        553        785        514        766        732        762
    ## AT1G01020        573        424        202        477        333        438
    ## AT1G03987          1          2          0          0          0          0
    ## AT1G01030         96         64         99        110         81         96
    ## AT1G01040       2766       2695       1932       3526       2739       3198
    ##           SRR5440819 SRR5440828 SRR5440837 SRR5440820 SRR5440829 SRR5440838
    ## AT1G01010        717        524        525        838        611        797
    ## AT1G01020        443        335        476        409        350        523
    ## AT1G03987          0          2          0          0          0          0
    ## AT1G01030         74         62         72        118         64         85
    ## AT1G01040       3079       2692       2637       3955       2473       3589
    ##           SRR5440821 SRR5440830 SRR5440839 SRR5440822 SRR5440831 SRR5440840
    ## AT1G01010        441        751        397        526        562        971
    ## AT1G01020        642        674        399        514        547        728
    ## AT1G03987          0          0          0          0          0          0
    ## AT1G01030        129        114         46         88         97        122
    ## AT1G01040       2624       3187       2320       2787       2885       4172
    ## 32828 more rows ...
    ## 
    ## $samples
    ##            group  lib.size norm.factors
    ## SRR5440784     0  80084739            1
    ## SRR5440785     0 118358949            1
    ## SRR5440786     0 120487357            1
    ## SRR5440814     5  33821200            1
    ## SRR5440823     5  36535298            1
    ## 25 more rows ...
    ## 
    ## $genes
    ##               genes
    ## AT1G01010 AT1G01010
    ## AT1G01020 AT1G01020
    ## AT1G03987 AT1G03987
    ## AT1G01030 AT1G01030
    ## AT1G01040 AT1G01040
    ## 32828 more rows ...

A more detailed view on $samples will show you that they are grouped, as
expected.

``` r
kable(dgList$samples)
```

|            | group |  lib.size | norm.factors |
| ---------- | :---- | --------: | -----------: |
| SRR5440784 | 0     |  80084739 |            1 |
| SRR5440785 | 0     | 118358949 |            1 |
| SRR5440786 | 0     | 120487357 |            1 |
| SRR5440814 | 5     |  33821200 |            1 |
| SRR5440823 | 5     |  36535298 |            1 |
| SRR5440832 | 5     |  34940880 |            1 |
| SRR5440815 | 10    |  27438300 |            1 |
| SRR5440824 | 10    |  27493782 |            1 |
| SRR5440833 | 10    |  27354641 |            1 |
| SRR5440816 | 15    |  32378605 |            1 |
| SRR5440825 | 15    |  48431969 |            1 |
| SRR5440834 | 15    |  43822059 |            1 |
| SRR5440817 | 20    |  28444049 |            1 |
| SRR5440826 | 20    |  29809654 |            1 |
| SRR5440835 | 20    |  19823796 |            1 |
| SRR5440818 | 30    |  35934994 |            1 |
| SRR5440827 | 30    |  29025019 |            1 |
| SRR5440836 | 30    |  33060473 |            1 |
| SRR5440819 | 45    |  32357400 |            1 |
| SRR5440828 | 45    |  28803027 |            1 |
| SRR5440837 | 45    |  34570401 |            1 |
| SRR5440820 | 60    |  46978269 |            1 |
| SRR5440829 | 60    |  35386483 |            1 |
| SRR5440838 | 60    |  42840822 |            1 |
| SRR5440821 | 90    |  39266602 |            1 |
| SRR5440830 | 90    |  45466861 |            1 |
| SRR5440839 | 90    |  29298806 |            1 |
| SRR5440822 | 120   |  40632550 |            1 |
| SRR5440831 | 120   |  35091302 |            1 |
| SRR5440840 | 120   |  53612258 |            1 |

``` r
kable(head(dgList$counts)) # segment of the table
```

|           | SRR5440784 | SRR5440785 | SRR5440786 | SRR5440814 | SRR5440823 | SRR5440832 | SRR5440815 | SRR5440824 | SRR5440833 | SRR5440816 | SRR5440825 | SRR5440834 | SRR5440817 | SRR5440826 | SRR5440835 | SRR5440818 | SRR5440827 | SRR5440836 | SRR5440819 | SRR5440828 | SRR5440837 | SRR5440820 | SRR5440829 | SRR5440838 | SRR5440821 | SRR5440830 | SRR5440839 | SRR5440822 | SRR5440831 | SRR5440840 |
| --------- | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: | ---------: |
| AT1G01010 |       2074 |       3004 |       2310 |       1033 |        671 |        731 |        750 |        804 |        591 |        876 |       1181 |       1170 |        553 |        785 |        514 |        766 |        732 |        762 |        717 |        524 |        525 |        838 |        611 |        797 |        441 |        751 |        397 |        526 |        562 |        971 |
| AT1G01020 |        750 |       1420 |       1340 |        405 |        513 |        445 |        444 |        390 |        402 |        419 |        723 |        556 |        573 |        424 |        202 |        477 |        333 |        438 |        443 |        335 |        476 |        409 |        350 |        523 |        642 |        674 |        399 |        514 |        547 |        728 |
| AT1G03987 |          2 |          0 |          8 |          0 |          0 |          0 |          2 |          0 |          2 |          0 |          2 |          0 |          1 |          2 |          0 |          0 |          0 |          0 |          0 |          2 |          0 |          0 |          0 |          0 |          0 |          0 |          0 |          0 |          0 |          0 |
| AT1G01030 |        232 |        389 |        352 |         99 |        153 |        168 |        119 |        110 |        119 |        108 |        154 |        153 |         96 |         64 |         99 |        110 |         81 |         96 |         74 |         62 |         72 |        118 |         64 |         85 |        129 |        114 |         46 |         88 |         97 |        122 |
| AT1G01040 |       7811 |      11842 |      10856 |       2798 |       3336 |       3427 |       2955 |       2702 |       2719 |       2800 |       4821 |       4415 |       2766 |       2695 |       1932 |       3526 |       2739 |       3198 |       3079 |       2692 |       2637 |       3955 |       2473 |       3589 |       2624 |       3187 |       2320 |       2787 |       2885 |       4172 |
| AT1G03993 |        629 |       1058 |        970 |        180 |        211 |        202 |        176 |        208 |        129 |        156 |        345 |        279 |         69 |        138 |        170 |        200 |        120 |        204 |        198 |        223 |        128 |        290 |        121 |        269 |        124 |        183 |        149 |        154 |        188 |        365 |

We can add metadata to the edgeR object to use it directly

``` r
dgList$metadata <- metadata
```

Now, take a look to the object and you will see a new variable called
$metadata at the end

``` r
dgList
```

    ## An object of class "DGEList"
    ## $counts
    ##           SRR5440784 SRR5440785 SRR5440786 SRR5440814 SRR5440823 SRR5440832
    ## AT1G01010       2074       3004       2310       1033        671        731
    ## AT1G01020        750       1420       1340        405        513        445
    ## AT1G03987          2          0          8          0          0          0
    ## AT1G01030        232        389        352         99        153        168
    ## AT1G01040       7811      11842      10856       2798       3336       3427
    ##           SRR5440815 SRR5440824 SRR5440833 SRR5440816 SRR5440825 SRR5440834
    ## AT1G01010        750        804        591        876       1181       1170
    ## AT1G01020        444        390        402        419        723        556
    ## AT1G03987          2          0          2          0          2          0
    ## AT1G01030        119        110        119        108        154        153
    ## AT1G01040       2955       2702       2719       2800       4821       4415
    ##           SRR5440817 SRR5440826 SRR5440835 SRR5440818 SRR5440827 SRR5440836
    ## AT1G01010        553        785        514        766        732        762
    ## AT1G01020        573        424        202        477        333        438
    ## AT1G03987          1          2          0          0          0          0
    ## AT1G01030         96         64         99        110         81         96
    ## AT1G01040       2766       2695       1932       3526       2739       3198
    ##           SRR5440819 SRR5440828 SRR5440837 SRR5440820 SRR5440829 SRR5440838
    ## AT1G01010        717        524        525        838        611        797
    ## AT1G01020        443        335        476        409        350        523
    ## AT1G03987          0          2          0          0          0          0
    ## AT1G01030         74         62         72        118         64         85
    ## AT1G01040       3079       2692       2637       3955       2473       3589
    ##           SRR5440821 SRR5440830 SRR5440839 SRR5440822 SRR5440831 SRR5440840
    ## AT1G01010        441        751        397        526        562        971
    ## AT1G01020        642        674        399        514        547        728
    ## AT1G03987          0          0          0          0          0          0
    ## AT1G01030        129        114         46         88         97        122
    ## AT1G01040       2624       3187       2320       2787       2885       4172
    ## 32828 more rows ...
    ## 
    ## $samples
    ##            group  lib.size norm.factors
    ## SRR5440784     0  80084739            1
    ## SRR5440785     0 118358949            1
    ## SRR5440786     0 120487357            1
    ## SRR5440814     5  33821200            1
    ## SRR5440823     5  36535298            1
    ## 25 more rows ...
    ## 
    ## $genes
    ##               genes
    ## AT1G01010 AT1G01010
    ## AT1G01020 AT1G01020
    ## AT1G03987 AT1G03987
    ## AT1G01030 AT1G01030
    ## AT1G01040 AT1G01040
    ## 32828 more rows ...
    ## 
    ## $metadata
    ##       Sample Tissue Treatment Replicate Time
    ## 1 SRR5440784  Shoot      None         1    0
    ## 2 SRR5440785  Shoot      None         2    0
    ## 3 SRR5440786  Shoot      None         3    0
    ## 4 SRR5440814  Shoot      None         1    5
    ## 5 SRR5440823  Shoot      None         2    5
    ## 25 more rows ...

Samples names are not informative. Let’s transform them to a more
informative form.

Set a vector with the new names and apply them to our data

``` r
ColRename <- paste(paste0(rep("t",30),dgList$metadata[,"Time"]),paste0(rep("r",30),dgList$metadata[,"Replicate"]),sep=".")
colnames(dgList) <- ColRename
```

Now, review your new names

``` r
dgList$samples
```

    ##         group  lib.size norm.factors
    ## t0.r1       0  80084739            1
    ## t0.r2       0 118358949            1
    ## t0.r3       0 120487357            1
    ## t5.r1       5  33821200            1
    ## t5.r2       5  36535298            1
    ## t5.r3       5  34940880            1
    ## t10.r1     10  27438300            1
    ## t10.r2     10  27493782            1
    ## t10.r3     10  27354641            1
    ## t15.r1     15  32378605            1
    ## t15.r2     15  48431969            1
    ## t15.r3     15  43822059            1
    ## t20.r1     20  28444049            1
    ## t20.r2     20  29809654            1
    ## t20.r3     20  19823796            1
    ## t30.r1     30  35934994            1
    ## t30.r2     30  29025019            1
    ## t30.r3     30  33060473            1
    ## t45.r1     45  32357400            1
    ## t45.r2     45  28803027            1
    ## t45.r3     45  34570401            1
    ## t60.r1     60  46978269            1
    ## t60.r2     60  35386483            1
    ## t60.r3     60  42840822            1
    ## t90.r1     90  39266602            1
    ## t90.r2     90  45466861            1
    ## t90.r3     90  29298806            1
    ## t120.r1   120  40632550            1
    ## t120.r2   120  35091302            1
    ## t120.r3   120  53612258            1

## 3\. Data normalization

During the sample preparation or sequencing process, external factors
that are not of biological interest can affect the expression of
individual samples. For example, samples processed in the first batch of
an experiment can have higher expression overall when compared to
samples processed in a second batch. It is assumed that all samples
should have a similar range and distribution of expression values.
Normalisation is required to ensure that the expression distributions of
each sample are similar across the entire experiment.

Any plot showing the per sample expression distributions, such as a
density or boxplot, is useful in determining whether any samples are
dissimilar to others. Distributions of log-CPM values are similar
throughout all samples within this dataset.

Nonetheless, normalisation by the method of trimmed mean of M-values
(TMM) (Robinson and Oshlack 2010) is performed using the calcNormFactors
function in edgeR (but you could use others like upperquartile. The
normalisation \# factors calculated here are used as a scaling factor
for the library sizes. When working with DGEList-objects, these
normalisation factors are automatically stored in
x\(samples\)norm.factors. For this dataset the effect of
TMM-normalisation is mild, as evident in the magnitude of the scaling
factors, which are all relatively close to 1.

More info on this command:

``` r
?calcNormFactors
```

We will create a new edgeR object with the normalization

``` r
dgList2 <- calcNormFactors(dgList, method="TMM")
```

Now let’s compare the samples on both variables

``` r
head(dgList$samples)
```

    ##       group  lib.size norm.factors
    ## t0.r1     0  80084739            1
    ## t0.r2     0 118358949            1
    ## t0.r3     0 120487357            1
    ## t5.r1     5  33821200            1
    ## t5.r2     5  36535298            1
    ## t5.r3     5  34940880            1

``` r
head(dgList2$samples)
```

    ##       group  lib.size norm.factors
    ## t0.r1     0  80084739    0.9777590
    ## t0.r2     0 118358949    0.9706289
    ## t0.r3     0 120487357    0.9066125
    ## t5.r1     5  33821200    1.0849764
    ## t5.r2     5  36535298    1.1103770
    ## t5.r3     5  34940880    1.0810259

The column $norm.factors is different. As expected, the calcNormFactors
function fill those values according to the chosen method.

``` r
dgList$samples$norm.factors
```

    ##  [1] 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1

``` r
dgList2$samples$norm.factors
```

    ##  [1] 0.9777590 0.9706289 0.9066125 1.0849764 1.1103770 1.0810259 1.1250347
    ##  [8] 1.0799020 1.1121500 1.0604866 1.0958632 1.0663918 1.1381982 1.0782132
    ## [15] 1.0995999 1.1039724 1.0148196 1.1314464 1.0879510 0.9937129 0.9352691
    ## [22] 0.8714842 0.8916032 0.9437251 0.9146101 0.8884828 0.8827364 0.7870135
    ## [29] 0.9063070 0.8212572

Let’s see the normalization results with a boxplot

> To ensure a good visualization I will use a data transformation called
> “cpm” from “counts per million” and log2. This will be profundized on
> the next section.

``` r
par(mfrow=c(1,2))
# plot of unnormalised data
cldgList <- cpm(dgList, log=TRUE, prior.count = 1)
boxplot(cldgList, las=2, col=as.vector(dgList$metadata$Color), main="")
title(main="A. Unnormalised data",ylab="Log2-cpm")

# plot of normalised data
cldgList2 <- cpm(dgList2, log=TRUE, prior.count = 1)
boxplot(cldgList2, las=2, col=as.vector(dgList$metadata$Color), main="")
title(main="B. Normalised data",ylab="Log2-cpm")
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig1-1.png" style="display: block; margin: auto;" />

Did you see the difference? (hint: compare the median and the upper
quartile). From now on, we will work with normalized data.

## 4\. Data transformation for visualization

Count data has a has a broad spectrum of distribution, from genes with
zero count to genes with thousands.

Plot the following boxplots:

``` r
Color <- c("#FFC900", "#FFB900","#FFA800", "#FF9200", "#FF7700","#FF5D00","#FF4300","#FF2900","#FF0000","#E40000")[as.factor(metadata$Time)]

par(mfrow=c(1,4)) #Figure2
boxplot(dgList$counts, las=2, col=as.matrix(Color), main="")
title(main="A. Untransformed data",ylab="original data")

Log2 <- log2(dgList2$counts)
Log2_pseudoCounts <- log2(dgList2$counts+1)
cpmCounts <- cpm(dgList2, log=TRUE, prior.count = 1)

boxplot(Log2, las=2, col=as.matrix(Color), main="")
title(main="B. Log2 transformation",ylab="Log2")

boxplot(Log2_pseudoCounts, las=2, col=as.matrix(Color), main="")
title(main="C. Log2(x+1) (pseudoCounts)",ylab="pLog2")

boxplot(cpmCounts, las=2, col=as.matrix(Color), main="")
title(main="D. cpm with log2 and 0=1",ylab="pLog2(cpm)")
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig2-1.png" style="display: block; margin: auto;" />

As you can see on panel A, the most datapoints are sticked to the x bar
due that the majority of genes having low count values. A way to “see
all the datapoints” is to apply a data transformation. In panel B, a
log2 transformation was applied. However, since log2 function can’t deal
with “zero” counts, those samples are lost in the graph. This is not a
problem in a boxplot representation but could be a problem for scatter
plots or density plots.

A trick to avoid this issue is to add 1 count to all samples. This is
known as “pseudocount” transformation, which is applied in panel C.

Panel D shows cpm transformation, which is another kind of data
manipulation, where data is divided by the total number of counts and
normalized by 1 million. cpm(x) = (x/sum(x))\*1000000. Then, “only
zeros” are transformed to 1 (0=1) and log2 transformation can be
applied.

We can check the distribution of a specific sample using the *hist*
function;

``` r
par(mfrow=c(1,4))
hist(dgList2$counts[,"t0.r2"],col="lightblue", main="A. Untransf data")
hist(Log2[,"t0.r2"],col="lightblue", main="B. Log2 transf")
hist(Log2_pseudoCounts[,"t0.r2"],col="lightblue", main="C. Log2_pseudoCounts")
hist(cpmCounts[,"t0.r2"],col="lightblue", main="D. cpm+log2+pseudocounts")
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig3-1.png" style="display: block; margin: auto;" />

Take a look at the peak at zero, which is lost on the Log2 graph.

## 5\. Gene filtering

How many genes are not expressed over all samples?

``` r
kable(table(rowSums(dgList2$counts==0)==30))
```

| Var1  |  Freq |
| :---- | ----: |
| FALSE | 29768 |
| TRUE  |  3065 |

There are 32,833 genes in this dataset. However, many of them are not
expressed or are not represented by enough reads to contribute to the
analysis. Removing those genes means that we ultimately have fewer tests
to perform, thereby reducing problems associated with multiple testing.

Here, we present two ways to do perform counts filtering.

### 5.1. Manual cutoff

We will retain only those genes that are represented by at least 1 cpm
in at least two samples.

Calculate the cpm

``` r
countsPerMillion <- cpm(dgList2)
```

Let’s compare pre and post cpm for the first sample

``` r
summary(dgList2$counts[,1])
```

    ##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
    ##       0       6     357    2439    1978 3198498

``` r
summary(countsPerMillion[,1]) #first sample (t0.r1)
```

    ##     Min.  1st Qu.   Median     Mean  3rd Qu.     Max. 
    ##     0.00     0.08     4.56    31.15    25.26 40847.41

What’s the difference?… Let’s found what cells of the matrix have a cpm
count above 1

``` r
countCheck <- countsPerMillion > 1
kable(head(countCheck))
```

|           | t0.r1 | t0.r2 | t0.r3 | t5.r1 | t5.r2 | t5.r3 | t10.r1 | t10.r2 | t10.r3 | t15.r1 | t15.r2 | t15.r3 | t20.r1 | t20.r2 | t20.r3 | t30.r1 | t30.r2 | t30.r3 | t45.r1 | t45.r2 | t45.r3 | t60.r1 | t60.r2 | t60.r3 | t90.r1 | t90.r2 | t90.r3 | t120.r1 | t120.r2 | t120.r3 |
| --------- | :---- | :---- | :---- | :---- | :---- | :---- | :----- | :----- | :----- | :----- | :----- | :----- | :----- | :----- | :----- | :----- | :----- | :----- | :----- | :----- | :----- | :----- | :----- | :----- | :----- | :----- | :----- | :------ | :------ | :------ |
| AT1G01010 | TRUE  | TRUE  | TRUE  | TRUE  | TRUE  | TRUE  | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE    | TRUE    | TRUE    |
| AT1G01020 | TRUE  | TRUE  | TRUE  | TRUE  | TRUE  | TRUE  | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE    | TRUE    | TRUE    |
| AT1G03987 | FALSE | FALSE | FALSE | FALSE | FALSE | FALSE | FALSE  | FALSE  | FALSE  | FALSE  | FALSE  | FALSE  | FALSE  | FALSE  | FALSE  | FALSE  | FALSE  | FALSE  | FALSE  | FALSE  | FALSE  | FALSE  | FALSE  | FALSE  | FALSE  | FALSE  | FALSE  | FALSE   | FALSE   | FALSE   |
| AT1G01030 | TRUE  | TRUE  | TRUE  | TRUE  | TRUE  | TRUE  | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE    | TRUE    | TRUE    |
| AT1G01040 | TRUE  | TRUE  | TRUE  | TRUE  | TRUE  | TRUE  | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE    | TRUE    | TRUE    |
| AT1G03993 | TRUE  | TRUE  | TRUE  | TRUE  | TRUE  | TRUE  | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE   | TRUE    | TRUE    | TRUE    |

Now, sum the “TRUE” values across rows to keep only the ones where at
least 2 cells have 1 cpm value

``` r
keep <- which(rowSums(countCheck) >= 2) # over 1 cpm
str(keep)
```

    ##  Named int [1:21960] 1 2 4 5 6 7 8 10 11 13 ...
    ##  - attr(*, "names")= chr [1:21960] "AT1G01010" "AT1G01020" "AT1G01030" "AT1G01040" ...

There are 21,960 genes that meet our requeriments

Apply the filter to the normalized edgeR table and save the result to a
new edgeR object

``` r
dgList3 <- dgList2[keep,, keep.lib.sizes=FALSE]
```

Compare the dimensions of original and filtered edgeR data:

``` r
dim(dgList2)
```

    ## [1] 32833    30

``` r
dim(dgList3)
```

    ## [1] 21960    30

Now, let’s compare all samples pre and post filtering with a density
plot. The following lines produce a density of log-CPM values for (A)
raw pre-filtered data and (B) post-filtered data for each sample.

Dotted vertical lines mark the log-CPM threshold that we use (cpm \>1).
Samples are colored by time as it is represented on the legend.

``` r
nsamples <- ncol(dgList)
lcpm.cutoff <- log2(1)
```

``` r
par(mfrow=c(1,2))

z <- cpm(dgList, log=TRUE, prior.count=1)
plot(density(z[,1]), col=Color, lwd=2, ylim=c(0,0.26), las=2, main="", xlab="")
title(main="A. Raw data", xlab="Log-cpm")
abline(v=lcpm.cutoff, lty=3)
for (i in 2:nsamples){
  den <- density(z[,i])
  lines(den$x, den$y, col=Color[i], lwd=2)
}
legend("topright", title="time", unique(as.character(dgList$metadata$Time)), text.col=unique(Color), bty="n")

z <- cpm(dgList3, log=TRUE, prior.count=1)
plot(density(z[,1]), col=Color, lwd=2, ylim=c(0,0.26), las=2, main="", xlab="")
title(main="B. Filtered data", xlab="Log-cpm")
abline(v=lcpm.cutoff, lty=3)
for (i in 2:nsamples){
  den <- density(z[,i])
  lines(den$x, den$y, col=Color[i], lwd=2)
}
legend("topright", title="time", unique(as.character(dgList$metadata$Time)), text.col=unique(Color), bty="n")
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig4-1.png" style="display: block; margin: auto;" />

### 5.2. EdgeR cutoff

The filterByExpr function of the edgeR package provides an automatic way
to filter genes while keeping as many genes as possible with worthwhile
counts. By default, the function keeps genes with about 10 read counts
or more in a minimum number of samples, where the number of samples is
chosen according to the minimum group sample size.

More information of the function:

``` r
?filterByExpr
```

Let’s apply the filter gruping samples by time:

``` r
keep.exprs <- filterByExpr(dgList2, group=dgList2$metadata$Time)
dgList4 <- dgList2[keep.exprs,, keep.lib.sizes=FALSE]
```

Compare the dimensions of original, manual filtered and edgeR filtered
data:

``` r
dim(dgList2)
```

    ## [1] 32833    30

``` r
dim(dgList3)
```

    ## [1] 21960    30

``` r
dim(dgList4)
```

    ## [1] 24007    30

Now, let’s compare all samples pre and post filtering with a density
plot. The following lines produce a density of log-CPM values for (A)
raw pre-filtered data and (B) post-filtered data for each sample. Dotted
vertical lines mark the log-CPM threshold (equivalent to a CPM value of
about 0.3) used in the filtering step. The actual filtering uses CPM
values rather than counts in order to avoid giving preference to samples
with large library sizes. For this dataset, the median library size is
about 35 million and 10/35 approx. 0.3, so the filterByExpr function
keeps genes that have a CPM of 0.3 or more in at least three samples.

``` r
L <- mean(dgList$samples$lib.size) * 1e-6 # = 42 million reads
M <- median(dgList$samples$lib.size) * 1e-6 # = 35 million reads
lcpm.cutoff <- log2(10/M + 2/L) # cutoff used by filterByExpr function
```

``` r
par(mfrow=c(1,2)) 

z <- cpm(dgList, log=TRUE, prior.count=1)
plot(density(z[,1]), col=Color, lwd=2, ylim=c(0,0.26), las=2, main="", xlab="")
title(main="A. Raw data", xlab="Log-cpm")
abline(v=lcpm.cutoff, lty=3)
for (i in 2:nsamples){
  den <- density(z[,i])
  lines(den$x, den$y, col=Color[i], lwd=2)
}
legend("topright", title="time", unique(as.character(dgList$metadata$Time)), text.col=unique(Color), bty="n")

z <- cpm(dgList4, log=TRUE, prior.count=1)
plot(density(z[,1]), col=Color, lwd=2, ylim=c(0,0.26), las=2, main="", xlab="")
title(main="B. Filtered data", xlab="Log-cpm")
abline(v=lcpm.cutoff, lty=3)
for (i in 2:nsamples){
  den <- density(z[,i])
  lines(den$x, den$y, col=Color[i], lwd=2)
}
legend("topright", title="time", unique(as.character(dgList$metadata$Time)), text.col=unique(Color), bty="n")
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig5-1.png" style="display: block; margin: auto;" />

## 6\. Data Exploration

### 6.1. edgeR MDS

We can examine inter-sample relationships by producing a plot based on
multidimensional scaling. More details of this kind of exploration on
Session4… this is just an example. When an object of type DGEList is the
input of plotMDS function, the real called function is a modified
version designed by edgeR team with real name “plotMDS.DGEList”. It
convert the counts to log-cpm and pass these to the limma plotMDS
function. There, distance between each pair of samples (columns) is the
root-mean-square deviation (Euclidean distance) for the top genes.

Distances on the plot can be interpreted as leading log2-fold-change,
meaning the typical (root-mean-square) log2-fold-change between the
samples for the genes that distinguish those samples.

More information about this functions:

``` r
?plotMDS.DGEList
?plotMDS
```

``` r
par(mfrow=c(1,2))  #Figure6
dgMDS <- plotMDS(dgList3, prior.count=1, xlab = "dim1", ylab="dim2", plot = TRUE)
title(main="A. MDS plot")
plotMDS(dgList3, prior.count=1, col=Color, labels=dgList$metadata$Time, xlab = "dim1", ylab="dim2")
title(main="B. Setting colors and labels")
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig6-1.png" style="display: block; margin: auto;" />

On panel A you see the samples ordination in MDS space. Did you see time
groups?

“Labels” in panel B comes from “Time” column of metadata table  
“Colors” in panel B comes from “Color” column of metadata table

### 6.2. Clustering analysis

Another common data exploration technique used is clustering analysis. A
distance is calculated between samples based on some combination of
variable and a distance matrix generated. A clustering algorithm is then
run over the distance matrix that aggregates/groups samples based on
their distance. There are several clustering methods and several
clustering packages available in R. Of particular use in this context
are the heirarchical clustering functions hclust() available as a base R
function and pvclust() available in the pvclust package (Suzuki and
Shimodaira, 2006). The advantage with the pvclust() function is it uses
bootstrapping to provide a level of confidence that the observed
clusters are robust to slight changes in the samples.

#### 6.2.1. Clustering with hclust() function

More details here:
<https://2-bitbio.com/2017/04/clustering-rnaseq-data-making-heatmaps.html>

First, data as cpm:

``` r
cpmCounts <- cpm(dgList3, log=TRUE, prior.count=1)
cpmCounts <- cpmCounts[complete.cases(cpmCounts),] # no missing values
```

Samples clustering (columns of the matrix)

``` r
hc <- hclust(as.dist(1-cor(cpmCounts, method="pearson")), method="average")
```

Let’s plot the sample’s dendogram

``` r
TreeC = as.dendrogram(hc, method="average")
plot(TreeC,
     main = "Sample's Clustering",
     ylab = "Height")
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig7-1.png" style="display: block; margin: auto;" />

We can select a tree “level” to obtain clusters. For example, at eight
0.06 we will obtain 3 clusters

``` r
TreeC = as.dendrogram(hc, method="average")
plot(TreeC,
     main = "Sample's Clustering",
     ylab = "Height")
abline(h=0.04, lwd=2, col="pink") # 0.05=2groups, 0.04=3groups, 0.03=4groups
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig8-1.png" style="display: block; margin: auto;" />

This is the commmand to cut the tree and save clusters

``` r
group <- as.factor(cutree(hc, h=0.04)) # 0.05=2groups, 0.04=3groups, 0.03=4groups
```

Number of clusters

``` r
levels(group)
```

    ## [1] "1" "2" "3"

And this clustering information could be used to enrich PCA plots (next
Sesion)

We have clusters, but without statistical confidence we can’t do a
credible observation.

#### 6.2.2. Clustering with pvclust() function

This method calculates p-values for hierarchical clustering via
multiscale bootstrap resampling. Hierarchical clustering is done for
given data and p-values are computed for each of the clusters.

More info about pvclust:

``` r
?pvclust
```

On this variable we will set the bootstrap value. You can set it to 10
for a quick result, but change it to 100 or even 1000 for the final
figure (and meanwhile go for a coffee cup).

``` r
bootstrap <- 99
pv <- pvclust(cpm(dgList3, log=TRUE, prior.count=1), method.dist="cor", method.hclust="average", nboot=bootstrap)
```

    ## Bootstrap (r = 0.5)... Done.
    ## Bootstrap (r = 0.6)... Done.
    ## Bootstrap (r = 0.7)... Done.
    ## Bootstrap (r = 0.8)... Done.
    ## Bootstrap (r = 0.9)... Done.
    ## Bootstrap (r = 1.0)... Done.
    ## Bootstrap (r = 1.1)... Done.
    ## Bootstrap (r = 1.2)... Done.
    ## Bootstrap (r = 1.3)... Done.
    ## Bootstrap (r = 1.4)... Done.

This variable is to set the number’s colors of the plot

``` r
pvcolors <- c(si=4,au=2,bp=4,edge=8)
```

The cluster plot:

``` r
plot(pv,print.num=FALSE, col.pv=pvcolors, cex.pv=0.8, main=paste("Cluster dendogram with 'Approximately Unbiased' values (%), bootstrap=", bootstrap))
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig9-1.png" style="display: block; margin: auto;" />

The cluster plot selecting groups according to a specific alpha value

``` r
plot(pv,print.num=FALSE, col.pv=pvcolors, cex.pv=0.8, main=paste("Cluster dendogram with 'Approximately Unbiased' values (%), bootstrap=", bootstrap))
pvrect(pv, alpha=0.90, pv="au", type="geq", max.only=TRUE)
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig10-1.png" style="display: block; margin: auto;" />

Graph explanation: Two types of values are provided for the confidence
of the nodes: AU (Approximately Unbiased) values and BP (Bootstrap
Probability) values. In both cases, higher is more confident so not to
be confused with conventional p-values. The AU value is computed by
multiscale bootstrap resampling and is a better approximation to
unbiased p-value than BP value computed by normal bootstrap resampling.
Red values are AU p-values, and blue values are BP values. Clusters with
AU larger than 90% are highlighted by rectangles, which are strongly
supported by data.

With the following command we can select the “significant” clusters and
use them in further analysis.

``` r
group <- pvpick(pv, alpha=0.90, pv="au", type="geq", max.only=TRUE)
group$clusters
```

    ## [[1]]
    ##  [1] "t0.r1"  "t0.r2"  "t0.r3"  "t5.r1"  "t5.r2"  "t5.r3"  "t10.r1" "t10.r2"
    ##  [9] "t10.r3" "t15.r1" "t15.r2" "t15.r3" "t20.r1" "t20.r2" "t30.r2"
    ## 
    ## [[2]]
    ##  [1] "t20.r3"  "t30.r1"  "t30.r3"  "t45.r1"  "t45.r2"  "t45.r3"  "t60.r1" 
    ##  [8] "t60.r2"  "t60.r3"  "t90.r1"  "t90.r2"  "t90.r3"  "t120.r1" "t120.r2"
    ## [15] "t120.r3"

#### 6.2.3. Model based clustering

The traditional clustering methods, such as hierarchical clustering and
k-means clustering, are heuristic and are not based on formal models.
Furthermore, k-means algorithm is commonly randomnly initialized, so
different runs of k-means will often yield different results.
Additionally, k-means requires the user to specify the the optimal
number of clusters.

Clustering algorithms can also be developed based on probability models,
such as the finite mixture model for probability densities. The word
model is usually used to represent the type of constraints and geometric
properties of the covariance matrices (Martinez and Martinez, 2005). In
the family of model-based clustering algorithms, one uses certain models
for clusters and tries to optimize the fit between the data and the
models. In the model-based clustering approach, the data are viewed as
coming from a mixture of probability distributions, each of which
represents a different cluster. In other words, in model-based
clustering, it is assumed that the data are generated by a mixture of
probability distributions in which each component represents a different
cluster. Thus a particular clustering method can be expected to work
well when the data conform to the model. Read More:
<https://epubs.siam.org/doi/abs/10.1137/1.9780898718348.ch14?mobileUi=0&>
<https://www.datanovia.com/en/lessons/model-based-clustering-essentials/>

Here we choose a model-based clustering with Mclust of the package
mclust.

``` r
?mclust
```

This method needs a lot of resources so, the recommendation is to use a
subset of the data. For instance, we could choose a random subset or
choose genes withthe highest expression variability over samples.

Use this variable to set the subset length

``` r
numberRows <- 1000
```

Option1: Random selection of “numberRows” genes

``` r
cpmCounts_subset.random <- cpmCounts[sample(nrow(cpmCounts), numberRows), ]
```

Option2: Selection of genes with the highest expression variation. We
will use the function rowSds to select the top “numberRows” genes.

``` r
orderPositions <- order(rowSds(cpmCounts), decreasing=T)
cpmCounts_subset.Sds <- cpmCounts[orderPositions,][1:numberRows,]
```

> Try both options and compare results)

Now, let’s fit the data to a model using Mclust function

``` r
fit <- Mclust(t(cpmCounts_subset.random), scale.=FALSE)
```

view solution summary… what’s the used model? what it means?

``` r
fit
```

    ## 'Mclust' model object: (EEI,3) 
    ## 
    ## Available components: 
    ##  [1] "call"           "data"           "modelName"      "n"             
    ##  [5] "d"              "G"              "BIC"            "loglik"        
    ##  [9] "df"             "bic"            "icl"            "hypvol"        
    ## [13] "parameters"     "z"              "classification" "uncertainty"

Information about models:

``` r
?mclustModelNames
```

Lookup all the attempted options

``` r
summary(fit$BIC) 
```

    ## Best BIC values:
    ##              EEI,3        VEI,3        VEI,5
    ## BIC      -11949.91 -11964.03851 -11971.32663
    ## BIC diff      0.00    -14.13279    -21.42092

Models and their score by component

``` r
plot(fit$BIC) 
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig11-1.png" style="display: block; margin: auto;" />

Classification vector that we could use on other plots, like PCA (next
Sesion)

``` r
group = as.factor(fit$classification) # classification vector
```

Number of clusters

``` r
levels(group)
```

    ## [1] "1" "2" "3"

### 6.3. Heatmap of samples distance using mixOmics package

We can examine inter-sample relationships by producing heatmap of
samples distance.

There are several functions to do a Heatmaps in R. Here we will use cim
function that comes with the package mixOmics.

First, we need to calculate the distance between samples. The function
“dist” could do this job using different methods, but let’s start with
a correlation distance over pseudocounts.

``` r
Log2_pseudoCounts <- log2(dgList3$counts+1)
sampleDists <- as.matrix(dist(t(Log2_pseudoCounts)), method = "cor")
```

Setting a color pallete over red color:

``` r
cimColor <- colorRampPalette(rev(brewer.pal(9, "Reds")))(20)
```

``` r
cim(sampleDists, color = cimColor, symkey = FALSE, row.cex = 1.3, col.cex = 1.3)
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig12-1.png" style="display: block; margin: auto;" />

What if we use cpm instead of pseudocount?

``` r
cpmCounts <- cpm(dgList3, log=TRUE, prior.count = 1)
sampleDists <- as.matrix(dist(t(cpmCounts)), method = "cor")
```

``` r
cim(sampleDists, color = cimColor, symkey = FALSE, row.cex = 1.3, col.cex = 1.3)
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig13-1.png" style="display: block; margin: auto;" />

Note that sample clustering is different when using pseudocounts or cpm.
The distance between samples is sensible to the kind of data that you
use and pseudocounts are different than cpm, as you see on the panel C
and D of the first graph of data transformation section. Some people do
prefer to use pseudocounts but the correct way is to use cpm
(<https://www.biostars.org/p/165619/>).

## 7\. Lineal model

Linear modelling in edgeR is carried out using the glmFit (GLM) and
contrasts.fit functions originally written for application to
microarrays. The functions can be used for both microarray and RNA-seq
data and fit a separate model to the expression values for each gene.
Next, empirical Bayes moderation is carried out by borrowing information
across all the genes to obtain more precise estimates of gene-wise
variability (Smyth 2004).

Here, we fit a GLM to account for the time effect.

### 7.1. Creating the model

In our analysis, linear models are fitted to the data with the
assumption that the underlying data is normally distributed. To get
started, a design matrix is set up with both time series information.
This matrix describes the setup of the experiment.

Our dataset have have 30 samples that belongs to 10 “times” with
triplicates, we will use “Time” as the comparison factor.

``` r
design.matrix <- model.matrix(~ dgList3$samples$group)
length(design.matrix[1,]) #10 factors
```

    ## [1] 10

Number of factors

``` r
length(design.matrix[1,]) #10 factors
```

    ## [1] 10

Detail of factors and assigned samples: 30 rows for 30 samples. You can
see a number “1” on the corresponding replicates of each sample. Note
that **dgList3$samples$group0** is not present. As a time series, all
factors will be compared against time0.

``` r
design.matrix # factors
```

    ##    (Intercept) dgList3$samples$group5 dgList3$samples$group10
    ## 1            1                      0                       0
    ## 2            1                      0                       0
    ## 3            1                      0                       0
    ## 4            1                      1                       0
    ## 5            1                      1                       0
    ## 6            1                      1                       0
    ## 7            1                      0                       1
    ## 8            1                      0                       1
    ## 9            1                      0                       1
    ## 10           1                      0                       0
    ## 11           1                      0                       0
    ## 12           1                      0                       0
    ## 13           1                      0                       0
    ## 14           1                      0                       0
    ## 15           1                      0                       0
    ## 16           1                      0                       0
    ## 17           1                      0                       0
    ## 18           1                      0                       0
    ## 19           1                      0                       0
    ## 20           1                      0                       0
    ## 21           1                      0                       0
    ## 22           1                      0                       0
    ## 23           1                      0                       0
    ## 24           1                      0                       0
    ## 25           1                      0                       0
    ## 26           1                      0                       0
    ## 27           1                      0                       0
    ## 28           1                      0                       0
    ## 29           1                      0                       0
    ## 30           1                      0                       0
    ##    dgList3$samples$group15 dgList3$samples$group20 dgList3$samples$group30
    ## 1                        0                       0                       0
    ## 2                        0                       0                       0
    ## 3                        0                       0                       0
    ## 4                        0                       0                       0
    ## 5                        0                       0                       0
    ## 6                        0                       0                       0
    ## 7                        0                       0                       0
    ## 8                        0                       0                       0
    ## 9                        0                       0                       0
    ## 10                       1                       0                       0
    ## 11                       1                       0                       0
    ## 12                       1                       0                       0
    ## 13                       0                       1                       0
    ## 14                       0                       1                       0
    ## 15                       0                       1                       0
    ## 16                       0                       0                       1
    ## 17                       0                       0                       1
    ## 18                       0                       0                       1
    ## 19                       0                       0                       0
    ## 20                       0                       0                       0
    ## 21                       0                       0                       0
    ## 22                       0                       0                       0
    ## 23                       0                       0                       0
    ## 24                       0                       0                       0
    ## 25                       0                       0                       0
    ## 26                       0                       0                       0
    ## 27                       0                       0                       0
    ## 28                       0                       0                       0
    ## 29                       0                       0                       0
    ## 30                       0                       0                       0
    ##    dgList3$samples$group45 dgList3$samples$group60 dgList3$samples$group90
    ## 1                        0                       0                       0
    ## 2                        0                       0                       0
    ## 3                        0                       0                       0
    ## 4                        0                       0                       0
    ## 5                        0                       0                       0
    ## 6                        0                       0                       0
    ## 7                        0                       0                       0
    ## 8                        0                       0                       0
    ## 9                        0                       0                       0
    ## 10                       0                       0                       0
    ## 11                       0                       0                       0
    ## 12                       0                       0                       0
    ## 13                       0                       0                       0
    ## 14                       0                       0                       0
    ## 15                       0                       0                       0
    ## 16                       0                       0                       0
    ## 17                       0                       0                       0
    ## 18                       0                       0                       0
    ## 19                       1                       0                       0
    ## 20                       1                       0                       0
    ## 21                       1                       0                       0
    ## 22                       0                       1                       0
    ## 23                       0                       1                       0
    ## 24                       0                       1                       0
    ## 25                       0                       0                       1
    ## 26                       0                       0                       1
    ## 27                       0                       0                       1
    ## 28                       0                       0                       0
    ## 29                       0                       0                       0
    ## 30                       0                       0                       0
    ##    dgList3$samples$group120
    ## 1                         0
    ## 2                         0
    ## 3                         0
    ## 4                         0
    ## 5                         0
    ## 6                         0
    ## 7                         0
    ## 8                         0
    ## 9                         0
    ## 10                        0
    ## 11                        0
    ## 12                        0
    ## 13                        0
    ## 14                        0
    ## 15                        0
    ## 16                        0
    ## 17                        0
    ## 18                        0
    ## 19                        0
    ## 20                        0
    ## 21                        0
    ## 22                        0
    ## 23                        0
    ## 24                        0
    ## 25                        0
    ## 26                        0
    ## 27                        0
    ## 28                        1
    ## 29                        1
    ## 30                        1
    ## attr(,"assign")
    ##  [1] 0 1 1 1 1 1 1 1 1 1
    ## attr(,"contrasts")
    ## attr(,"contrasts")$`dgList3$samples$group`
    ## [1] "contr.treatment"

Another model option is to use **model.matrix(0\~ +
dgList3$samples$group)**, but that’s not a time series analysis. That’s
a matrix for paired comparisons (ie, control vs treatment) like will be
used on next Session.

A key strength of limma’s linear modelling approach, is the ability
accommodate arbitrary experimental complexity. Simple designs, such cell
type and batch, through to more complicated factorial designs and models
with interaction terms can be handled relatively easily. Where
experimental or technical effects can be modelled using a random effect,
another possibility in limma is to estimate correlations using
duplicateCorrelation by specifying a block argument for both this
function and in the lmFit linear modelling step.

### 7.2. Estimating dispersions

We need to estimate the dispersion parameter for our negative binomial
model. If there are only a few samples, it is difficult to estimate the
dispersion accurately for each gene, and so we need a way of’sharing’
information between genes. Possible solutions include: - Using a common
estimate across all genes. - Fitting an estimate based on the
mean-variance trend across the dataset, such that genes with similar
abundances have similar variance estimates (trended dispersion). -
Computing a genewise dispersion (tagwise dispersion)

In edgeR, we use an empirical Bayes method to ‘shrink’ the genewise
dispersion estimates towards the common dispersion (tagwise dispersion).
Note that either the common or trended dispersion needs to be estimated
before we can estimate the tagwise dispersion.

``` r
dgList3a <- estimateGLMCommonDisp(dgList3, design=design.matrix)
dgList3a <- estimateGLMTrendedDisp(dgList3a, design=design.matrix)
dgList3a <- estimateGLMTagwiseDisp(dgList3a, design=design.matrix)
```

We can plot the estimates and see how they differ. The biological
coefficient of variation (BCV) is the square root of the dispersion
parameter in the negative binomial model.

``` r
plotBCV(dgList3a)
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig14-1.png" style="display: block; margin: auto;" />

There is a function in EdgeR that merges the three previous commands
but, the result is not the same therefore, we prefer the three step
method. “The estimateDisp function doesn’t give exactly the same
estimates as the traditional calling sequences”.

``` r
dgList3b <- estimateDisp(dgList3, design.matrix,trend.method="locfit")
```

``` r
par(mfrow=c(1,2))
plotBCV(dgList3a)
title(main="A. three steps")
plotBCV(dgList3b)
title(main="B. one step")
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig15-1.png" style="display: block; margin: auto;" />

The trend is more realistic using “three steps” method. Finally, we
choose to follow using that result

``` r
dgList3 <- dgList3a
```

### 7.3. Fitting and making comparisons

Now, we can use edgeR as a GLM. First fit the data to the count model
before making contrasts of interest.

``` r
fit <- glmFit(dgList3, design.matrix)
fit$method
```

    ## [1] "oneway"

Let’s see the coefficients of this model as derived from the design
matrix (colnames of table):

``` r
colnames(fit) # factors
```

    ##  [1] "(Intercept)"              "dgList3$samples$group5"  
    ##  [3] "dgList3$samples$group10"  "dgList3$samples$group15" 
    ##  [5] "dgList3$samples$group20"  "dgList3$samples$group30" 
    ##  [7] "dgList3$samples$group45"  "dgList3$samples$group60" 
    ##  [9] "dgList3$samples$group90"  "dgList3$samples$group120"

Then, tests can be performed with a log-ratio test (function glmRT). For
instance, to test differential genes between “time 30” and “time 0”,
this is equivalent to testing the nullity of the sixth coefficient (see
the design matrix, section 8.1). This would then ask the question, “Is
there an effect of **time30** on a given gene?”

``` r
lrt <- glmLRT(fit,coef=6)
#kable(head(lrt$table))
head(lrt$table)
```

    ##                 logFC    logCPM         LR      PValue
    ## AT1G01010 -0.19399184 4.3925406 1.07094697 0.300731646
    ## AT1G01020  0.03410815 3.7288930 0.04076184 0.839998327
    ## AT1G01030 -0.24364836 1.5959491 0.93672082 0.333122592
    ## AT1G01040 -0.17675448 6.4727504 2.00560721 0.156718530
    ## AT1G03993 -0.84038901 2.5394185 7.11026405 0.007664382
    ## AT1G01046 -0.06885899 0.2678951 0.05775410 0.810081547

This result does not have FDR correction The following lines will add
this correction using the function topTags

``` r
lrtFDR <- topTags(lrt, n = nrow(dgList3$counts))
#kable(head(lrtFDR$table)) 
head(lrtFDR$table) 
```

    ##               genes    logFC    logCPM       LR        PValue           FDR
    ## AT3G49940 AT3G49940 7.725850  6.337054 613.1818 2.273577e-135 4.992775e-131
    ## AT5G40850 AT5G40850 5.229407  9.389755 461.3782 2.409722e-102  2.645875e-98
    ## AT5G63160 AT5G63160 4.444130  4.434650 422.3554  7.494779e-94  5.486178e-90
    ## AT5G67420 AT5G67420 6.641651  7.683491 386.5701  4.619230e-86  2.535957e-82
    ## AT2G15620 AT2G15620 6.544619 11.115728 384.3355  1.415987e-85  6.219016e-82
    ## AT2G22200 AT2G22200 5.609249  2.896289 372.8157  4.561867e-83  1.669643e-79

Now we included FDR correction :)

We want to keep only gene with significant expression difference across
time so, we could filter our data with a 2X fold of change cutoff and a
FDR \< 0.05 or less. Some people only use the FDR cutoff but, that’s
only a numerical significance… a confidence that two numbers are
sufficiently different considering a distribution model. However, an
important concept to keep in mind is the biological significance: having
twice the concentration of transcripts (2X) is a starting point to
filter out genes without a real impact on the phenotype. Of course,
there are some genes that with low increments will produce big changes.
My advice is to fisrt evaluate final numbers of selected genes with the
standard cutoff. This because downstream analysis don’t work well with
big numbers. You would like to work with a subset of the data (hundreds
to several thousands), not all the genes.

``` r
selectedLRT.list <- lrtFDR$table$FDR < 0.05 & abs(lrtFDR$table$logFC) > 1
selectedLRT <- lrtFDR$table[selectedLRT.list, ]
nrow(selectedLRT)
```

    ## [1] 1040

“selectedLRT” is the list of genes with a expression pattern on “time
30” that differ from the linear model.

The following are the number of genes calculated for each coeficient of
the design matrix and what they represent.

1 21960 \# Intercept  
2 87 \# time 5  
3 210 \# time 10  
4 524 \# time 15  
5 636 \# time 20  
6 1040 \# time 30  
7 2148 \# time 45  
8 1867 \# time 60  
9 3448 \# time 90  
10 3278 \# time 120

Other options to choose are a mix of factors (times).  
For example, from factor 2 to factor 10 (time 2 to time 120): This would
then ask the question, “Is there an effect of **time** on a given gene?”

``` r
lrt210 <- glmLRT(fit,2:10)
```

``` r
lrt210FDR <- topTags(lrt210, n = nrow(dgList3$counts)) # adds FDR correction
```

``` r
selectedLRT210 <- lrt210FDR$table$FDR < 0.05
selectedLRT210 <- lrt210FDR$table[selectedLRT210, ]
nrow(selectedLRT210)
```

    ## [1] 12074

Now, we can explore expression of selected DE genes of factor 6 using
different plots.

#### 7.3.1. MA plot of selected genes

``` r
plotSmear(lrt, de.tags = rownames(selectedLRT))
abline(h=c(-1, 1), col=2)
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig16-1.png" style="display: block; margin: auto;" />

#### 7.3.2. Volcano plot of selected genes

``` r
volcanoData <- cbind(lrtFDR$table$logFC, -log10(lrtFDR$table$FDR))
colnames(volcanoData) <- c("logFC", "negLogPval")
point.col <- ifelse(selectedLRT.list, "red", "black")
plot(volcanoData, pch = 16, col = point.col, cex = 0.5)
abline(v=c(-1, 1), col=2)
abline(h=-log10(0.05), col=2)
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig17-1.png" style="display: block; margin: auto;" />

#### 7.3.3. Heatmap of selected genes using cpm

Selecting the subset

``` r
cpmCounts.select <- cpmCounts[match(rownames(selectedLRT), rownames(dgList3$counts)), ]
```

Ploting a heatmap of selected genes with “Samples as rows”

``` r
finalHMr <- cim(t(cpmCounts.select), color = cimColor, symkey = FALSE, row.cex = 1,
               col.cex = 0.7)
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig19-1.png" style="display: block; margin: auto;" />

Ploting a heatmap with “Samples as columns”

``` r
finalHMc <- cim(cpmCounts.select, color = cimColor, symkey = FALSE, row.cex = 0.7,
               col.cex = 1.5)
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig20-1.png" style="display: block; margin: auto;" />

#### 7.3.4. Some other heatmaps schemes

Default heatmap.2 output

``` r
heatmap.2(cpmCounts.select)
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig21-1.png" style="display: block; margin: auto;" />

Scaling by genes (rows)… also known as z-score

``` r
heatmap.2(cpmCounts.select,scale="row", trace="none", cexRow = 0.8,
          keysize = 1, key.title = NA, key.ylab = NA, 
          lmat = matrix(c(4,2,3,1),
            nrow=2,
            ncol=2),
          lhei = c(0.2,0.8),
          lwid = c(0.15,0.85)
)
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig22-1.png" style="display: block; margin: auto;" />

Grouping samples by time with a distintive color

``` r
heatmap.2(cpmCounts.select,scale="row", trace="none", cexRow = 0.8,
          ColSideColors=Color,
          keysize = 1, key.title = NA, key.ylab = NA, 
)
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig23-1.png" style="display: block; margin: auto;" />

More options:

``` r
?heatmap.2
```

### 7.4. Gene clustering of selected genes

In clustering we are interested in whether there are groups of genes or
groups of samples that have similar gene expression patterns. The first
thing that we have to do is to articulate what we mean by similarity or
dissimilarity as expressed by a measure of distance. We can then use
this measure to cluster genes or samples that are similar. Here, we will
use DE genes to generate a set of clusters with similar expression.

#### 7.4.1. Dendogram of genes

You could use any of the methods described in “Clustering analysis”
(Section 6.2)

On this example we will use the “samples dendogram” of the previous
heatmap “finalHMr” (Section 7.3.3)

``` r
plot(finalHMc$ddr, leaflab="none")
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig24-1.png" style="display: block; margin: auto;" />

The tree could be “cutted” at a desired level to obtain clusters. If we
cut at 35%, we will produce 5 clusters

``` r
plot(finalHMc$ddr, leaflab="none")
abline(h=35, lwd=2, col="pink")
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig25-1.png" style="display: block; margin: auto;" />

This is the commmand to cut the tree at the desired level and save
clusters:

``` r
geneClust <- cutree(as.hclust(finalHMc$ddr), h=35) # 50x3 35x5
```

Number of clusters

``` r
length(unique(geneClust))
```

    ## [1] 5

Listing gene names of cluster 2:

``` r
names(which(geneClust == 2))
```

    ##  [1] "AT5G40850" "AT2G15620" "AT5G41670" "AT1G77760" "AT5G50200" "AT4G05390"
    ##  [7] "AT2G27510" "AT5G03380" "AT5G53460" "AT1G70780" "AT3G47520" "AT2G36580"
    ## [13] "AT2G40140" "AT5G12860" "AT2G38170" "AT1G37130" "AT1G70410" "AT1G64140"
    ## [19] "AT3G55980" "AT4G02380" "AT4G04125" "AT5G25350" "AT1G43710" "AT3G60750"
    ## [25] "ATMG01390" "AT4G32020" "AT4G34135" "AT2G38470" "AT1G12110" "ATCG01210"
    ## [31] "ATCG00920" "AT2G36530" "AT4G05320" "AT4G15550" "AT2G37040" "AT5G40450"
    ## [37] "ATCG00280" "AT2G01021" "AT1G12940" "AT1G61800" "AT4G32940" "AT3G28510"
    ## [43] "ATMG00020" "ATCG00270" "ATCG00900" "ATCG01240" "AT2G01020" "AT3G41979"
    ## [49] "AT4G22495" "AT4G22505" "AT4G22475" "AT4G22485"

To know the number of genes on each cluster:

``` r
length(names(which(geneClust == 1)))
```

    ## [1] 499

``` r
length(names(which(geneClust == 2)))
```

    ## [1] 52

``` r
length(names(which(geneClust == 3)))
```

    ## [1] 390

``` r
length(names(which(geneClust == 4)))
```

    ## [1] 95

``` r
length(names(which(geneClust == 5)))
```

    ## [1] 4

#### 7.4.2. Expression patterns of group of genes

Now, we would like to see the expression pattern of each cluster. We
will use cpm data.

``` r
scaledata <- cpmCounts.select
```

We need a function to obtain the mean expression on each sample of a
desired cluster:

``` r
clust.core = function(i, dat, clusters) {
  ind = (clusters == i)
  colMeans(dat[ind,])
}
```

When we apply the function, there will be obtained core expression of
each cluster

``` r
cores <- sapply(unique(geneClust), clust.core, scaledata, geneClust)
head(cores)
```

    ##           [,1]     [,2]     [,3]      [,4]     [,5]
    ## t0.r1 4.305324 8.043857 1.089193 -1.842164 14.20026
    ## t0.r2 4.259310 8.098287 1.026089 -1.922748 14.45050
    ## t0.r3 4.259639 8.196532 1.059627 -1.881147 15.12746
    ## t5.r1 4.225494 7.822716 1.113468 -1.770136 13.17946
    ## t5.r2 4.336090 7.783656 1.261457 -1.492072 12.89047
    ## t5.r3 4.307237 7.815662 1.077194 -1.944021 12.99491

Now, we prepare the data frame

``` r
d <- data.frame(cbind(dgList$metadata$Time,cores))
colnames(d) <-c("time", paste0("clust_",1:ncol(cores)))
#get the data frame into long format for plotting
dmolten <- melt(d, id.vars = "time")
#order by time
dmolten <- dmolten[order(dmolten$time),]
breaks=c(as.numeric(levels(factor(dgList$metadata$Time))))
```

Make the plot:

``` r
p0 <- ggplot(dmolten, aes(time, value, col=variable)) +
  geom_point() +
  geom_line() +
  scale_x_continuous(minor_breaks = NULL, breaks=breaks) +
  xlab("Time") +
  ylab("Expression") +
  labs(title= "Clusters",color = "Cluster")
p0
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig26-1.png" style="display: block; margin: auto;" />

We can select a specific cluster to draw the expression of their genes.
I will select cluster1

**Cluster 1**

Subset the complete data by cluster =1:

``` r
dClust1 <- t(scaledata[geneClust==1,])
# add the time
dClust1 <- data.frame(cbind(data.frame(dgList$metadata$Time),dClust1))
colnames(dClust1)[1] <- "time"
# get the data frame into long format for plotting
dClust1molten <- melt(dClust1, id.vars = "time")
# order by time
dClust1molten <- dClust1molten[order(dClust1molten$time),]
# Subset the cores molten dataframe so we can plot core1
core1 <- dmolten[dmolten$variable=="clust_1",]
```

Now, to plot this gene expression we use the group=variable and change
the geom\_line to grey. Then we add on top of expression the mean
expression of the core in blue passing the the core data to geom\_line.

``` r
p1 <- ggplot(dClust1molten, aes(time, value, group=variable)) + 
  geom_line(color="grey") +
  geom_point(data=core1, aes(time,value), color="blue") + 
  geom_line(data=core1, aes(time,value), color="blue") +
  scale_x_continuous(minor_breaks = NULL, breaks=breaks) +
  xlab("Time") +
  ylab("Expression") +
  labs(title= "Cluster1", color = "Cluster")
p1
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig27-1.png" style="display: block; margin: auto;" />

**All clusters**

We can do it for all the clusters. The following lines create ggplot
objects p2 to p5, with the expression data of all clusters. Then, all
plots will be joined as an unique plot.

``` r
#Function to create plots
cluster_plot<- function(scaledata, clust){
  dClust <- t(scaledata[geneClust==clust,])
  #add time
  dClust <- data.frame(cbind(data.frame(dgList$metadata$Time),dClust))
  colnames(dClust)[1] <- "time"
  #get the data frame into long format for plotting
  dClustmolten <- melt(dClust, id.vars = "time")
  #order by time
  dClustmolten <- dClustmolten[order(dClustmolten$time),]
  #Subset the cores molten dataframe so we can plot core1
  core <- dmolten[dmolten$variable==paste0("clust_",clust),]

  p <- ggplot(dClustmolten, aes(time,value, group=variable)) + 
  geom_line(color="grey") +
  geom_point(data=core, aes(time,value), color="blue") + 
  geom_line(data=core, aes(time,value), color="blue") +
  scale_x_continuous(minor_breaks = NULL, breaks=c(as.numeric(levels(factor(dmolten$time))))) +
  xlab("Time") +
  ylab("Expression") +
  labs(title= paste0("Cluster",clust),color = paste0("Cluster",clust))
return(p)
}
```

``` r
p2 <- cluster_plot(scaledata, 2)
p3 <- cluster_plot(scaledata, 3)
p4 <- cluster_plot(scaledata, 4)
p5 <- cluster_plot(scaledata, 5)
```

Now, we are ready to plot all clusters:

``` r
grid.arrange(p0 + theme(legend.position = "none"),
             p1+ ylab(NULL),
             p2+ ylab(NULL),
             p3+ ylab(NULL),
             p4+ ylab(NULL),
             p5+ ylab(NULL),
             ncol=6)
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig28-1.png" style="display: block; margin: auto;" />

Cool, but each graph have their own scale and they are not sorted. Look
for the max and min axes value through the graphs to set global values.

``` r
# Setting min a max limits of y axes based on all five plots
ylim1=-6; ylim2=17
```

Finally, sort the command lines of expression plots according to p0
order and plot\!

``` r
# don't forget to maximize the empty window before the plot
grid.arrange(p0 + ylim(ylim1, ylim2) + theme(legend.position = "none"),
             p5 + ylim(ylim1, ylim2) + ylab(NULL), 
             p2 + ylim(ylim1, ylim2) + ylab(NULL), 
             p1 + ylim(ylim1, ylim2) + ylab(NULL), 
             p3 + ylim(ylim1, ylim2) + ylab(NULL), 
             p4 + ylim(ylim1, ylim2) + ylab(NULL), 
             ncol=6)
```

<img src="3_Pipeline-Gene_expression_analysis.v2_files/figure-gfm/fig29-1.png" style="display: block; margin: auto;" />

#### 7.4.3. Saving results

Now, you would want to know what functions are represented on this
selected genes. This will be explained on session5.

Meanwhile you can save the gene list of each cluster nor the list of all
clusters to a text file to be ready.

``` r
# Cluster2 of factor6
write.table(names(which(geneClust == 2)), "factor6.clust2.txt", sep="\t", quote = FALSE, row.names = F, col.names = F)

# All genes of factor6
write.table(names(geneClust), "factor6.all.txt", sep="\t", quote = FALSE, row.names = F, col.names = F)
```

## 8\. Session info

``` r
sessionInfo()
```

    ## R version 3.5.3 (2019-03-11)
    ## Platform: x86_64-w64-mingw32/x64 (64-bit)
    ## Running under: Windows 10 x64 (build 18363)
    ## 
    ## Matrix products: default
    ## 
    ## locale:
    ## [1] LC_COLLATE=Spanish_Chile.1252  LC_CTYPE=Spanish_Chile.1252   
    ## [3] LC_MONETARY=Spanish_Chile.1252 LC_NUMERIC=C                  
    ## [5] LC_TIME=Spanish_Chile.1252    
    ## 
    ## attached base packages:
    ## [1] stats     graphics  grDevices utils     datasets  methods   base     
    ## 
    ## other attached packages:
    ##  [1] matrixStats_0.56.0 mclust_5.4.6       gplots_3.0.3       reshape2_1.4.4    
    ##  [5] mixOmics_6.6.2     lattice_0.20-38    MASS_7.3-51.5      RColorBrewer_1.1-2
    ##  [9] gridExtra_2.3      ggplot2_3.3.0      pvclust_2.2-0      dplyr_0.8.5       
    ## [13] knitr_1.28         edgeR_3.24.3       limma_3.38.3      
    ## 
    ## loaded via a namespace (and not attached):
    ##  [1] gtools_3.8.2       tidyselect_1.0.0   locfit_1.5-9.4     xfun_0.13         
    ##  [5] purrr_0.3.4        splines_3.5.3      colorspace_1.4-1   vctrs_0.2.4       
    ##  [9] htmltools_0.4.0    yaml_2.2.1         rlang_0.4.5        pillar_1.4.3      
    ## [13] glue_1.4.0         withr_2.2.0        lifecycle_0.2.0    plyr_1.8.6        
    ## [17] stringr_1.4.0      munsell_0.5.0      gtable_0.3.0       caTools_1.17.1.2  
    ## [21] evaluate_0.14      labeling_0.3       parallel_3.5.3     highr_0.8         
    ## [25] rARPACK_0.11-0     Rcpp_1.0.4.6       KernSmooth_2.23-15 corpcor_1.6.9     
    ## [29] scales_1.1.0       gdata_2.18.0       farver_2.0.3       RSpectra_0.16-0   
    ## [33] ellipse_0.4.1      digest_0.6.25      stringi_1.4.6      grid_3.5.3        
    ## [37] tools_3.5.3        bitops_1.0-6       magrittr_1.5       tibble_3.0.1      
    ## [41] crayon_1.3.4       tidyr_1.0.2        pkgconfig_2.0.3    ellipsis_0.3.0    
    ## [45] Matrix_1.2-18      assertthat_0.2.1   rmarkdown_2.1      R6_2.4.1          
    ## [49] igraph_1.2.5       compiler_3.5.3
