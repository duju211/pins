pins: Track, Discover and Share Datasets
================

# pins

[![Build
Status](https://travis-ci.org/javierluraschi/pins.svg?branch=master)](https://travis-ci.org/javierluraschi/pins)
[![CRAN\_Status\_Badge](https://www.r-pkg.org/badges/version/pins)](https://cran.r-project.org/package=pins)

  - **Track** local and remote datasets from Python and R using `pin()`
    and `get_pin()`.
  - **Discover** new datasets from packages, online and your
    organization using `find_pin()`.
  - **Share** datasets with your team, or the world, using customizable
    boards through `use_board()`.
  - **Extend** storage locations with custom boards, you decide where
    your data lives.

# R

## Installation

You can install `pins` using the `remotes` package:

``` r
install.packages("remotes")
remotes::install_github("rstudio/pins")
```

## Tracking Datasets

It’s easy to track local datasets using `pin()` and `get_pin()`. In
addition, you can retrive remote datasets using `DBI` or pin remote
datasets without retrieving them using `dplyr`.

### Local Datasets

You can track your datasets privately by pinning with `pin()`.

``` r
library(dplyr, warn.conflicts = FALSE)
library(pins)

iris %>%
  filter(Sepal.Width < 3, Petal.Width < 1) %>%
  pin("iris-small-width", "A subset of 'iris' with only small widths.")
```

    ## # A tibble: 2 x 5
    ##   Sepal.Length Sepal.Width Petal.Length Petal.Width Species
    ##          <dbl>       <dbl>        <dbl>       <dbl> <fct>  
    ## 1          4.4         2.9          1.4         0.2 setosa 
    ## 2          4.5         2.3          1.3         0.3 setosa

You can then retrieve them back with `get_pin()`.

``` r
get_pin("iris-small-width")
```

    ## # A tibble: 2 x 5
    ##   Sepal.Length Sepal.Width Petal.Length Petal.Width Species
    ##          <dbl>       <dbl>        <dbl>       <dbl> <fct>  
    ## 1          4.4         2.9          1.4         0.2 setosa 
    ## 2          4.5         2.3          1.3         0.3 setosa

A pin is a tool to help you track your datasets to easily fetch results
from past data analysis sessions.

For instance, once a dataset is tidy, you are likely to reuse it several
times. You might also have a past analysis in GitHub, but you might not
want to clone, install dependencies and rerun your code just to access
your dataset. Another use case is to cross-join between datasets to
analyse across multiple projects or help you remember which datasets
you’ve used in the past.

### Remote Datasets

Some datasets are stored in datasets which you usually access with `DBI`
or `dplyr`. For instance, you might want to access a public dataset
stored in
`bigrquery`:

``` r
con <- DBI::dbConnect(bigrquery::bigquery(), project = bq_project, dataset = bq_dataset)
```

Which we can analyze with `DBI` and then pin the results locally:

``` r
DBI::dbGetQuery(con, "
  SELECT score, count(*) as n
  FROM (SELECT 10 * floor(score/10) as score FROM `bigquery-public-data.hacker_news.full`)
  GROUP BY score") %>%
  pin("hacker-news-scores", "Hacker News scores grouped by tens.")
```

However, you can only use `DBI` when you can fetch all the data back
into R, this is not feasible in many cases. Instead, when using `dplyr`,
you can pin large datasets and transform them without having to fetch
any data at all.

Lets pin the entire dataset using `dplyr`:

``` r
tbl(con, "bigquery-public-data.hacker_news.full") %>%
  pin("hacker-news-full")
```

    ## # Source:   SQL [?? x 14]
    ## # Database: BigQueryConnection
    ##    by    score   time timestamp           title type  url   text   parent
    ##    <chr> <int>  <int> <dttm>              <chr> <chr> <chr> <chr>   <int>
    ##  1 shel…     2 1.29e9 2010-10-10 22:15:54 Runn… story http… ""    NA     
    ##  2 Gibb…    NA 1.55e9 2019-03-19 20:28:46 ""    comm… ""    No i…  1.94e7
    ##  3 bcook    NA 1.46e9 2016-04-19 10:20:24 ""    comm… ""    Whic…  1.15e7
    ##  4 matt…    NA 1.49e9 2017-03-01 20:30:46 ""    comm… ""    I ho…  1.38e7
    ##  5 scho…     1 1.38e9 2013-09-07 15:01:19 Yaho… story http… ""    NA     
    ##  6 Fice     NA 1.49e9 2017-03-16 23:55:05 ""    comm… ""    Nota…  1.39e7
    ##  7 dark…    NA 1.36e9 2013-04-01 14:45:48 ""    comm… ""    &#62…  5.47e6
    ##  8 arav…    NA 1.35e9 2012-11-23 08:01:24 ""    comm… ""    You …  4.82e6
    ##  9 robr…    NA 1.49e9 2017-04-08 10:32:21 ""    comm… ""    I&#x…  1.41e7
    ## 10 eries    NA 1.43e9 2015-04-14 17:20:46 ""    comm… ""    Than…  9.37e6
    ## # … with more rows, and 5 more variables: deleted <lgl>, dead <lgl>,
    ## #   descendants <int>, id <int>, ranking <int>

This works well if you provide the connection, after your R session gets
restarted, you would have to provide a connection yourself before
retrieving the
pin:

``` r
con <- DBI::dbConnect(bigrquery::bigquery(), project = bq_project, dataset = bq_dataset)
get_pin("hacker-news-full")
```

This is acceptable but not ideal – it’s hard to remember what connection
to use for each dataset. So instead, pin a
connection:

``` r
con <- pin(~DBI::dbConnect(bigrquery::bigquery(), project = bq_project, dataset = bq_dataset), "bigquery")
```

Then pin your dataset as you would usually would,

``` r
tbl(con, "bigquery-public-data.hacker_news.full") %>%
  pin("hacker-news-full", "The Hacker News dataset in Google BigQuery.")
```

From now on, after restarting your R session and retrieving the pin, the
pin will initialize the connection before retrieving a `dplyr` reference
to it with `pin("hacker-news-full")`.

Which in turn, allows you to further process the datset using `dplyr`
and pin additional remote datasets.

``` r
get_pin("hacker-news-full") %>%
  transmute(score = 10 * floor(score/10)) %>%
  group_by(score) %>%
  summarize(n = n()) %>%
  filter(score < 2000) %>%
  pin("hacker-news-scores")
```

You can then use this `dplyr` pin to process data further; for instance,
by visualizing it with ease:

``` r
library(ggplot2)

get_pin("hacker-news-scores") %>%
  ggplot() +
    geom_bar(aes(x = score, y = n), stat="identity") +
    scale_y_log10() + theme_light()
```

![](tools/readme/remote-dplyr-plot-1.png)<!-- -->

You can also cache this dataset locally by running `collect()` on the
pin and then re-pinning it with `pin()`.

## Discovering Datasets

The `pins` package can help you discover interesting datasets, currently
it searches datasets inside CRAN packages but we are planning to extend
this further.

You can search datasets that contain “seattle” in their description or
name as follows:

``` r
find_pin("seattle")
```

    ## # A tibble: 4 x 4
    ##   name               description                               type  board 
    ##   <chr>              <chr>                                     <chr> <chr> 
    ## 1 hpiR_seattle_sales Seattle Home Sales from hpiR package.     table packa…
    ## 2 microsynth_seattl… Data for a crime intervention in Seattle… table packa…
    ## 3 vegawidget_data_s… Example dataset: Seattle daily weather f… table packa…
    ## 4 vegawidget_data_s… Example dataset: Seattle hourly temperat… table packa…

You can the retrieve a specific dataset with `get_pin()`:

``` r
get_pin("hpiR_seattle_sales")
```

    ## # A tibble: 43,313 x 16
    ##    pinx  sale_id sale_price sale_date  use_type  area lot_sf  wfnt
    ##    <chr> <chr>        <int> <date>     <chr>    <int>  <int> <dbl>
    ##  1 ..00… 2013..…     289000 2013-02-06 sfr         79   9295     0
    ##  2 ..00… 2013..…     356000 2013-07-11 sfr         18   6000     0
    ##  3 ..00… 2010..…     333500 2010-12-29 sfr         79   7200     0
    ##  4 ..00… 2016..…     577200 2016-03-17 sfr         79   7200     0
    ##  5 ..00… 2012..…     237000 2012-05-02 sfr         79   5662     0
    ##  6 ..00… 2014..…     347500 2014-03-11 sfr         79   5830     0
    ##  7 ..00… 2012..…     429000 2012-09-20 sfr         18  12700     0
    ##  8 ..00… 2015..…     653295 2015-07-21 sfr         79   7000     0
    ##  9 ..00… 2014..…     427650 2014-02-19 townhou…    79   3072     0
    ## 10 ..00… 2015..…     488737 2015-03-19 townhou…    79   3072     0
    ## # … with 43,303 more rows, and 8 more variables: bldg_grade <int>,
    ## #   tot_sf <int>, beds <int>, baths <dbl>, age <int>, eff_age <int>,
    ## #   longitude <dbl>, latitude <dbl>

## Sharing Datasets

`pins` supports shared storage locations using boards. A board is a
remote location for you to share pins with your team privately, or with
the world, publicly. Use `use_board()` to choose a board, currently
`database` and `arrow` boards are supported; however, `pins` provide an
extensible API you can use to store pins anywhere.

### Arrow

In order to share datasets with other programming languages, an `arrow`
board can be used. This board uses Apache Arrow to share datasets across
R and Python and can be easily activated as follows:

``` r
use_board("arrow")
```

The you can pin and retrieve pins as usual.

``` r
mtcars %>% pin("cars")
```

    ## # A tibble: 32 x 11
    ##      mpg   cyl  disp    hp  drat    wt  qsec    vs    am  gear  carb
    ##    <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl>
    ##  1  21       6  160    110  3.9   2.62  16.5     0     1     4     4
    ##  2  21       6  160    110  3.9   2.88  17.0     0     1     4     4
    ##  3  22.8     4  108     93  3.85  2.32  18.6     1     1     4     1
    ##  4  21.4     6  258    110  3.08  3.22  19.4     1     0     3     1
    ##  5  18.7     8  360    175  3.15  3.44  17.0     0     0     3     2
    ##  6  18.1     6  225    105  2.76  3.46  20.2     1     0     3     1
    ##  7  14.3     8  360    245  3.21  3.57  15.8     0     0     3     4
    ##  8  24.4     4  147.    62  3.69  3.19  20       1     0     4     2
    ##  9  22.8     4  141.    95  3.92  3.15  22.9     1     0     4     2
    ## 10  19.2     6  168.   123  3.92  3.44  18.3     1     0     4     4
    ## # … with 22 more rows

``` r
get_pin("cars")
```

    ## # A tibble: 32 x 11
    ##      mpg   cyl  disp    hp  drat    wt  qsec    vs    am  gear  carb
    ##    <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl>
    ##  1  21       6  160    110  3.9   2.62  16.5     0     1     4     4
    ##  2  21       6  160    110  3.9   2.88  17.0     0     1     4     4
    ##  3  22.8     4  108     93  3.85  2.32  18.6     1     1     4     1
    ##  4  21.4     6  258    110  3.08  3.22  19.4     1     0     3     1
    ##  5  18.7     8  360    175  3.15  3.44  17.0     0     0     3     2
    ##  6  18.1     6  225    105  2.76  3.46  20.2     1     0     3     1
    ##  7  14.3     8  360    245  3.21  3.57  15.8     0     0     3     4
    ##  8  24.4     4  147.    62  3.69  3.19  20       1     0     4     2
    ##  9  22.8     4  141.    95  3.92  3.15  22.9     1     0     4     2
    ## 10  19.2     6  168.   123  3.92  3.44  18.3     1     0     4     4
    ## # … with 22 more rows

While the functionality is similar, pins are stored using the Apache
Arrow format which can be easily read from Python and many other
languages.

### Databases

We can reuse our `bigrquery` connection to define a database-backed
shared board,

``` r
use_board("database", con)
```

Which we can also use to pin a dataset,

``` r
pin(iris, "iris", "The entire 'iris' dataset.")
```

    ## # A tibble: 150 x 5
    ##    Species    Petal_Width Petal_Length Sepal_Width Sepal_Length
    ##    <chr>            <dbl>        <dbl>       <dbl>        <dbl>
    ##  1 versicolor         1.1          3           2.5          5.1
    ##  2 versicolor         1            3.5         2            5  
    ##  3 versicolor         1            3.5         2.6          5.7
    ##  4 versicolor         1            4           2.2          6  
    ##  5 versicolor         1.2          4           2.6          5.8
    ##  6 versicolor         1.3          4           2.3          5.5
    ##  7 versicolor         1.3          4           2.8          6.1
    ##  8 versicolor         1.3          4           2.5          5.5
    ##  9 versicolor         1.5          4.5         3.2          6.4
    ## 10 versicolor         1.5          4.5         3            5.6
    ## # … with 140 more rows

find pins,

``` r
find_pin()
```

    ## # A tibble: 6 x 4
    ##   name              description                              type   board  
    ##   <chr>             <chr>                                    <chr>  <chr>  
    ## 1 cars              ""                                       table  arrow  
    ## 2 iris              The entire 'iris' dataset.               table  databa…
    ## 3 iris-small-width  A subset of 'iris' with only small widt… table  local  
    ## 4 bigquery          ""                                       formu… local  
    ## 5 hacker-news-full  The Hacker News dataset in Google BigQu… dbplyr local  
    ## 6 hacker-news-scor… ""                                       dbplyr local

and retrieve shared datasets.

``` r
get_pin("iris")
```

    ## # A tibble: 150 x 5
    ##    Species    Petal_Width Petal_Length Sepal_Width Sepal_Length
    ##    <chr>            <dbl>        <dbl>       <dbl>        <dbl>
    ##  1 versicolor         1.1          3           2.5          5.1
    ##  2 versicolor         1            3.5         2            5  
    ##  3 versicolor         1            3.5         2.6          5.7
    ##  4 versicolor         1            4           2.2          6  
    ##  5 versicolor         1.2          4           2.6          5.8
    ##  6 versicolor         1.3          4           2.3          5.5
    ##  7 versicolor         1.3          4           2.8          6.1
    ##  8 versicolor         1.3          4           2.5          5.5
    ##  9 versicolor         1.5          4.5         3.2          6.4
    ## 10 versicolor         1.5          4.5         3            5.6
    ## # … with 140 more rows

### Connections

Connections can also be pinned to shared boards; however, you should pin
them using a proper connection object, not an R
formula:

``` r
con <- pin_connection("bigquery", driver = "bigrquery::bigquery", project = bq_project, dataset = bq_dataset)
```

Other packages that don’t use `DBI` connections, like `sparklyr`, can
use an explicit `initializer` function:

``` r
sc <- pin_connection(
  "spark-local",
  "sparklyr::spark_connect",
  master = "local",
  config = list("sparklyr.shell.driver-memory" = "4g")
)
```

**Note:** Remove username, password and other sensitive information from
your pinned connections. By default, `username` and `password` fields
will be replaced with “@prompt”, which will prompt the user when
connecting.

## RStudio

This package provides an [RStudio
Addin](https://rstudio.github.io/rstudio-extensions/rstudio_addins.html)
to search for datasets and an [RStudio
Connection](https://rstudio.github.io/rstudio-extensions/rstudio-connections.html)
extension to track local or remote datasets.

![](tools/readme/rstudio-pins-addmin.png)

The addin provides a list of datasets and visual clues that describe how
large and wide eachd dataset is.

# Python

You can install `pins` using
`pip`:

``` python
pip install git+https://github.com/rstudio/pins/#egg=pins\&subdirectory=python --user
```

You can then track your datasets privately with `pin()`,

``` python
import pins

# TODO: pins.pin()
```

and retrieve them back with `get_pin()`.

``` python
pins.get_pin("iris-small-width")
```

    ## TODO

You can search datasets that contain “seattle” in their description or
name as follows:

``` python
pins.find_pin("seattle")
```

    ## # A tibble: 4 x 4
    ##   name               description                               type  board
    ##   <chr>              <chr>                                     <chr> <chr>
    ## 1 hpiR_seattle_sales Seattle Home Sales from hpiR package.     table packa…
    ## 2 microsynth_seattl… Data for a crime intervention in Seattle… table packa…
    ## 3 vegawidget_data_s… Example dataset: Seattle daily weather f… table packa…
    ## 4 vegawidget_data_s… Example dataset: Seattle hourly temperat… table packa…

You can then retrieve a specific dataset with `get_pin()`:

``` python
pins.get_pin("hpiR_seattle_sales")
```

    ## TODO

Please reference the [Python README](Python/README.md) for additional
functionality and details.