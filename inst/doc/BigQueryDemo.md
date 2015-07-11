<!-- R Markdown Documentation, DO NOT EDIT THE PLAIN MARKDOWN VERSION OF THIS FILE -->

<!-- Copyright 2015 Google Inc. All rights reserved. -->

<!-- Licensed under the Apache License, Version 2.0 (the "License"); -->
<!-- you may not use this file except in compliance with the License. -->
<!-- You may obtain a copy of the License at -->

<!--     http://www.apache.org/licenses/LICENSE-2.0 -->

<!-- Unless required by applicable law or agreed to in writing, software -->
<!-- distributed under the License is distributed on an "AS IS" BASIS, -->
<!-- WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. -->
<!-- See the License for the specific language governing permissions and -->
<!-- limitations under the License. -->

# Analyzing Variants with BigQuery

Google Genomics can import variant calls from VCF files or Complete Genomics masterVar files so that you can query them with a simple API as we saw earlier in vignette [Working with Variants](http://bioconductor.org/packages/devel/bioc/vignettes/GoogleGenomics/inst/doc/VariantAnnotation-comparison-test.html).  You can also export Variants to BigQuery for interactive analysis of these large datasets.  For more detail, see https://cloud.google.com/genomics/v1/managing-variants

In this example we will work with the [Illumina Platinum Genomes](http://googlegenomics.readthedocs.org/en/latest/use_cases/discover_public_data/platinum_genomes.html) dataset.

## Run a query from R

The [bigrquery](https://github.com/hadley/bigrquery) package written by Hadley Wickham implements an R interface to [Google BigQuery](https://cloud.google.com/bigquery/).


```r
require(bigrquery)
```


```r
######################[ TIP ]##################################
## Set the Google Cloud Platform project id under which these queries will run.
## If you are using the Bioconductor workshop docker image, this is already
## set for you in your .Rprofile.

# project <- "YOUR-PROJECT-ID"
#####################################################################
```


```r
# Change the table here if you wish to run these queries against a different Variants table.
theTable <- "genomics-public-data:platinum_genomes.variants"
# theTable <- "genomics-public-data:1000_genomes.variants"
# theTable <- "genomics-public-data:1000_genomes_phase_3.variants"
```

Let's start by just counting the number of records in the table:

```r
querySql <- paste("SELECT COUNT(*) FROM [", theTable, "]", sep="")
querySql
```

```
## [1] "SELECT COUNT(*) FROM [genomics-public-data:platinum_genomes.variants]"
```

And send the query to the cloud for execution:

```r
result <- query_exec(querySql, project=project)
result
```

```
##         f0_
## 1 688167235
```
And we see that the table has 688167235 rows - wow!

## Run a query using the BigQuery Web User Interface

So what is actually in this table?  Click on [this link](https://bigquery.cloud.google.com/table/genomics-public-data:platinum_genomes.variants) to view the schema in the BigQuery web user interface.

We can also run the exact same query using the BigQuery web user interface.  In the BigQuery web user interface:

(1) click on the *"Compose Query"* button
(2) paste in the SQL for the query we just ran via R
(3) click on *"Run Query"*.

## Run a query stored in a file from R

Instead of typing SQL directly into our R code, we can use a convenience function to read SQL from a file.

```r
DisplayAndDispatchQuery <- function(queryUri, project, replacements=list()) {
  if (missing(queryUri)) {
    stop("Pass the file path or url to the file containing the query.")
  }
  if(missing(project)) {
    stop("Pass the project id of your Google Cloud Platform project.")
  }

  if (grepl("^https.*", queryUri)) {
    # Read the query from a remote location.
    querySql <- getURL(queryUri, ssl.verifypeer=FALSE)
  } else {
    # Read the query from the local filesystem.
    querySql <- readChar(queryUri, nchars=1e6)
  }

  # If applicable, substitute values in the query template.
  for(replacement in names(replacements)) {
    querySql <- gsub(replacement, replacements[[replacement]], querySql, fixed=TRUE)
  }

  # Display the query to the terminal.
  cat(querySql)

  # Dispatch the query to BigQuery.
  query_exec(querySql, project)
}
```

This allows queries to be more easily shared among analyses and also reused for different datasets.  For example, in the following file we have a query that will retrieve data from any table exported from a Google Genomics Variant Set.

```r
file.show(file.path(system.file(package = "Bioc2015Workshop"),
                    "sql",
                    "variant-level-data-for-brca1.sql"))
```

Now let's run the query to retrieve variant data for BRCA1:

```r
result <- DisplayAndDispatchQuery(file.path(system.file(package = "Bioc2015Workshop"),
                                            "sql",
                                            "variant-level-data-for-brca1.sql"),
                                  project=project,
                                  replacements=list("_THE_TABLE_"=theTable))
```

```
# Retrieve variant-level information for BRCA1 variants.
SELECT
  reference_name,
  start,
  end,
  reference_bases,
  GROUP_CONCAT(alternate_bases) WITHIN RECORD AS alternate_bases,
  quality,
  GROUP_CONCAT(filter) WITHIN RECORD AS filter,
  GROUP_CONCAT(names) WITHIN RECORD AS names,
  COUNT(call.call_set_name) WITHIN RECORD AS num_samples,
FROM
  [genomics-public-data:platinum_genomes.variants]
WHERE
  reference_name CONTAINS '17' # To match both 'chr17' and '17'
  AND start BETWEEN 41196311 AND 41277499
# Skip non-variant segments if the source data was gVCF or CGI data
OMIT RECORD IF EVERY(alternate_bases IS NULL) OR EVERY(alternate_bases = '<NON_REF>')
ORDER BY
  start,
  alternate_bases
```
Number of rows returned by this query: 335.

Results from [bigrquery](https://github.com/hadley/bigrquery) are dataframes:

```r
mode(result)
```

```
## [1] "list"
```

```r
class(result)
```

```
## [1] "data.frame"
```

```r
summary(result)
```

```
##  reference_name         start               end          
##  Length:335         Min.   :41196407   Min.   :41196408  
##  Class :character   1st Qu.:41218836   1st Qu.:41218837  
##  Mode  :character   Median :41239914   Median :41239916  
##                     Mean   :41238083   Mean   :41238085  
##                     3rd Qu.:41258167   3rd Qu.:41258168  
##                     Max.   :41277186   Max.   :41277187  
##  reference_bases    alternate_bases       quality       
##  Length:335         Length:335         Min.   :   0.00  
##  Class :character   Class :character   1st Qu.:  48.56  
##  Mode  :character   Mode  :character   Median : 317.76  
##                                        Mean   : 445.50  
##                                        3rd Qu.: 737.80  
##                                        Max.   :6097.85  
##     filter             names            num_samples    
##  Length:335         Length:335         Min.   : 1.000  
##  Class :character   Class :character   1st Qu.: 2.000  
##  Mode  :character   Mode  :character   Median : 7.000  
##                                        Mean   : 5.304  
##                                        3rd Qu.: 7.000  
##                                        Max.   :17.000
```

```r
head(result)
```

```
##   reference_name    start      end reference_bases alternate_bases quality
## 1          chr17 41196407 41196408               G               A  733.47
## 2          chr17 41196820 41196822              CT               C   63.74
## 3          chr17 41196820 41196823             CTT            C,CT  314.59
## 4          chr17 41196840 41196841               G               T   85.68
## 5          chr17 41197273 41197274               C               A 1011.08
## 6          chr17 41197938 41197939               A              AT   86.95
##                                       filter names num_samples
## 1                                       PASS  <NA>           7
## 2                                      LowQD  <NA>           1
## 3                                       PASS  <NA>           3
## 4 TruthSensitivityTranche99.90to100.00,LowQD  <NA>           2
## 5                                       PASS  <NA>           7
## 6                                      LowQD  <NA>           3
```

## Visualize Query Results

The prior query was basically a data retrieval similar to what we performed earlier in this workshop when we used the GoogleGenomics Bioconductor package to retrieve data from the Google Genomics Variants API.  

But BigQuery really shines when it is used to perform an actual *analysis* - do the heavy-lifting on the big data resident in the cloud, and bring back the result of the analysis to R for further downstream analysis and visualization.

Let's do that now with a query that computes the Transition Transversion ratio for the variants within genomic region windows.

```r
result <- DisplayAndDispatchQuery(file.path(system.file(package = "Bioc2015Workshop"),
                                            "sql",
                                            "ti-tv-ratio.sql"),
                                  project=project,
                                  replacements=list("_THE_TABLE_"=theTable,
                                                    "_WINDOW_SIZE_"=100000))
```

```
# Compute the Ti/Tv ratio for variants within genomic region windows.
SELECT
  reference_name,
  window * 1e+05 AS window_start,
  transitions,
  transversions,
  transitions/transversions AS titv,
  num_variants_in_window,
FROM (
  SELECT
    reference_name,
    window,
    SUM(mutation IN ('A->G', 'G->A', 'C->T', 'T->C')) AS transitions,
    SUM(mutation IN ('A->C', 'C->A', 'G->T', 'T->G',
                     'A->T', 'T->A', 'C->G', 'G->C')) AS transversions,
    COUNT(mutation) AS num_variants_in_window
  FROM (
    SELECT
      reference_name,
      INTEGER(FLOOR(start / 1e+05)) AS window,
      CONCAT(reference_bases, CONCAT(STRING('->'), alternate_bases)) AS mutation,
      COUNT(alternate_bases) WITHIN RECORD AS num_alts,
    FROM
      [genomics-public-data:platinum_genomes.variants]
    # Optionally add clause here to limit the query to a particular
    # region of the genome.
    #_WHERE_
    HAVING
      # Skip 1/2 genotypes _and non-SNP variants
      num_alts = 1
      AND reference_bases IN ('A','C','G','T')
      AND alternate_bases IN ('A','C','G','T'))
  GROUP BY
    reference_name,
    window)
ORDER BY
  window_start
Retrieving data:  2.2sRetrieving data:  4.0s
```
Number of rows returned by this query: 28734.


```r
summary(result)
```

```
##  reference_name      window_start        transitions     transversions   
##  Length:28734       Min.   :        0   Min.   :   0.0   Min.   :   0.0  
##  Class :character   1st Qu.: 34600000   1st Qu.: 145.0   1st Qu.: 109.0  
##  Mode  :character   Median : 71000000   Median : 194.0   Median : 142.0  
##                     Mean   : 79795469   Mean   : 214.8   Mean   : 158.2  
##                     3rd Qu.:115400000   3rd Qu.: 252.0   3rd Qu.: 182.0  
##                     Max.   :249200000   Max.   :6313.0   Max.   :6703.0  
##                                                                          
##       titv       num_variants_in_window
##  Min.   :0.000   Min.   :    1.0       
##  1st Qu.:1.173   1st Qu.:  258.0       
##  Median :1.362   Median :  338.0       
##  Mean   :1.375   Mean   :  373.1       
##  3rd Qu.:1.557   3rd Qu.:  432.0       
##  Max.   :5.000   Max.   :12001.0       
##  NA's   :7
```

```r
head(result)
```

```
##   reference_name window_start transitions transversions     titv
## 1           chr7            0         483           272 1.775735
## 2           chr1            0         293           198 1.479798
## 3           chr8            0         248           109 2.275229
## 4          chr16            0         138           125 1.104000
## 5           chr3            0         100           101 0.990099
## 6          chr20            0          43            36 1.194444
##   num_variants_in_window
## 1                    755
## 2                    491
## 3                    357
## 4                    263
## 5                    201
## 6                     79
```

Since [bigrquery](https://github.com/hadley/bigrquery) results are dataframes, we can make use of all sorts of other great R packages to do our downstream work.  Here we use a few more packages from the Hadleyverse for data filtering and visualization.


```r
require(dplyr)
```


```r
# Change this filter if you want to visualize the result of this analysis for a different chromosome.
chromosomeOneResults <- filter(result, reference_name == "chr1" | reference_name == "1")
```


```r
require(ggplot2)
require(scales)
```


```r
ggplot(chromosomeOneResults, aes(x=window_start, y=titv)) +
  geom_point() +
  stat_smooth() +
  scale_x_continuous(labels=comma) +
  xlab("Genomic Position") +
  ylab("Ti/Tv") +
  ggtitle("Ti/Tv by 100,000 base pair windows on Chromosome 1")
```

![plot of chunk titv](figure/titv-1.png) 

## Provenance

```r
sessionInfo()
```

```
R version 3.2.0 (2015-04-16)
Platform: x86_64-apple-darwin13.4.0 (64-bit)
Running under: OS X 10.10.3 (Yosemite)

locale:
[1] en_US.UTF-8/en_US.UTF-8/en_US.UTF-8/C/en_US.UTF-8/en_US.UTF-8

attached base packages:
[1] stats     graphics  grDevices utils     datasets  methods   base     

other attached packages:
[1] mgcv_1.8-6           nlme_3.1-120         scales_0.2.4        
[4] ggplot2_1.0.1        dplyr_0.4.1          bigrquery_0.1.0     
[7] knitr_1.10.5         Bioc2015Workshop_0.1

loaded via a namespace (and not attached):
 [1] Rcpp_0.11.6      rstudioapi_0.3.1 magrittr_1.5     MASS_7.3-40     
 [5] munsell_0.4.2    lattice_0.20-31  colorspace_1.2-6 R6_2.1.0        
 [9] stringr_1.0.0    httr_1.0.0       plyr_1.8.2       tools_3.2.0     
[13] parallel_3.2.0   grid_3.2.0       gtable_0.1.2     DBI_0.3.1       
[17] lazyeval_0.1.10  assertthat_0.1   digest_0.6.8     Matrix_1.2-0    
[21] reshape2_1.4.1   formatR_1.2      curl_0.9.1       evaluate_0.7    
[25] labeling_0.3     stringi_0.5-5    jsonlite_0.9.16  proto_0.3-10    
```
