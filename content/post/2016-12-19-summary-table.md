---
title:  "Baseline Characteristics Tables with qwraps2"
date:   2016-12-19
tags: [R, data summaries, qwraps2]
---



Almost every biomedical research paper requires a "Table 1: baseline patient
characteristics."  Many developers have published tools to help streamline the
construction of such tables.  The `qwraps2::summary_table` function is my
contribution to the toolbox.

I have constructed hundreds of Table 1s while working as a biostatistics
consultant in a biostatistics department.  I've also constructed many helper
functions to try and streamline the production of such tables.  However, every
project, every data set, every lead author, every target journal, etc., will
present slightly different requirements for the contents and formating of the
tables.  As such, functions which tried to "do it all", required
constant modification to provide the needed output from each nuanced project.

The [`tableone`](https://CRAN.R-project.org/package=tableone) package does a lot
of good things, and is a great tool for quickly building the baseline summary
tables.  What my experiences has taught me is that each row group, or even each
row, might require some specific formating.  A function that treats all
continuous variables one way and all categorical variables another way, may work
for many cases but not all.

The approach to building the tables I've taken now is explicitly define the
summary statistics I want for each variable in the data set, the formatting
for the summary statistics, and in a way that is easy to work with one or more
grouping variables.

The function `summary_table` within my
[`qwraps2`](https://CRAN.R-project.org/package=qwraps2) package is the tool I
and a few colleagues have started to rely on for building baseline patient
characteristic tables.  (qwraps2, "quick wraps 2", is a package of formatting
functions I've found useful for formating results and generating some graphics
when authoring .Rmd and .Rnw files.)

Load and attach the qwraps2 namespace.  We'll set the `qwraps2_markup` option to
`markdown`.  If this option is not set, `qwraps2` uses
`get0ption(qwraps2_markup, "latex")` as the default markup language.


```r
library(qwraps2)
options(qwraps2_markup = 'markdown') # default is latex
```

<br>

We'll use the `mtcars` dataset for our examples.  Let's report several summary
statistics for miles per gallon, number of cylinders, and weight of the
vehicles.  The following summary is provided to illustrate the functions and
thus will include some summaries that would not be used in a publication.  
The data summary we want will be:

* Miles Per Gallon
  * min
  * mean (sd)
  * median (iqr)
  * max
* Cylinders
  * mean
  * n (%) of four cylinders engines
  * n (%) of six cylinders engines
  * n (%) of eight cylinders engines
* Weight
  * range

For cylinders we'll report several things, the mean number of cylinders, and the
count (%) of 4, 6, and 8 cylinders cars.  In a publication we would likely not
report such a summary, treating cylinders as both a continuous and categorical
value.  However, doing so here helps to illustrate the flexibility of the
`summary_table` method.

Outlining the wanted summary statistics above as a list-of-lists helps to
explain the construction of the summary object constructed below.  The
`summary_table` method takes two arguments, 

1. `.data`, a `data.frame` or a `grouped_df` object, and 
2. `summaries` a list-of-lists of right hand sided `formula`e defining the
   summary statistics.

The construction of the summary table is achieved via `dplyr::summarize_`.

The `mtcar_summaries` object constructed below, defines each needed row of the
summary table via a `formula`.  I've included the  `qwraps2` namespace for
clarity.


```r
mtcar_summaries <-
  list("Miles Per Gallon" =
       list("min:"         = ~ min(mpg),
            "mean (sd)"    = ~ qwraps2::mean_sd(mpg, denote_sd = "paren"),
            "median (IQR)" = ~ qwraps2::median_iqr(mpg),
            "max:"         = ~ max(mpg)),
       "Cylinders:" = 
       list("mean"             = ~ mean(cyl),
            "mean (formatted)" = ~ qwraps2::frmt(mean(cyl)),
            "4 cyl, n (%)"     = ~ qwraps2::n_perc0(cyl == 4),
            "6 cyl, n (%)"     = ~ qwraps2::n_perc0(cyl == 6),
            "8 cyl, n (%)"     = ~ qwraps2::n_perc0(cyl == 8)),
       "Weight" =
       list("Range" = ~ paste(range(wt), collapse = ", "))
       )
```

<br>


The table is constructed and printed with ease:

```r
summary_table(mtcars, mtcar_summaries)
## 
## 
## |                              |mtcars (N = 32)      |
## |:-----------------------------|:--------------------|
## |**Miles Per Gallon**          |&nbsp;&nbsp;         |
## |&nbsp;&nbsp; min:             |10.4                 |
## |&nbsp;&nbsp; mean (sd)        |20.09 (6.03)         |
## |&nbsp;&nbsp; median (IQR)     |19.20 (15.43, 22.80) |
## |&nbsp;&nbsp; max:             |33.9                 |
## |**Cylinders:**                |&nbsp;&nbsp;         |
## |&nbsp;&nbsp; mean             |6.1875               |
## |&nbsp;&nbsp; mean (formatted) |6.19                 |
## |&nbsp;&nbsp; 4 cyl, n (%)     |11 (34)              |
## |&nbsp;&nbsp; 6 cyl, n (%)     |7 (22)               |
## |&nbsp;&nbsp; 8 cyl, n (%)     |14 (44)              |
## |**Weight**                    |&nbsp;&nbsp;         |
## |&nbsp;&nbsp; Range            |1.513, 5.424         |
```

The markdown output, rendered as html is:


|                              |mtcars (N = 32)      |
|:-----------------------------|:--------------------|
|**Miles Per Gallon**          |&nbsp;&nbsp;         |
|&nbsp;&nbsp; min:             |10.4                 |
|&nbsp;&nbsp; mean (sd)        |20.09 (6.03)         |
|&nbsp;&nbsp; median (IQR)     |19.20 (15.43, 22.80) |
|&nbsp;&nbsp; max:             |33.9                 |
|**Cylinders:**                |&nbsp;&nbsp;         |
|&nbsp;&nbsp; mean             |6.1875               |
|&nbsp;&nbsp; mean (formatted) |6.19                 |
|&nbsp;&nbsp; 4 cyl, n (%)     |11 (34)              |
|&nbsp;&nbsp; 6 cyl, n (%)     |7 (22)               |
|&nbsp;&nbsp; 8 cyl, n (%)     |14 (44)              |
|**Weight**                    |&nbsp;&nbsp;         |
|&nbsp;&nbsp; Range            |1.513, 5.424         |

<br>


Extending the table to show the same summary by a grouping variable, we'll use
`am` (Transmission: 0 = automatic, 1 = manual), is done as follows:


```r
summary_table(dplyr::group_by(mtcars, am), mtcar_summaries)
## 
## 
## |                              |am: 0 (N = 19)       |am: 1 (N = 13)       |
## |:-----------------------------|:--------------------|:--------------------|
## |**Miles Per Gallon**          |&nbsp;&nbsp;         |&nbsp;&nbsp;         |
## |&nbsp;&nbsp; min:             |10.4                 |15.0                 |
## |&nbsp;&nbsp; mean (sd)        |17.15 (3.83)         |24.39 (6.17)         |
## |&nbsp;&nbsp; median (IQR)     |17.30 (14.95, 19.20) |22.80 (21.00, 30.40) |
## |&nbsp;&nbsp; max:             |24.4                 |33.9                 |
## |**Cylinders:**                |&nbsp;&nbsp;         |&nbsp;&nbsp;         |
## |&nbsp;&nbsp; mean             |6.947368             |5.076923             |
## |&nbsp;&nbsp; mean (formatted) |6.95                 |5.08                 |
## |&nbsp;&nbsp; 4 cyl, n (%)     |3 (16)               |8 (62)               |
## |&nbsp;&nbsp; 6 cyl, n (%)     |4 (21)               |3 (23)               |
## |&nbsp;&nbsp; 8 cyl, n (%)     |12 (63)              |2 (15)               |
## |**Weight**                    |&nbsp;&nbsp;         |&nbsp;&nbsp;         |
## |&nbsp;&nbsp; Weight           |2.465, 5.424         |1.513, 3.57          |
```

|                              |am: 0 (N = 19)       |am: 1 (N = 13)       |
|:-----------------------------|:--------------------|:--------------------|
|**Miles Per Gallon**          |&nbsp;&nbsp;         |&nbsp;&nbsp;         |
|&nbsp;&nbsp; min:             |10.4                 |15.0                 |
|&nbsp;&nbsp; mean (sd)        |17.15 (3.83)         |24.39 (6.17)         |
|&nbsp;&nbsp; median (IQR)     |17.30 (14.95, 19.20) |22.80 (21.00, 30.40) |
|&nbsp;&nbsp; max:             |24.4                 |33.9                 |
|**Cylinders:**                |&nbsp;&nbsp;         |&nbsp;&nbsp;         |
|&nbsp;&nbsp; mean             |6.947368             |5.076923             |
|&nbsp;&nbsp; mean (formatted) |6.95                 |5.08                 |
|&nbsp;&nbsp; 4 cyl, n (%)     |3 (16)               |8 (62)               |
|&nbsp;&nbsp; 6 cyl, n (%)     |4 (21)               |3 (23)               |
|&nbsp;&nbsp; 8 cyl, n (%)     |12 (63)              |2 (15)               |
|**Weight**                    |&nbsp;&nbsp;         |&nbsp;&nbsp;         |
|&nbsp;&nbsp; Weight           |2.465, 5.424         |1.513, 3.57          |

<br>

And lastly, building one table with a column for the whole data set and columns
for each transmission type is:

```r
cbind(summary_table(mtcars, mtcar_summaries),
      summary_table(dplyr::group_by(mtcars, am), mtcar_summaries))
## 
## 
## |                              |mtcars (N = 32)      |am: 0 (N = 19)       |am: 1 (N = 13)       |
## |:-----------------------------|:--------------------|:--------------------|:--------------------|
## |**Miles Per Gallon**          |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |
## |&nbsp;&nbsp; min:             |10.4                 |10.4                 |15.0                 |
## |&nbsp;&nbsp; mean (sd)        |20.09 (6.03)         |17.15 (3.83)         |24.39 (6.17)         |
## |&nbsp;&nbsp; median (IQR)     |19.20 (15.43, 22.80) |17.30 (14.95, 19.20) |22.80 (21.00, 30.40) |
## |&nbsp;&nbsp; max:             |33.9                 |24.4                 |33.9                 |
## |**Cylinders:**                |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |
## |&nbsp;&nbsp; mean             |6.1875               |6.947368             |5.076923             |
## |&nbsp;&nbsp; mean (formatted) |6.19                 |6.95                 |5.08                 |
## |&nbsp;&nbsp; 4 cyl, n (%)     |11 (34)              |3 (16)               |8 (62)               |
## |&nbsp;&nbsp; 6 cyl, n (%)     |7 (22)               |4 (21)               |3 (23)               |
## |&nbsp;&nbsp; 8 cyl, n (%)     |14 (44)              |12 (63)              |2 (15)               |
## |**Weight**                    |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |
## |&nbsp;&nbsp; Range            |1.513, 5.424         |2.465, 5.424         |1.513, 3.57          |
```

|                              |mtcars (N = 32)      |am: 0 (N = 19)       |am: 1 (N = 13)       |
|:-----------------------------|:--------------------|:--------------------|:--------------------|
|**Miles Per Gallon**          |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |
|&nbsp;&nbsp; min:             |10.4                 |10.4                 |15.0                 |
|&nbsp;&nbsp; mean (sd)        |20.09 (6.03)         |17.15 (3.83)         |24.39 (6.17)         |
|&nbsp;&nbsp; median (IQR)     |19.20 (15.43, 22.80) |17.30 (14.95, 19.20) |22.80 (21.00, 30.40) |
|&nbsp;&nbsp; max:             |33.9                 |24.4                 |33.9                 |
|**Cylinders:**                |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |
|&nbsp;&nbsp; mean             |6.1875               |6.947368             |5.076923             |
|&nbsp;&nbsp; mean (formatted) |6.19                 |6.95                 |5.08                 |
|&nbsp;&nbsp; 4 cyl, n (%)     |11 (34)              |3 (16)               |8 (62)               |
|&nbsp;&nbsp; 6 cyl, n (%)     |7 (22)               |4 (21)               |3 (23)               |
|&nbsp;&nbsp; 8 cyl, n (%)     |14 (44)              |12 (63)              |2 (15)               |
|**Weight**                    |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |
|&nbsp;&nbsp; Range            |1.513, 5.424         |2.465, 5.424         |1.513, 3.57          |

<br>

Using `dplry::group_by` will allow you to build the table with more than one
grouping variable.  For example:

```r
cbind(summary_table(mtcars, mtcar_summaries),
      summary_table(dplyr::group_by(mtcars, am, vs), mtcar_summaries))
## 
## 
## |                              |mtcars (N = 32)      |am: 0 vs: 0 (N = 12) |am: 0 vs: 1 (N = 7)  |am: 1 vs: 0 (N = 6)  |am: 1 vs: 1 (N = 7)  |
## |:-----------------------------|:--------------------|:--------------------|:--------------------|:--------------------|:--------------------|
## |**Miles Per Gallon**          |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |
## |&nbsp;&nbsp; min:             |10.4                 |10.4                 |17.8                 |15.0                 |21.4                 |
## |&nbsp;&nbsp; mean (sd)        |20.09 (6.03)         |15.05 (2.77)         |20.74 (2.47)         |19.75 (4.01)         |28.37 (4.76)         |
## |&nbsp;&nbsp; median (IQR)     |19.20 (15.43, 22.80) |15.20 (14.05, 16.62) |21.40 (18.65, 22.15) |20.35 (16.78, 21.00) |30.40 (25.05, 31.40) |
## |&nbsp;&nbsp; max:             |33.9                 |19.2                 |24.4                 |26.0                 |33.9                 |
## |**Cylinders:**                |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |
## |&nbsp;&nbsp; mean             |6.1875               |8.000000             |5.142857             |6.333333             |4.000000             |
## |&nbsp;&nbsp; mean (formatted) |6.19                 |8.00                 |5.14                 |6.33                 |4.00                 |
## |&nbsp;&nbsp; 4 cyl, n (%)     |11 (34)              |0 (0)                |3 (43)               |1 (17)               |7 (100)              |
## |&nbsp;&nbsp; 6 cyl, n (%)     |7 (22)               |0 (0)                |4 (57)               |3 (50)               |0 (0)                |
## |&nbsp;&nbsp; 8 cyl, n (%)     |14 (44)              |12 (100)             |0 (0)                |2 (33)               |0 (0)                |
## |**Weight**                    |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |
## |&nbsp;&nbsp; Range            |1.513, 5.424         |3.435, 5.424         |2.465, 3.46          |2.14, 3.57           |1.513, 2.78          |
```

|                              |mtcars (N = 32)      |am: 0 vs: 0 (N = 12) |am: 0 vs: 1 (N = 7)  |am: 1 vs: 0 (N = 6)  |am: 1 vs: 1 (N = 7)  |
|:-----------------------------|:--------------------|:--------------------|:--------------------|:--------------------|:--------------------|
|**Miles Per Gallon**          |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |
|&nbsp;&nbsp; min:             |10.4                 |10.4                 |17.8                 |15.0                 |21.4                 |
|&nbsp;&nbsp; mean (sd)        |20.09 (6.03)         |15.05 (2.77)         |20.74 (2.47)         |19.75 (4.01)         |28.37 (4.76)         |
|&nbsp;&nbsp; median (IQR)     |19.20 (15.43, 22.80) |15.20 (14.05, 16.62) |21.40 (18.65, 22.15) |20.35 (16.78, 21.00) |30.40 (25.05, 31.40) |
|&nbsp;&nbsp; max:             |33.9                 |19.2                 |24.4                 |26.0                 |33.9                 |
|**Cylinders:**                |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |
|&nbsp;&nbsp; mean             |6.1875               |8.000000             |5.142857             |6.333333             |4.000000             |
|&nbsp;&nbsp; mean (formatted) |6.19                 |8.00                 |5.14                 |6.33                 |4.00                 |
|&nbsp;&nbsp; 4 cyl, n (%)     |11 (34)              |0 (0)                |3 (43)               |1 (17)               |7 (100)              |
|&nbsp;&nbsp; 6 cyl, n (%)     |7 (22)               |0 (0)                |4 (57)               |3 (50)               |0 (0)                |
|&nbsp;&nbsp; 8 cyl, n (%)     |14 (44)              |12 (100)             |0 (0)                |2 (33)               |0 (0)                |
|**Weight**                    |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |&nbsp;&nbsp;         |
|&nbsp;&nbsp; Range            |1.513, 5.424         |3.435, 5.424         |2.465, 3.46          |2.14, 3.57           |1.513, 2.78          |

<br>


The formatting of the output is controlled by the
`qwraps2:::print.qwraps2_summary_table` and `qwraps2::qable` functions.  


```r
args(qwraps2:::print.qwraps2_summary_table)
## function (x, rgroup = attr(x, "rgroups"), rnames = rownames(x), 
##     cnames = colnames(x), ...) 
## NULL
args(qwraps2::qable)
## function (x, rtitle, rgroup, rnames = rownames(x), cnames = colnames(x), 
##     markup = getOption("qwraps2_markup", "latex"), ...) 
## NULL
```

The `print` method for `qwraps2_summary_table` objects calls `qable` which is a
wrapper around `knitr::kable`.  The row groups,` rgroup`, row names `rnames`,
and column names, `cnames`, are explicitly set in the
`print.qwraps2_summary_table` method.  The `...` passes additional arguments to
`qwraps2::qable` which can then continue to pass to `knitr::kable`.

A quick example of modifying a table:

```r
by_am <- summary_table(dplyr::group_by(mtcars, am), mtcar_summaries)

print(by_am, 
      cnames = c("_Manual_", "_Automatic_"),
      rtitle = "Vehicle Characteristics", 
      align = "lcc")
## 
## 
## |Vehicle Characteristics       |       _Manual_       |     _Automatic_      |
## |:-----------------------------|:--------------------:|:--------------------:|
## |**Miles Per Gallon**          |     &nbsp;&nbsp;     |     &nbsp;&nbsp;     |
## |&nbsp;&nbsp; min:             |         10.4         |         15.0         |
## |&nbsp;&nbsp; mean (sd)        |     17.15 (3.83)     |     24.39 (6.17)     |
## |&nbsp;&nbsp; median (IQR)     | 17.30 (14.95, 19.20) | 22.80 (21.00, 30.40) |
## |&nbsp;&nbsp; max:             |         24.4         |         33.9         |
## |**Cylinders:**                |     &nbsp;&nbsp;     |     &nbsp;&nbsp;     |
## |&nbsp;&nbsp; mean             |       6.947368       |       5.076923       |
## |&nbsp;&nbsp; mean (formatted) |         6.95         |         5.08         |
## |&nbsp;&nbsp; 4 cyl, n (%)     |        3 (16)        |        8 (62)        |
## |&nbsp;&nbsp; 6 cyl, n (%)     |        4 (21)        |        3 (23)        |
## |&nbsp;&nbsp; 8 cyl, n (%)     |       12 (63)        |        2 (15)        |
## |**Weight**                    |     &nbsp;&nbsp;     |     &nbsp;&nbsp;     |
## |&nbsp;&nbsp; Weight           |     2.465, 5.424     |     1.513, 3.57      |
```


|Vehicle Characteristics       |       _Manual_       |     _Automatic_      |
|:-----------------------------|:--------------------:|:--------------------:|
|**Miles Per Gallon**          |     &nbsp;&nbsp;     |     &nbsp;&nbsp;     |
|&nbsp;&nbsp; min:             |         10.4         |         15.0         |
|&nbsp;&nbsp; mean (sd)        |     17.15 (3.83)     |     24.39 (6.17)     |
|&nbsp;&nbsp; median (IQR)     | 17.30 (14.95, 19.20) | 22.80 (21.00, 30.40) |
|&nbsp;&nbsp; max:             |         24.4         |         33.9         |
|**Cylinders:**                |     &nbsp;&nbsp;     |     &nbsp;&nbsp;     |
|&nbsp;&nbsp; mean             |       6.947368       |       5.076923       |
|&nbsp;&nbsp; mean (formatted) |         6.95         |         5.08         |
|&nbsp;&nbsp; 4 cyl, n (%)     |        3 (16)        |        8 (62)        |
|&nbsp;&nbsp; 6 cyl, n (%)     |        4 (21)        |        3 (23)        |
|&nbsp;&nbsp; 8 cyl, n (%)     |       12 (63)        |        2 (15)        |
|**Weight**                    |     &nbsp;&nbsp;     |     &nbsp;&nbsp;     |
|&nbsp;&nbsp; Weight           |     2.465, 5.424     |     1.513, 3.57      |

<br>

I hope that some readers will find this approach to building summary tables to
be useful.  If you find bugs or have suggestions on how to extend and improve
this tool please create an
[issue on github](https://github.com/dewittpe/qwraps2/issues).

