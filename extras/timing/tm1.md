tm1
================

From: <https://github.com/tidyverse/tidyr/issues/613>.

``` r
require(tidyverse)
```

    ## Loading required package: tidyverse

    ## ── Attaching packages ──────────────────────────────────────────── tidyverse 1.2.1 ──

    ## ✔ ggplot2 3.1.1       ✔ purrr   0.3.2  
    ## ✔ tibble  2.1.1       ✔ dplyr   0.8.0.1
    ## ✔ tidyr   0.8.3       ✔ stringr 1.4.0  
    ## ✔ readr   1.3.1       ✔ forcats 0.4.0

    ## Warning: package 'ggplot2' was built under R version 3.5.2

    ## Warning: package 'tibble' was built under R version 3.5.2

    ## Warning: package 'tidyr' was built under R version 3.5.2

    ## Warning: package 'purrr' was built under R version 3.5.2

    ## Warning: package 'dplyr' was built under R version 3.5.2

    ## Warning: package 'stringr' was built under R version 3.5.2

    ## Warning: package 'forcats' was built under R version 3.5.2

    ## ── Conflicts ─────────────────────────────────────────────── tidyverse_conflicts() ──
    ## ✖ dplyr::filter() masks stats::filter()
    ## ✖ dplyr::lag()    masks stats::lag()

``` r
require(data.table)
```

    ## Loading required package: data.table

    ## Warning: package 'data.table' was built under R version 3.5.2

    ## 
    ## Attaching package: 'data.table'

    ## The following objects are masked from 'package:dplyr':
    ## 
    ##     between, first, last

    ## The following object is masked from 'package:purrr':
    ## 
    ##     transpose

``` r
data.test <- matrix(
  data = sample(
    x = c(0L, 1L, 2L, NA_integer_),#the genotypes
    size = 2e+07,
    replace = TRUE,
    prob = c(0.8, 0.10, 0.05, 0.05)
    ),
  nrow = 20000,#number of SNPs/markers
  ncol = 1000,#number of samples
  dimnames = list(rownames = seq(1, 20000, 1), colnames = seq(1, 1000, 1))
  ) %>%
  tibble::as_tibble(x = ., rownames = "MARKERS") 
```

``` r
library("cdata")
library("rqdatatable")
```

    ## Loading required package: rquery

    ## Warning: package 'rquery' was built under R version 3.5.2

``` r
packageVersion("data.table")
```

    ## [1] '1.12.2'

``` r
packageVersion("tidyr")
```

    ## [1] '0.8.3'

``` r
packageVersion("dplyr")
```

    ## [1] '0.8.0.1'

``` r
packageVersion("cdata")
```

    ## [1] '1.1.0'

``` r
packageVersion("rqdatatable")
```

    ## [1] '1.1.5'

``` r
data.test <- data.frame(data.test)
```

test1: data.table::melt.data.table

``` r
system.time(
test1 <- data.table::as.data.table(data.test) %>%
  data.table::melt.data.table(
    data = .,
    id.vars = "MARKERS",
    variable.name = "INDIVIDUALS",
    value.name = "GENOTYPES",
    variable.factor = FALSE) 
)
```

    ##    user  system elapsed 
    ##   0.605   0.140   0.775

``` r
# reported: #~0.41sec
```

``` r
test1 <- orderby(test1, qc(MARKERS, INDIVIDUALS, GENOTYPES)) 
```

test2: tidyr::gather

``` r
system.time(
test2 <- tidyr::gather(
  data = data.test,
  key = "INDIVIDUALS",
  value = "GENOTYPES",
  -MARKERS)
)
```

    ##    user  system elapsed 
    ##   0.433   0.190   0.627

``` r
# reported: #~0.39sec
```

``` r
test2 <- orderby(test2, qc(MARKERS, INDIVIDUALS, GENOTYPES)) 
stopifnot(isTRUE(all.equal(test1, test2)))
```

test 2b: reverse

``` r
system.time({
  test2b <- tidyr::spread(test2, key = INDIVIDUALS, value = GENOTYPES)
})
```

    ##    user  system elapsed 
    ##   6.041   1.674   7.890

``` r
test2b <- orderby(test2b, colnames(test2b)) 
data.test <- orderby(test2b, colnames(data.test)) 
stopifnot(isTRUE(all.equal(data.test, test2b)))
```

test3: latest tidyr::pivot\_longer

``` r
run_pivot_longer <- exists('pivot_longer', 
                           where = 'package:tidyr', 
                           mode = 'function')
```

``` r
system.time(
test3 <- tidyr::pivot_longer(
  df = data.test,
  cols = -MARKERS,
  names_to = "INDIVIDUALS",
  values_to = "GENOTYPES")
)
# reported: #~90sec !!!
```

``` r
test3 <- orderby(test3, qc(MARKERS, INDIVIDUALS, GENOTYPES)) 
stopifnot(isTRUE(all.equal(test1, test3)))
```

test 4: cdata::unpivot\_to\_blocks() (with data.table)

``` r
system.time({
  cT <- build_unpivot_control(
    nameForNewKeyColumn = "INDIVIDUALS",
    nameForNewValueColumn = "GENOTYPES",
    columnsToTakeFrom = setdiff(colnames(data.test), 
                                c("MARKERS", "INDIVIDUALS", "GENOTYPES")))
  layout <- rowrecs_to_blocks_spec(
    cT,
    recordKeys = "MARKERS",
    allow_rqdatatable = TRUE)
  
  print(layout$allow_rqdatatable)
  
  test4 <- layout_by(layout, data.test)
})
```

    ## [1] TRUE

    ##    user  system elapsed 
    ##   0.689   0.245   0.841

``` r
test4 <- orderby(test4, qc(MARKERS, INDIVIDUALS, GENOTYPES)) 
stopifnot(isTRUE(all.equal(test1, test4)))
```

Slow.

``` r
system.time({
  inv_layout <- t(layout)
  
  print(inv_layout$allow_rqdatatable)

  back4 <- layout_by(inv_layout, test4)
})
```

    ## [1] TRUE

    ##    user  system elapsed 
    ## 107.981   2.945 113.544

``` r
back4 <- orderby(back4, colnames(back4)) 
data.test <- orderby(back4, colnames(data.test)) 
stopifnot(isTRUE(all.equal(data.test, back4)))
```

------------------------------------------------------------------------

test 5: cdata::unpivot\_to\_blocks() (without data.table)

Slow.

``` r
system.time({
  cT <- build_unpivot_control(
    nameForNewKeyColumn = "INDIVIDUALS",
    nameForNewValueColumn = "GENOTYPES",
    columnsToTakeFrom = setdiff(colnames(data.test), 
                                c("MARKERS", "INDIVIDUALS", "GENOTYPES")))
  layout <- rowrecs_to_blocks_spec(
    cT,
    recordKeys = "MARKERS",
    allow_rqdatatable = FALSE)
  
  print(layout$allow_rqdatatable)
  
  test5 <- layout_by(layout, data.test)
})
```

    ## [1] FALSE

    ##    user  system elapsed 
    ##  93.218  76.064 179.400

``` r
test5 <- orderby(test5, qc(MARKERS, INDIVIDUALS, GENOTYPES)) 
stopifnot(isTRUE(all.equal(test1, test5)))
```

Slow.

``` r
system.time({
  inv_layout <- t(layout)

  print(inv_layout$allow_rqdatatable)
    
  back5 <- layout_by(inv_layout, test5)
})
```

    ## [1] FALSE

    ##    user  system elapsed 
    ## 177.780  55.351 251.659

``` r
back5 <- orderby(back5, colnames(back5)) 
data.test <- orderby(back5, colnames(data.test)) 
stopifnot(isTRUE(all.equal(data.test, back5)))
```
