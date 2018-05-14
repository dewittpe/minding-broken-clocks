---
title:  "Installing Package Dependencies without external http(s) requests"
date:   2018-02-25
tags: [R, programming, package development]
---



Consider you have a server that is running behind a firewall and, for security
reasons, cannot make external http(s) requests.  Further, you have R running on
this server and you need to install a set of packages.  The simple approach of 

    install.packages("<pkg-name>", repo = "<favorite cran mirror>")

is not an option since you will have no access to the CRAN repository.

Another option would be to download the source files (.tar.gz files) form CRAN
or BioConductor, transfer those files to the sever via FTP, and then install the
packages via

    install.packages("<path to pkg-name_version.tar.gz", type = "source")

This approach will work well, with one big exception, the dependencies of the
package may not be on the server.  How do you get the source files for all the
dependencies of your package?  What about the dependencies of the dependencies,
and the dependencies of the dependencies of the dependencies?  Simply, how do
you install R packages on a machine that is not allowed to make external http(s)
requests?  

Here is how I approached this problem.  On my local machine, a machine with
internet access, I ran a script (a script that will be shown and explained in
detail below) which will download all the dependencies and dependencies of
dependencies, etc., from both CRAN and BioConductor, and generate a `makefile`
to install the packages in the correct order, i.e., in an order such that the
dependencies are met.

When the script finished, the source files and the `makefile` can be transfered
to the server without external http(s) request authority.  Running the
`makefile` will install the packages, and is an easy way to track and report
install errors.

We need to define what constitutes a dependency.   In a package `DESCRIPTION`
file packages listed under the field `Depends`, `Imports`, and `LinkingTo` are
what we will consider dependencies.  `Suggests` and `Enhances` are omitted as
they are not needed for the package to work.

## Build Dependencies
An R script `build-dep-list.R` has been written and is expected to be evaluated
from the command line via

    Rscript --vanilla build-dep-list.R [pkg1] [pkg2] [...] [pkgn]

Where `pkg1` is the name of the first known package to download, `pkg2` the
second known package to download, ..., and `pkgn` the nth package to download.
The script will download all the dependencies for `pkg1`, `...`, `pkgn`, and the
dependencies of the dependencies, and so on.  The script will also generate a
`makefile` to help with the installation of the packages, aiming to get the
order of the installs so that the install of `pkg1`, `...`, `pkgn` will not
error.

The full script can be found on my
[github](https://github.com/dewittpe/R-install-dependencies) page.  The script
will be broken up into pieces here with additional detail and explanation.

When I develop scripts that I expect to evaluated in the terminal, I will start
the script with a check of `interactive()`.  If in an interactive session we'll
have set variables to values needed for testing and development, and if not in
an interactive session we'll use the command line arguments to define the value
of the variables.  This could also be edited so that the expected evaluation
would be done in an interactive session.  For then work we will have the
`character` vector `OUR_PACKAGES` to store the names of the packages we
want/need to install.

```r
if (interactive()) {
  OUR_PACKAGES <- c("graph", "gRbase", "gRain", "jsonlite", "plotly", "SHELF",
                    "rjson", "svglite", "magrittr")
} else {
  OUR_PACKAGES <- commandArgs(trailingOnly = TRUE)
}
```



We also need to define the repositories which we will query for the packages.
We'll use RStudio's CRAN mirror and the repository for BioConductor.


```r
# Repositories to look for packages
CRAN <- "https://cran.rstudio.com/"
BIOC <- "https://bioconductor.org/packages/release/bioc/"
```

Now, let's look into the packages.  Packages are classified into three priority
classes, "base", "recommended", and "NA".  The "base" packages are standard an R
installation, and the 'recommended' are in any standard installation of R.  All
other packages have `Priority == NA`.


```r
ipkgs <- utils::installed.packages()
ipkgs[ipkgs[, "Priority"] %in% "base", "Package"]
##        base    compiler    datasets    graphics   grDevices        grid 
##      "base"  "compiler"  "datasets"  "graphics" "grDevices"      "grid" 
##     methods    parallel     splines       stats      stats4       tcltk 
##   "methods"  "parallel"   "splines"     "stats"    "stats4"     "tcltk" 
##       tools       utils 
##     "tools"     "utils"
ipkgs[ipkgs[, "Priority"] %in% "recommended", "Package"]
##         boot        class      cluster    codetools      foreign 
##       "boot"      "class"    "cluster"  "codetools"    "foreign" 
##   KernSmooth      lattice         MASS       Matrix         mgcv 
## "KernSmooth"    "lattice"       "MASS"     "Matrix"       "mgcv" 
##         nnet        rpart      spatial     survival 
##       "nnet"      "rpart"    "spatial"   "survival"
```

Some packages will have dependencies on the "base" and/or "recommended"
packages.  We will need to know these packages and omit them form the packages
we will need to download and install.


```r
base_pkgs <-
  unname(utils::installed.packages()[utils::installed.packages()[, "Priority"] %in% c("base", "recommended"), "Package"])
```

Next step, get a list of the available packages from CRAN and BioConductor.  The
return from `available.packages` is a matrix with all the information we will
need about the packages.


```r
available_pkgs <- available.packages(repos = c(CRAN, BIOC))
str(available_pkgs)
##  chr [1:13659, 1:17] "A3" "abbyyR" "abc" "abc.data" "ABC.RAP" ...
##  - attr(*, "dimnames")=List of 2
##   ..$ : chr [1:13659] "A3" "abbyyR" "abc" "abc.data" ...
##   ..$ : chr [1:17] "Package" "Version" "Priority" "Depends" ...

available_pkgs[available_pkgs[, "Package"] %in% OUR_PACKAGES,
               c("Package", "Version", "Depends", "Imports", "LinkingTo",
                 "Repository")]
##          Package    Version 
## gRain    "gRain"    "1.3-0" 
## gRbase   "gRbase"   "1.8-3" 
## jsonlite "jsonlite" "1.5"   
## magrittr "magrittr" "1.5"   
## plotly   "plotly"   "4.7.1" 
## rjson    "rjson"    "0.2.15"
## SHELF    "SHELF"    "1.3.0" 
## svglite  "svglite"  "1.2.1" 
## graph    "graph"    "1.56.0"
##          Depends                                          
## gRain    "R (>= 3.0.2), methods, gRbase (>= 1.7-2)"       
## gRbase   "R (>= 3.0.2), methods"                          
## jsonlite "methods"                                        
## magrittr NA                                               
## plotly   "R (>= 3.2.0), ggplot2 (>= 2.2.1)"               
## rjson    "R (>= 3.1.0)"                                   
## SHELF    "R (>= 3.3.1)"                                   
## svglite  "R (>= 3.0.0)"                                   
## graph    "R (>= 2.10), methods, BiocGenerics (>= 0.13.11)"
##          Imports                                                                                                                                                                                                     
## gRain    "igraph, graph, magrittr, functional, Rcpp (>= 0.11.1)"                                                                                                                                                     
## gRbase   "graph, igraph, magrittr, Matrix, RBGL, Rcpp (>= 0.11.1)"                                                                                                                                                   
## jsonlite NA                                                                                                                                                                                                          
## magrittr NA                                                                                                                                                                                                          
## plotly   "tools, scales, httr, jsonlite, magrittr, digest, viridisLite,\nbase64enc, htmltools, htmlwidgets (>= 0.9), tidyr, hexbin,\nRColorBrewer, dplyr, tibble, lazyeval (>= 0.2.0), crosstalk,\npurrr, data.table"
## rjson    NA                                                                                                                                                                                                          
## SHELF    "ggplot2, grid, shiny, stats, graphics, tidyr, MASS, ggExtra"                                                                                                                                               
## svglite  "Rcpp, gdtools (>= 0.1.6)"                                                                                                                                                                                  
## graph    "stats, stats4, utils"                                                                                                                                                                                      
##          LinkingTo                                                       
## gRain    "Rcpp (>= 0.11.1), RcppArmadillo, RcppEigen, gRbase (>=\n1.8-0)"
## gRbase   "Rcpp (>= 0.11.1), RcppArmadillo, RcppEigen"                    
## jsonlite NA                                                              
## magrittr NA                                                              
## plotly   NA                                                              
## rjson    NA                                                              
## SHELF    NA                                                              
## svglite  "Rcpp, gdtools, BH"                                             
## graph    NA                                                              
##          Repository                                                  
## gRain    "https://cran.rstudio.com/src/contrib"                      
## gRbase   "https://cran.rstudio.com/src/contrib"                      
## jsonlite "https://cran.rstudio.com/src/contrib"                      
## magrittr "https://cran.rstudio.com/src/contrib"                      
## plotly   "https://cran.rstudio.com/src/contrib"                      
## rjson    "https://cran.rstudio.com/src/contrib"                      
## SHELF    "https://cran.rstudio.com/src/contrib"                      
## svglite  "https://cran.rstudio.com/src/contrib"                      
## graph    "https://bioconductor.org/packages/release/bioc/src/contrib"
```

In this example we see that the packages listed in `OUR_PACKAGES` except the
`graph` package can be downloaded from CRAN.   `graph` and at least one
dependencies, `BiocGenerics` will need to be downloaded from BioConductor.

The next step in building the list of dependencies and a script for installing
them is done in the following `while` loop.  We start with a character vector
`pkgs_to_download` which is initially equivalent to `OUR_PACKAGES`.  We will
iterate through this vector, appending the dependencies in order.
Use the `tools::package_dependencies` function to generate a list of the
packages dependencies, and dependencies of dependencies, and so on.

In the `while` loop we get a list of the dependencies for a package, stored in
the `deps` object.  We will omit any of the base and recommended packages from
the `deps` object and then append `deps` to the `pkgs_to_download` vector in the
position immediately to the right of the current package being looked up.  When
the indexer `i` is incremented, the next package to be considered will be the
first dependency.  This process continues until all the dependencies have been
explored.  Lastly, we reverse the order of the elements of  `pkgs_to_download`
so that we have the packages listed in a useful install order, i.e.,
`pkgs_to_download[1]` should be installed before `pkgs_to_download[2]`, etc.
After reversing the order of the elements of `pkgs_to_download` we look only at
the unique elements.  By default, the first occurrence of an element will be
keep and the repeated elements will be omitted.  By reversing the order then
taking the unique values, the deepest level of dependency will be retained for a
specific package.

```r
pkgs_to_download <- OUR_PACKAGES
i <- 1L
while(i <= length(pkgs_to_download)) {
  deps <-
    unlist(tools::package_dependencies(packages = pkgs_to_download[i],
                                       which = c("Depends", "Imports", "LinkingTo"),
                                       db = available_pkgs,
                                       recursive = FALSE),
           use.names = FALSE) 
  deps <- deps[!(deps %in% base_pkgs)]
  pkgs_to_download <- append(pkgs_to_download, deps, i) 
  i <- i + 1L
}
pkgs_to_download <- unique(rev(pkgs_to_download))
```

If you are having a difficult time envisioning what the above does, let's look
at and example for the `dplyr` package.  In this example we'll print out the
list of dependencies at each step through the while loop.  Note that packages
such as `Rcpp` will be assessed multiple times, but the final list will only
have `Rcpp` listed once.

```r
dplyr_dependencies <- "dplyr"
i <- 1L
while(i <= length(dplyr_dependencies)) {

  cat("\ni =", i, "\nLooking up dependencies for", dplyr_dependencies[i], "\n")
  deps <-
    unlist(tools::package_dependencies(packages = dplyr_dependencies[i],
                                       which = c("Depends", "Imports", "LinkingTo"),
                                       db = available_pkgs,
                                       recursive = FALSE),
           use.names = FALSE) 
  deps <- deps[!(deps %in% base_pkgs)]
  dplyr_dependencies <- append(dplyr_dependencies, deps, i) 
  
  cat(dplyr_dependencies[i], "dependencies:", paste(deps, collapse = ", "),
      "\ndplyr_dependencies =", paste(dplyr_dependencies, collapse = ", "), "\n")

  i <- i + 1L
}
## 
## i = 1 
## Looking up dependencies for dplyr 
## dplyr dependencies: assertthat, bindrcpp, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, BH, plogr 
## dplyr_dependencies = dplyr, assertthat, bindrcpp, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, BH, plogr 
## 
## i = 2 
## Looking up dependencies for assertthat 
## assertthat dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, BH, plogr 
## 
## i = 3 
## Looking up dependencies for bindrcpp 
## bindrcpp dependencies: Rcpp, bindr, plogr 
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, BH, plogr 
## 
## i = 4 
## Looking up dependencies for Rcpp 
## Rcpp dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, BH, plogr 
## 
## i = 5 
## Looking up dependencies for bindr 
## bindr dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, BH, plogr 
## 
## i = 6 
## Looking up dependencies for plogr 
## plogr dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, BH, plogr 
## 
## i = 7 
## Looking up dependencies for glue 
## glue dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, BH, plogr 
## 
## i = 8 
## Looking up dependencies for magrittr 
## magrittr dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, BH, plogr 
## 
## i = 9 
## Looking up dependencies for pkgconfig 
## pkgconfig dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, BH, plogr 
## 
## i = 10 
## Looking up dependencies for rlang 
## rlang dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, BH, plogr 
## 
## i = 11 
## Looking up dependencies for R6 
## R6 dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, BH, plogr 
## 
## i = 12 
## Looking up dependencies for Rcpp 
## Rcpp dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, BH, plogr 
## 
## i = 13 
## Looking up dependencies for tibble 
## tibble dependencies: cli, crayon, pillar, rlang 
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, cli, crayon, pillar, rlang, BH, plogr 
## 
## i = 14 
## Looking up dependencies for cli 
## cli dependencies: assertthat, crayon 
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, cli, assertthat, crayon, crayon, pillar, rlang, BH, plogr 
## 
## i = 15 
## Looking up dependencies for assertthat 
## assertthat dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, cli, assertthat, crayon, crayon, pillar, rlang, BH, plogr 
## 
## i = 16 
## Looking up dependencies for crayon 
## crayon dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, cli, assertthat, crayon, crayon, pillar, rlang, BH, plogr 
## 
## i = 17 
## Looking up dependencies for crayon 
## crayon dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, cli, assertthat, crayon, crayon, pillar, rlang, BH, plogr 
## 
## i = 18 
## Looking up dependencies for pillar 
## pillar dependencies: cli, crayon, rlang, utf8 
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, cli, assertthat, crayon, crayon, pillar, cli, crayon, rlang, utf8, rlang, BH, plogr 
## 
## i = 19 
## Looking up dependencies for cli 
## cli dependencies: assertthat, crayon 
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, cli, assertthat, crayon, crayon, pillar, cli, assertthat, crayon, crayon, rlang, utf8, rlang, BH, plogr 
## 
## i = 20 
## Looking up dependencies for assertthat 
## assertthat dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, cli, assertthat, crayon, crayon, pillar, cli, assertthat, crayon, crayon, rlang, utf8, rlang, BH, plogr 
## 
## i = 21 
## Looking up dependencies for crayon 
## crayon dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, cli, assertthat, crayon, crayon, pillar, cli, assertthat, crayon, crayon, rlang, utf8, rlang, BH, plogr 
## 
## i = 22 
## Looking up dependencies for crayon 
## crayon dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, cli, assertthat, crayon, crayon, pillar, cli, assertthat, crayon, crayon, rlang, utf8, rlang, BH, plogr 
## 
## i = 23 
## Looking up dependencies for rlang 
## rlang dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, cli, assertthat, crayon, crayon, pillar, cli, assertthat, crayon, crayon, rlang, utf8, rlang, BH, plogr 
## 
## i = 24 
## Looking up dependencies for utf8 
## utf8 dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, cli, assertthat, crayon, crayon, pillar, cli, assertthat, crayon, crayon, rlang, utf8, rlang, BH, plogr 
## 
## i = 25 
## Looking up dependencies for rlang 
## rlang dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, cli, assertthat, crayon, crayon, pillar, cli, assertthat, crayon, crayon, rlang, utf8, rlang, BH, plogr 
## 
## i = 26 
## Looking up dependencies for BH 
## BH dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, cli, assertthat, crayon, crayon, pillar, cli, assertthat, crayon, crayon, rlang, utf8, rlang, BH, plogr 
## 
## i = 27 
## Looking up dependencies for plogr 
## plogr dependencies:  
## dplyr_dependencies = dplyr, assertthat, bindrcpp, Rcpp, bindr, plogr, glue, magrittr, pkgconfig, rlang, R6, Rcpp, tibble, cli, assertthat, crayon, crayon, pillar, cli, assertthat, crayon, crayon, rlang, utf8, rlang, BH, plogr
dplyr_dependencies <- unique(rev(dplyr_dependencies))
dplyr_dependencies
##  [1] "plogr"      "BH"         "rlang"      "utf8"       "crayon"    
##  [6] "assertthat" "cli"        "pillar"     "tibble"     "Rcpp"      
## [11] "R6"         "pkgconfig"  "magrittr"   "glue"       "bindr"     
## [16] "bindrcpp"   "dplyr"
```

Now that we have `pkgs_to_download`, a character vector of package names that
we need to download, we can use the `download.packages` function to do so.  The
object `dwnld_pkgs` is a 2 column matrix with the name and file path to the
source file for each package.


```r
# Download the needed packages into the pkg-source-files directory
unlink("pkg-source-files/*")
dir.create("pkg-source-files/", showWarnings = FALSE)

dwnld_pkgs <-
  download.packages(pkgs = pkgs_to_download,
                    destdir = "pkg-source-files",
                    repos = c(CRAN, BIOC),
                    type = "source")

head(dwnld_pkgs)
```

The last step for the script to run on a machine with external http(s) request
authority, is to build a `makefile` to install all the needed packages.  I
prefer the `makefile` over a bash script because of the default error handling
that a `make` provided compared to a bash script.


```r
cat("all:\n",
    paste0("\tR CMD INSTALL ", dwnld_pkgs[, 2], "\n"),
    sep = "",
    file = "makefile") 
```

For this example, the first several lines of the `makefile` are:

    all:
      R CMD INSTALL pkg-source-files/magrittr_1.5.tar.gz
      R CMD INSTALL pkg-source-files/BH_1.66.0-1.tar.gz
      R CMD INSTALL pkg-source-files/withr_2.1.1.tar.gz
      R CMD INSTALL pkg-source-files/Rcpp_0.12.15.tar.gz
      R CMD INSTALL pkg-source-files/gdtools_0.1.6.tar.gz
      R CMD INSTALL pkg-source-files/svglite_1.2.1.tar.gz

Note that magrittr is the last package in the `OUR_PACKAGES` object and has no
dependencies, thus is the first package installed.  The svglite package is the
second to last package in `OUR_PACKAGES` and it will be installed after the
dependencies BH, withr, Rcpp, and gdtools are installed.

## Installing on the Remote Machine
Now that the source files have been downloaded and the `makefile` generated,
move the `pkg-source-files` directory and the `makefile` to the remote machine
and run the makefile.  If the makefile fails, there might be some system
dependencies that need to be updated.

## Download the script and/or contribute
The `build-dependency-list.R` file can be found on my
[github](https://github.com/dewittpe/R-install-dependencies) page.
