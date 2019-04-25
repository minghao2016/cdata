cdata Exercises
================
John Mount, Win-Vector LLC
Thu Apr 25, 2019

In [this note](https://github.com/WinVector/cdata/blob/master/extras/exercises/exercises_5.md) we are going to use some [`tidyr`](https://CRAN.R-project.org/package=tidyr) data transform demonstration examples, and use them to demonstrate how to use the [`cdata`](https://CRAN.R-project.org/package=cdata) [`R`](https://www.r-project.org) data layout package.

The idea is these are a good cross-section of data layout problems, so they are a good set of examples or exercises to work through.

For each of these examples we will re-layout the data using [`cdata`](https://github.com/WinVector/cdata).

Introduction
------------

The layout transform design of the layout process will always follow the same steps:

-   Get some example data.
-   Look if either the incoming or outgoing data format is "all data for a record is in a single row" or not, this determines if you later use `rowrecs_to_blocks_spec()` (specifying data moving from single rows to arbitrary block records), `blocks_to_rowrecs_spec()` (specifying data moving from block records to single rows), or `layout_specification()` (a general block record to block record case). Data transforms that take rows to rows are not so much re-shapings as combination of renames, selections, and column calculations.
-   Identify which columns that tell us which sets of rows are records, which we call the "record keys."
-   Draw the shape of the incoming record without the record keys.
-   Draw the shape of the outgoing record without the record keys.
-   Combine the above information as a one of the above data layout transform specifications.
-   Print the transform to confirm it is what you want.
-   Apply the transform.

This may seem like a lot of steps, but it is only because we are taking the problems very slowly. The thing to note is there is hopefully not a lot of additional problem solving to apply the `cdata` methodology. Usually when you need to transform data you are in the middle of some other more important task, so you want delegate the details of how the transform is implemented. In `cdata` the user only specifies the data transform, implementation is left to the `cdata` package. With `cdata` the user is not asked to perform additional puzzle solving to guess sequence of operators that may implement the desired transform. The `cdata` solution pattern is always the same, which can help mastering it.

Also with `cdata` record layout transforms are simple `R` objects with detailed `print()` methods- so they are convenient to save re-used later.

We will work some examples with the hope that practice brings familiarity. The examples for this note are all of the demo-examples from Examples from [tidyr/demo/](https://github.com/tidyverse/tidyr/tree/master/demo).

Example 1
---------

(From: <https://github.com/tidyverse/tidyr/blob/master/demo/dadmom.R>.)

Full data not available, as link is dead. From <https://stats.idre.ucla.edu/stata/modules/reshaping-data-wide-to-long/> we can get the following data extract. From this source we can extract the question:

``` r
# convert from this format
dadmomw <- wrapr::build_frame(
   "famid"  , "named", "incd", "namem", "incm" |
     1      , "Bill" , 30000 , "Bess" , 15000  |
     2      , "Art"  , 22000 , "Amy"  , 18000  |
     3      , "Paul" , 25000 , "Pat"  , 50000  )

# to this format
dadmomt <- wrapr::build_frame(
   "famid"  , "dadmom", "name", "inc" |
     1      , "d"     , "Bill", 30000 |
     1      , "m"     , "Bess", 15000 |
     2      , "d"     , "Art" , 22000 |
     2      , "m"     , "Amy" , 18000 |
     3      , "d"     , "Paul", 25000 |
     3      , "m"     , "Pat" , 50000 )
```

The `cdata` solution is given here, notice the incoming record is a single row so we don't have to specify it.

``` r
library("cdata")

# how to find records
recordKeys <- "famid"

# specify the outgoing record shape
outgoing_record <- wrapr::qchar_frame(
   "dadmom"  , "name", "inc" |
     "d"     , named , incd |
     "m"     , namem , incm )

# put it all together into a transform
transform <- rowrecs_to_blocks_spec(
  outgoing_record,
  recordKeys = recordKeys)

# confirm we have the right transform
print(transform)
```

    ## {
    ##  row_record <- wrapr::qchar_frame(
    ##    "famid"  , "named", "namem", "incd", "incm" |
    ##      .      , named  , namem  , incd  , incm   )
    ##  row_keys <- c('famid')
    ## 
    ##  # becomes
    ## 
    ##  block_record <- wrapr::qchar_frame(
    ##    "famid"  , "dadmom", "name", "inc" |
    ##      .      , "d"     , named , incd  |
    ##      .      , "m"     , namem , incm  )
    ##  block_keys <- c('famid', 'dadmom')
    ## 
    ##  # args: c(checkNames = TRUE, checkKeys = FALSE, strict = FALSE, allow_rqdatatable = TRUE)
    ## }

``` r
# apply the transform
dadmomw %.>% 
  transform %.>%
  knitr::kable(.)
```

|  famid| dadmom | name |    inc|
|------:|:-------|:-----|------:|
|      1| d      | Bill |  30000|
|      1| m      | Bess |  15000|
|      2| d      | Art  |  22000|
|      2| m      | Amy  |  18000|
|      3| d      | Paul |  25000|
|      3| m      | Pat  |  50000|

Notice we take the column names from the incoming row-record and use them as cell-names in the outgoing record; this is how we show where the data goes.

Also notice the `print()` method fully documents what columns are expected and the intent of the transform. The `print()` method is using a convention that quoted entities are values we know (values that specify column names, or keys that describe block record structure), and un-quoted entities are values we expect to be in the record or "dot" for row-keys (keys that specify which sets of rows are in the same block record).

(The original "`tidyr`" solution may have been on different data, or at least appears to not work on the above data. Because the `tidyr` solution is multiple independent transform steps: it does not have a single print method that can document intent. So we can't know we if we have the correct data for the `tidyr` example, and hence can't know if the `tidyr` example is correct or not.)

Example 2
---------

(From: <http://stackoverflow.com/questions/15668870/>, [https://github.com/tidyverse/tidyr/blob/master/demo/so-15668870.R](http://stackoverflow.com/questions/15668870/).)

The original question was:

    I want to reshape a wide format dataset that has multiple tests which are measured at 3 time points:

       ID   Test Year   Fall Spring Winter
        1   1   2008    15      16      19
        1   1   2009    12      13      27
        1   2   2008    22      22      24
        1   2   2009    10      14      20
     ...

    into a data set that separates the tests by column but converts the measurement time into long format, for each of the new columns like this:

        ID  Year    Time        Test1 Test2
        1   2008    Fall        15      22
        1   2008    Spring      16      22
        1   2008    Winter      19      24
        1   2009    Fall        12      10
        1   2009    Spring      13      14
        1   2009    Winter      27      20
     ...

    I have unsuccessfully tried to use reshape and melt. Existing posts address transforming to single column outcome.

To solve this using `cdata`:

``` r
library("cdata")

# how to find records
recordKeys <- c("ID", "Year")

# specify the incoming record shape
incoming_record <- wrapr::qchar_frame(
  "Test"  , "Fall", "Spring", "Winter" |
    "1"   , F1    , S1      , W1       |
    "2"   , F2    , S2      , W2       )

# specify the outgoing record shape
outgoing_record <- wrapr::qchar_frame(
  "Semester" , "Test1", "Test2" |
    "Fall"   , F1,      F2      |
    "Spring" , S1,      S2      |
    "Winter" , W1,      W2      )

# put it all together into a transform
transform <- layout_specification(
  incoming_shape = incoming_record,
  outgoing_shape = outgoing_record,
  recordKeys = recordKeys)

# confirm we have the right transform
print(transform)
```

    ## {
    ##  in_record <- wrapr::qchar_frame(
    ##    "ID"  , "Year", "Test", "Fall", "Spring", "Winter" |
    ##      .   , .     , "1"   , F1    , S1      , W1       |
    ##      .   , .     , "2"   , F2    , S2      , W2       )
    ##  in_keys <- c('ID', 'Year', 'Test')
    ## 
    ##  # becomes
    ## 
    ##  out_record <- wrapr::qchar_frame(
    ##    "ID"  , "Year", "Semester", "Test1", "Test2" |
    ##      .   , .     , "Fall"    , F1     , F2      |
    ##      .   , .     , "Spring"  , S1     , S2      |
    ##      .   , .     , "Winter"  , W1     , W2      )
    ##  out_keys <- c('ID', 'Year', 'Semester')
    ## 
    ##  # args: c(checkNames = TRUE, checkKeys = TRUE, strict = FALSE, allow_rqdatatable = FALSE)
    ## }

``` r
# example data
grades <- wrapr::build_frame(
   "ID"  , "Test", "Year", "Fall", "Spring", "Winter" |
     1   , 1     , 2008  , 15    , 16      , 19       |
     1   , 1     , 2009  , 12    , 13      , 27       |
     1   , 2     , 2008  , 22    , 22      , 24       |
     1   , 2     , 2009  , 10    , 14      , 20       |
     2   , 1     , 2008  , 12    , 13      , 25       |
     2   , 1     , 2009  , 16    , 14      , 21       |
     2   , 2     , 2008  , 13    , 11      , 29       |
     2   , 2     , 2009  , 23    , 20      , 26       |
     3   , 1     , 2008  , 11    , 12      , 22       |
     3   , 1     , 2009  , 13    , 11      , 27       |
     3   , 2     , 2008  , 17    , 12      , 23       |
     3   , 2     , 2009  , 14    ,  9      , 31       )

# apply the transform
grades %.>% 
  transform %.>%
  knitr::kable(.)
```

|   ID|  Year| Semester |  Test1|  Test2|
|----:|-----:|:---------|------:|------:|
|    1|  2008| Fall     |     15|     22|
|    1|  2008| Spring   |     16|     22|
|    1|  2008| Winter   |     19|     24|
|    1|  2009| Fall     |     12|     10|
|    1|  2009| Spring   |     13|     14|
|    1|  2009| Winter   |     27|     20|
|    2|  2008| Fall     |     12|     13|
|    2|  2008| Spring   |     13|     11|
|    2|  2008| Winter   |     25|     29|
|    2|  2009| Fall     |     16|     23|
|    2|  2009| Spring   |     14|     20|
|    2|  2009| Winter   |     21|     26|
|    3|  2008| Fall     |     11|     17|
|    3|  2008| Spring   |     12|     12|
|    3|  2008| Winter   |     22|     23|
|    3|  2009| Fall     |     13|     14|
|    3|  2009| Spring   |     11|      9|
|    3|  2009| Winter   |     27|     31|

Example 3
=========

(From: <https://github.com/tidyverse/tidyr/blob/master/demo/so-16032858.R> , [r](http://stackoverflow.com/questions/16032858).)

Question: given data such as below how does one move treatment and control values for each individual into columns? Or how does take `a` to `b`?

``` r
a <- wrapr::build_frame(
   "Ind"   , "Treatment", "value" |
     "Ind1", "Treat"    , 1       |
     "Ind2", "Treat"    , 2       |
     "Ind1", "Cont"     , 3       |
     "Ind2", "Cont"     , 4       )

b <- wrapr::build_frame(
   "Ind"   , "Treat" , "Cont"|
     "Ind1", 1       , 3     |
     "Ind2", 2       , 4     )
```

The `cdata` solution is as follows.

``` r
library("cdata")

# how to find records
recordKeys <- "Ind"

# specify the incoming record shape
incoming_record <- wrapr::qchar_frame(
   "Treatment"  , "value" |
    "Treat"     , Treat   |
    "Cont"      , Cont    )


# put it all together into a transform
transform <- blocks_to_rowrecs_spec(
  incoming_record,
  recordKeys = recordKeys)

# confirm we have the right transform
print(transform)
```

    ## {
    ##  block_record <- wrapr::qchar_frame(
    ##    "Ind"  , "Treatment", "value" |
    ##      .    , "Treat"    , Treat   |
    ##      .    , "Cont"     , Cont    )
    ##  block_keys <- c('Ind', 'Treatment')
    ## 
    ##  # becomes
    ## 
    ##  row_record <- wrapr::qchar_frame(
    ##    "Ind"  , "Treat", "Cont" |
    ##      .    , Treat  , Cont   )
    ##  row_keys <- c('Ind')
    ## 
    ##  # args: c(checkNames = TRUE, checkKeys = TRUE, strict = FALSE, allow_rqdatatable = FALSE)
    ## }

``` r
# apply the transform
a %.>% 
  transform %.>%
  knitr::kable(.)
```

| Ind  |  Treat|  Cont|
|:-----|------:|-----:|
| Ind1 |      1|     3|
| Ind2 |      2|     4|

Example 4
=========

(From: <https://github.com/tidyverse/tidyr/blob/master/demo/so-17481212.R> , <http://stackoverflow.com/questions/17481212>.)

Convert data that has one different observation for each column to a data that has all observations in rows. That is take `a` to `b` in the following.

``` r
a <- wrapr::build_frame(
   "Name"   , "50", "100", "150", "200", "250", "300", "350" |
     "Carla", 1.2 , 1.8  , 2.2  , 2.3  , 3    , 2.5  , 1.8   |
     "Mace" , 1.5 , 1.1  , 1.9  , 2    , 3.6  , 3    , 2.5   )

b <- wrapr::build_frame(
   "Name"   , "Time", "Score" |
     "Carla", 50    , 1.2     |
     "Carla", 100   , 1.8     |
     "Carla", 150   , 2.2     |
     "Carla", 200   , 2.3     |
     "Carla", 250   , 3       |
     "Carla", 300   , 2.5     |
     "Carla", 350   , 1.8     |
     "Mace" , 50    , 1.5     |
     "Mace" , 100   , 1.1     |
     "Mace" , 150   , 1.9     |
     "Mace" , 200   , 2       |
     "Mace" , 250   , 3.6     |
     "Mace" , 300   , 3       |
     "Mace" , 350   , 2.5     )
```

The `cdata` solution is as before, but as we have a large number of columns we will us a helper function to specify the transform.

``` r
library("cdata")
library("rqdatatable")
```

    ## Loading required package: rquery

``` r
# how to find records
recordKeys <- "Name"

# specify the outgoing record shape, using a helper function
outgoing_record <- build_unpivot_control(
  nameForNewKeyColumn = "Time",
  nameForNewValueColumn = "Score",
  columnsToTakeFrom = setdiff(colnames(a), recordKeys))

# put it all together into a transform
transform <- rowrecs_to_blocks_spec(
  outgoing_record,
  recordKeys = recordKeys)

# confirm we have the right transform
print(transform)
```

    ## {
    ##  row_record <- wrapr::qchar_frame(
    ##    "Name"  , "50", "100", "150", "200", "250", "300", "350" |
    ##      .     , 50  , 100  , 150  , 200  , 250  , 300  , 350   )
    ##  row_keys <- c('Name')
    ## 
    ##  # becomes
    ## 
    ##  block_record <- wrapr::qchar_frame(
    ##    "Name"  , "Time", "Score" |
    ##      .     , "50"  , 50      |
    ##      .     , "100" , 100     |
    ##      .     , "150" , 150     |
    ##      .     , "200" , 200     |
    ##      .     , "250" , 250     |
    ##      .     , "300" , 300     |
    ##      .     , "350" , 350     )
    ##  block_keys <- c('Name', 'Time')
    ## 
    ##  # args: c(checkNames = TRUE, checkKeys = FALSE, strict = FALSE, allow_rqdatatable = TRUE)
    ## }

``` r
# apply the transform
a %.>% 
  transform %.>%
  extend(., Time = as.numeric(Time)) %.>%
  orderby(., c("Name", "Time")) %.>%
  knitr::kable(.)
```

| Name  |  Time|  Score|
|:------|-----:|------:|
| Carla |    50|    1.2|
| Carla |   100|    1.8|
| Carla |   150|    2.2|
| Carla |   200|    2.3|
| Carla |   250|    3.0|
| Carla |   300|    2.5|
| Carla |   350|    1.8|
| Mace  |    50|    1.5|
| Mace  |   100|    1.1|
| Mace  |   150|    1.9|
| Mace  |   200|    2.0|
| Mace  |   250|    3.6|
| Mace  |   300|    3.0|
| Mace  |   350|    2.5|

Example 5
=========

(From: <https://github.com/tidyverse/tidyr/blob/master/demo/so-9684671.R> , <http://stackoverflow.com/questions/9684671>.)

Convert from `a` to `b`.

``` r
a <- wrapr::build_frame(
   "id"     , "trt", "work.T1", "play.T1", "talk.T1", "work.T2", "play.T2", "talk.T2" |
     "x1.01", "tr" , 0.6517   , 0.8647   , 0.5356   , 0.2755   , 0.3543   , 0.03189   |
     "x1.02", "cnt", 0.5677   , 0.6154   , 0.09309  , 0.2289   , 0.9364   , 0.1145    )

b <- wrapr::build_frame(
   "id"     , "trt", "time", "play", "talk" , "work" |
     "x1.01", "tr" , "T1"  , 0.8647, 0.5356 , 0.6517 |
     "x1.01", "tr" , "T2"  , 0.3543, 0.03189, 0.2755 |
     "x1.02", "cnt", "T1"  , 0.6154, 0.09309, 0.5677 |
     "x1.02", "cnt", "T2"  , 0.9364, 0.1145 , 0.2289 )
```

`cdata` solution.

``` r
library("cdata")
library("rqdatatable")

# how to find records
recordKeys <- c("id", "trt")

# specify the outgoing record shape, using a helper function
outgoing_record <- wrapr::qchar_frame(
    "time"  , "play" , "talk" , "work"  |
    "T1"    , play.T1, talk.T1, work.T1 |
    "T2"    , play.T2, talk.T2, work.T2 )

# put it all together into a transform
transform <- rowrecs_to_blocks_spec(
  outgoing_record,
  recordKeys = recordKeys)

# confirm we have the right transform
print(transform)
```

    ## {
    ##  row_record <- wrapr::qchar_frame(
    ##    "id"  , "trt", "play.T1", "play.T2", "talk.T1", "talk.T2", "work.T1", "work.T2" |
    ##      .   , .    , play.T1  , play.T2  , talk.T1  , talk.T2  , work.T1  , work.T2   )
    ##  row_keys <- c('id', 'trt')
    ## 
    ##  # becomes
    ## 
    ##  block_record <- wrapr::qchar_frame(
    ##    "id"  , "trt", "time", "play" , "talk" , "work"  |
    ##      .   , .    , "T1"  , play.T1, talk.T1, work.T1 |
    ##      .   , .    , "T2"  , play.T2, talk.T2, work.T2 )
    ##  block_keys <- c('id', 'trt', 'time')
    ## 
    ##  # args: c(checkNames = TRUE, checkKeys = FALSE, strict = FALSE, allow_rqdatatable = TRUE)
    ## }

``` r
# apply the transform
a %.>% 
  transform %.>%
  orderby(., recordKeys) %.>%
  knitr::kable(.)
```

| id    | trt | time |    play|     talk|    work|
|:------|:----|:-----|-------:|--------:|-------:|
| x1.01 | tr  | T1   |  0.8647|  0.53560|  0.6517|
| x1.01 | tr  | T2   |  0.3543|  0.03189|  0.2755|
| x1.02 | cnt | T1   |  0.6154|  0.09309|  0.5677|
| x1.02 | cnt | T2   |  0.9364|  0.11450|  0.2289|

Conclusion
----------

Notice how in all cases we can solve the record layout transform by insisting an an example of the incoming and outgoing records, knowing which columns specify records (`recordKeys`) and then copying column names from the incoming/outgoing records into cells of the outgoing/incoming records. Really all one is doing when using `cdata` is formalizing the transform "ask" into a machine readable example.

To make your own solutions, we suggest just using one of these solutions as a template

`cdata` also supplies a number of convenience functions for specifying common transforms, or one can build up transformations descriptions using code as shown in the grid scatter-plot example [here](https://github.com/WinVector/cdata).

We also note the value of being able to print and review the bulk of transform, as it documents expected incoming data columns and interior block record key values.

The source-code for [this note](https://github.com/WinVector/cdata/blob/master/extras/exercises/exercises_5.md) can be found [here](https://github.com/WinVector/cdata/blob/master/extras/exercises/exercises_5.Rmd).