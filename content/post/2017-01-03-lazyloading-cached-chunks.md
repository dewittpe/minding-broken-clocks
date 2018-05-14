---
title:  "(lazy)Loading Cached Chunks into an Interactive R Session"
date:   2017-01-03
tags: [R, knitr, cache, qwraps2]
---

If you cache code chunks when using
[`knitr`](https://CRAN.R-project.org/package=knitr) to generate reproducible
documents then you've likely had the issue arrise of needing to load the results
of cached chunks into an active interactive R session.  The functions
`lazyload_cache_dir` and `lazyload_cache_labels` have been added to the
developmental version of [qwraps2](https://github.com/dewittpe/qwraps2) to make
loading cached objects into an interactive R session easy.



Let's make some trivial cached chunks in a file `report.Rmd`

```
  ---
  title:  "A Report"
  output: html_document
  ---
  
  ```{r first-chunk, cache = TRUE}
  fit <- lm(mpg ~ wt + hp, data = mtcars)
  x <- pi
  ```
  
  ```{r second-chunk, cache = TRUE}
  fit <- lm(mpg ~ wt + hp + am, data = mtcars)
  xx <- exp(1)
  ```
```

  
The resulting project directory looks like this:

```
    .
    ├── report_cache
    │   └── html
    │       ├── first-chunk_bf368425c25f0c3d95cac85aff007ad1.RData
    │       ├── first-chunk_bf368425c25f0c3d95cac85aff007ad1.rdb
    │       ├── first-chunk_bf368425c25f0c3d95cac85aff007ad1.rdx
    │       ├── __packages
    │       ├── second-chunk_2c7d6b477306be1d4d4ed451f2f1b52a.RData
    │       ├── second-chunk_2c7d6b477306be1d4d4ed451f2f1b52a.rdb
    │       └── second-chunk_2c7d6b477306be1d4d4ed451f2f1b52a.rdx
    ├── report.html
    └── report.Rmd
```

Now, let's assume you need to come back to this project at a later date.  You
want to get the objects created in the cached chunks into a new interactive R
session.

If you use `lazyload_cache_dir` all the cached chunks will be load.


```r
rm(list = ls())
ls()
## character(0)
qwraps2::lazyload_cache_dir(path = "report_cache/html")
## Lazyloading: report_cache/html/first-chunk
## Lazyloading: report_cache/html/second-chunk
ls()
## [1] "fit" "x"   "xx"
fit
## 
## Call:
## lm(formula = mpg ~ wt + hp + am, data = mtcars)
## 
## Coefficients:
## (Intercept)           wt           hp           am  
##    34.00288     -2.87858     -0.03748      2.08371
x
## [1] 3.141593
xx
## [1] 2.718282
```
This could be a problem.  There was a `fit` created in `first-chunk` and another
object `fit` created in `second-chunk`.  The `fit` object in the active
workspace is from `second-chunk.`  What if you want the fit from `first-chunk`?
Use `lazyload_cache_labels`.


```r
rm(list = ls())
ls()
## character(0)
qwraps2::lazyload_cache_labels("first-chunk", path = "report_cache/html")
## Lazyloading report_cache/html/first-chunk_bf368425c25f0c3d95cac85aff007ad1
ls()
## [1] "fit" "x"
fit
## 
## Call:
## lm(formula = mpg ~ wt + hp, data = mtcars)
## 
## Coefficients:
## (Intercept)           wt           hp  
##    37.22727     -3.87783     -0.03177
x
## [1] 3.141593
xx
## Error in eval(expr, envir, enclos): object 'xx' not found
```

Now, what if you want `fit` from `first-chunk` and `xx` which was created in
`second-chunk`?


```r
rm(list = ls())
ls()
## character(0)
qwraps2::lazyload_cache_dir(path = "report_cache/html")
## Lazyloading: report_cache/html/first-chunk
## Lazyloading: report_cache/html/second-chunk
qwraps2::lazyload_cache_labels("first-chunk", path = "report_cache/html")
## Lazyloading report_cache/html/first-chunk_bf368425c25f0c3d95cac85aff007ad1
ls()
## [1] "fit" "x"   "xx"
fit
## 
## Call:
## lm(formula = mpg ~ wt + hp, data = mtcars)
## 
## Coefficients:
## (Intercept)           wt           hp  
##    37.22727     -3.87783     -0.03177
x
## [1] 3.141593
xx
## [1] 2.718282
```

A better solution would be to use unique object names.  

Lastly, say you only want the `fit` from `first-chunk` and no other objects.
The optional `filter` argument of `lazyload_cache_labels` can be used.  The
`filter` argument is passed to `lazyLoad`.  "filter: An optional function which
when called on a character vector of object names returns a logical vector:
only objects for which this is true will be loaded."


```r
rm(list = ls())
ls()
## character(0)
qwraps2::lazyload_cache_labels("first-chunk",
                               path = "report_cache/html",
                               filter = function(x) x == "fit")
## Lazyloading report_cache/html/first-chunk_bf368425c25f0c3d95cac85aff007ad1
ls()
## [1] "fit"
fit
## 
## Call:
## lm(formula = mpg ~ wt + hp, data = mtcars)
## 
## Coefficients:
## (Intercept)           wt           hp  
##    37.22727     -3.87783     -0.03177
```

The `lazyload_cache_dir` and `lazyload_cache_labels` functions are currently in
the developmental version of `qwarps2`.  You can get the code and install the
package from [https://github.com/dewittpe/qwraps2](https://github.com/dewittpe/qwraps2).
I'm not sure when the next version of `qwraps2` will be pushed to CRAN, so the
developmental version will have to do for now.

If you find any bugs or have suggestions on how to improve the functions please
[create and issue](https://github.com/dewittpe/qwraps2/issues).

Note, the behavior of `knitr::load_cache` is very different from the behavior of
`qwraps2::lazyload_cache_dir` and `qwraps2::lazyload_cache_labels`.
`knitr::load_cache` is to be used to load cached values within a .Rmd prior to
the chunk being called.  See the documentation and 
[example 114-load-cache.Rmd](https://github.com/yihui/knitr-examples) from the 
[knitr-examples](https://github.com/yihui/knitr-examples) repository for the use of
`knitr::load_cache`.

