
<!-- README.md is generated from README.Rmd. Please edit that file -->
The `cdata` package is a demonstration of the ["coordinatized data" theory](http://winvector.github.io/FluidData/RowsAndColumns.html) and includes an implementation of the ["fluid data" methodology](http://winvector.github.io/FluidData/FluidData.html). The recommended tutorial is: [Fluid data reshaping with cdata](http://winvector.github.io/FluidData/FluidDataReshapingWithCdata.html). We also have a [short free cdata screencast](https://youtu.be/4cYbP3kbc0k) (and another example can be found [here](http://winvector.github.io/FluidData/DataWranglingAtScale.html)).

![](https://raw.githubusercontent.com/WinVector/cdata/master/tools/cdata.png)

Briefly: `cdata` supplies data transform operators that:

-   Work on local data or with any `DBI` data source.
-   Are powerful generalizations of the operators commonly called `pivot` and `un-pivot`.

A quick example: plot iris petal and sepal dimensions in a faceted graph.

``` r
iris <- data.frame(iris)

library("ggplot2")
library("cdata")

#
# build a control table with a "key column" dimension
# and "value columns" Length and Width
#
controlTable <- wrapr::qchar_frame(
   dimension, Length      , Width       |
   Petal    , Petal.Length, Petal.Width |
   Sepal    , Sepal.Length, Sepal.Width )

# do the unpivot to convert the row records to block records
iris_aug <- rowrecs_to_blocks(
  iris,
  controlTable,
  columnsToCopy = c("Species"))


ggplot(iris_aug, aes(x=Length, y=Width)) +
  geom_point(aes(color=Species, shape=Species)) + 
  facet_wrap(~dimension, labeller = label_both, scale = "free") +
  ggtitle("Iris dimensions") +  scale_color_brewer(palette = "Dark2")
```

![](tools/README_files/figure-markdown_github/ex0-1.png)

An example of using `cdata` to creat a scatterplot matrix, or pair plot:

``` r
iris <- data.frame(iris)

library("ggplot2")
library("cdata")

# declare our columns of interest
meas_vars <- qc(Sepal.Length, Sepal.Width, Petal.Length, Petal.Width)
category_variable <- "Species"

# build a control with all pairs of variables as value columns
# and pair_key as the key column
controlTable <- data.frame(expand.grid(meas_vars, meas_vars, 
                                       stringsAsFactors = FALSE))
# name the value columns value1 and value2
colnames(controlTable) <- qc(value1, value2)
# insert first, or key column
controlTable <- cbind(
  data.frame(pair_key = paste(controlTable[[1]], controlTable[[2]]),
             stringsAsFactors = FALSE),
  controlTable)


# do the unpivot to convert the row records to multiple block records
iris_aug <- rowrecs_to_blocks(
  iris,
  controlTable,
  columnsToCopy = category_variable)

# unpack the key column into two variable keys for the facet_grid
splt <- strsplit(iris_aug$pair_key, split = " ", fixed = TRUE)
iris_aug$v1 <- vapply(splt, function(si) si[[1]], character(1))
iris_aug$v2 <- vapply(splt, function(si) si[[2]], character(1))


ggplot(iris_aug, aes(x=value1, y=value2)) +
  geom_point(aes_string(color=category_variable, shape=category_variable)) + 
  facet_grid(v2~v1, labeller = label_both, scale = "free") +
  ggtitle("Iris dimensions") +
  scale_color_brewer(palette = "Dark2") +
  ylab(NULL) + 
  xlab(NULL)
```

![](tools/README_files/figure-markdown_github/ex0_1-1.png)

A quick database example:

``` r
library("cdata")
library("rquery")

use_spark <- TRUE

if(use_spark) {
  my_db <- sparklyr::spark_connect(version='2.2.0', 
                                   master = "local")
} else {
  my_db <- DBI::dbConnect(RSQLite::SQLite(),
                          ":memory:")
}



# pivot example
d <- wrapr::build_frame(
   "meas", "val" |
   "AUC" , 0.6   |
   "R2"  , 0.2   )
DBI::dbWriteTable(my_db,
                  'd',
                  d,
                  temporary = TRUE)
rstr(my_db, 'd')
```

    ## table `d` spark_connection spark_shell_connection DBIConnection 
    ##  nrow: 2 
    ## 'data.frame':    2 obs. of  2 variables:
    ##  $ meas: chr  "AUC" "R2"
    ##  $ val : num  0.6 0.2

``` r
td <- db_td(my_db, "d")
td
```

    ## [1] "table(`d`; meas, val)"

``` r
cT <- td %.>%
  build_pivot_control(.,
                      columnToTakeKeysFrom= 'meas',
                      columnToTakeValuesFrom= 'val') %.>%
  execute(my_db, .)
print(cT)
```

    ##   meas val
    ## 1  AUC AUC
    ## 2   R2  R2

``` r
tab <- td %.>%
  blocks_to_rowrecs(.,
                    keyColumns = NULL,
                    controlTable = cT,
                    temporary = FALSE) %.>%
  materialize(my_db, .)

print(tab)
```

    ## [1] "table(`rquery_mat_13782577874446676343_0000000000`; AUC, R2)"

``` r
rstr(my_db, tab)
```

    ## table `rquery_mat_13782577874446676343_0000000000` spark_connection spark_shell_connection DBIConnection 
    ##  nrow: 1 
    ## 'data.frame':    1 obs. of  2 variables:
    ##  $ AUC: num 0.6
    ##  $ R2 : num 0.2

``` r
if(use_spark) {
  sparklyr::spark_disconnect(my_db)
} else {
  DBI::dbDisconnect(my_db)
}
```

Install via CRAN:

``` r
install.packages("cdata")
```

Or from GitHub using devtools:

``` r
devtools::install_github("WinVector/cdata")
```

Note: `cdata` is targeted at data with "tame column names" (column names that are valid both in databases, and as `R` unquoted variable names) and basic types (column values that are simple `R` types such as `character`, `numeric`, `logical`, and so on).
