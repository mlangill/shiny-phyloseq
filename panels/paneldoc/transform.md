# 

**NOTE:** This is merely a documentation panel.

The following subsections define
different options available
in one or more of the `Transform` widgets
that are **located on other panels**
in Shiny-phyloseq.

- **Implication** - When one of the following options is selected,
it implies that the transformed values
are being used for the analysis/graphics
of the currently-active panel
that you are looking at.
- **Efficiency** - The choice of transformation in one widget
does not affect the `Transform` widget selection in other panels --
so you can freely try different transformations in one panel
without worrying that you are inadvertantly changing
the result you've already seen in some other panel.
However, the transform computation for a given transformation option
is always the same
so Shiny-phyloseq will only need to compute the transformation
the first time that you select it *in any panel*.
This notion is reset if you change the dataset selection on the `Data` panel,
or if you change the filtering options in the `Filter` panel.

## Counts

The `Counts` option is the default, and simply means 
that the filtered count values are used in the respective analysis 
of the currently active panel.
No calculation is performed or required.
If you skipped the `Filter` panel,
then this option implies that the original unfiltered counts values are being used.


## Prop

This option means that the simple proportion values are being used. 
All entries in the OTU table are divided
by their respective library size.
In an OTU-by-Sample table, this means dividing every value by its column's sum.
This is equivalent in phyloseq to the following


```r
transform_sample_counts(physeq, function(x) x/sum(x))
```


## RLog - Regularized Log

The [the DESeq2 package](http://www.bioconductor.org/packages/release/bioc/html/DESeq2.html)
provides two forms of variance-stabilizing transformations.
Both [the DESeq2 beginner's vignette](http://www.bioconductor.org/packages/release/bioc/vignettes/DESeq2/inst/doc/beginner.pdf)
and [the DESeq2 main vignette, Data transformations and visualization section](http://www.bioconductor.org/packages/release/bioc/vignettes/DESeq2/inst/doc/DESeq2.pdf),
provide helpful theoretical information about these transformations,
and why they might be useful
for sample-wise exploratory data analysis.

The `RLog` option is one of our favorites,
and implies the results of a Regularized Transformation.
Because this is an interactive tool and these dispersion estimates
can sometimes take a long time to calculate,
we have implemented this option with the `fast = TRUE` option.
This helps tremendously in wait-time,
at some potential cost to accuracy that you might want to be aware of.
See the `rlog` documentation of DESeq2 for more details.
The `blind=TRUE` option is also used,
meaning that the entire dataset is used for dispersion estimation,
rather than separating the samples by their experimental design class.
In some instances this might imply an overestimate of dispersion,
but is a conservative choice in the context of Shiny-phyloseq,
because there is a potentially wide-range of experimental designs.
For more complicated designs, or if dispersion is being wildly overestimated
as a result of this choice,
you may want to transform your data ahead of time, in batch.

To that end, and for your own guidance using Shiny-phyloseq,
the following is an example of the regularized log transformation
implemented in Shiny-phyloseq.


```r
dds = phyloseq_to_deseq2(physeq, ~ 1)
rld <- DESeq2::rlog(dds, blind = TRUE, fast = TRUE)
rlogMat <- GenomicRanges::assay(rld)
physeq1 = physeq
otu_table(physeq1) <- otu_table(rlogMat, taxa_are_rows = TRUE)
```

where `physeq` is an object containing the imported experimental data,
represented in phyloseq format. 


## CLR Centered Log-Ratio transformation

CLR transformation is championed in an 
[article describing ALDEx2](http://www.ncbi.nlm.nih.gov/pmc/articles/PMC4030730/)
for analyzing HTSeq count data.

Defined in 1986 by Aitchison, a more-recent article 
([Egozcue 2003](http://link.springer.com/article/10.1023%2FA%3A1023818214614))
defines and explains some details surrounding CLR.

CLR can be defined as

`x / g(x)`

Where `x` is our original vector of compositional values, and `g(x)` is the geometric mean of `x`.

So, to define a zero- and NA-tolerant function for calculating CLR:


```r
x = c(0, 0, 15, 3, 1, 0, 9)
gm_mean = function(x, na.rm=TRUE){
  # The geometric mean, with some error-protection bits.
  exp(sum(log(x[x > 0 & !is.na(x)]), na.rm=na.rm) / length(x))
}
clr = function(x, base=2){
  x <- log((x / gm_mean(x)), base)
  x[!is.finite(x) | is.na(x)] <- 0.0
  return(x)
}
clr(x, 2)
```

```
## [1]  0.0000  0.0000  2.6695  0.3476 -1.2374  0.0000  1.9325
```

```r
clr(x, exp(1))
```

```
## [1]  0.0000  0.0000  1.8504  0.2409 -0.8577  0.0000  1.3395
```

Example from aldex2 paper:


```r
x = c(10, 35, 50, 500)
clr(x, 2)
```

```
## [1] -2.4433 -0.6359 -0.1214  3.2006
```

The values reported in the ALDex2 article for their example were -2.44, -0.64, -0.12, 3.2.
Which matches the rounded versions exactly,


```r
c(-2.44, -0.64, -0.12, 3.20) - round(clr(x, 2), 2)
```

```
## [1] 0 0 0 0
```

Note that they use a Bayesian method for handling zeroes in the ALDex2 artice.
This approach assumes that the reason no reads were detected in some features 
was because of sampling variance. 
This is of course wrong in the microbial case.
It is possible for microbes to be present in some samples, 
and truly absent in others.

Here is what the example values look like if there was one or two zeroes present,
in my formulation of the transformation.


```r
x = c(10, 35, 50, 500)
clr(x, 2)
```

```
## [1] -2.4433 -0.6359 -0.1214  3.2006
```

```r
x = c(10, 35, 50, 500, 0)
clr(x, 2)
```

```
## [1] -1.2902  0.5171  1.0317  4.3536  0.0000
```

```r
x = c(10, 35, 50, 500, 0, 0)
clr(x, 2)
```

```
## [1] -0.5215  1.2858  1.8004  5.1223  0.0000  0.0000
```

The presence of zeroes clearly makes a difference in the CLR result,
as expected by how they are (somewhat crudely) managed in this approach.
It does not change the ranking of features, 
but shifts them more positive (it is equivalent to replacing values `<=0` with a `1`).

