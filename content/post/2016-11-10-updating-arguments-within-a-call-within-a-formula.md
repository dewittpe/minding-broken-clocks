---
title:  "Updating Call Arguments"
date:   2016-11-10
tags: [R, programming]
---

The `stats::update` function is one of my favorite tools in R.  Using this
function saves a lot of time and effort when needing to modify an object.
However, this function has its limits.  The following is an example of how to
extend the use of `stats::update`.

This post is an extension of the lightening talk I gave to the [Denver R Users
Group](http://www.meetup.com/DenverRUG/) in March of 2016.  You can get the
[slides from that talk](https://github.com/dewittpe/drug-20160330) from my
[github](https://github.com/dewittpe) page.



## Basics of the `stats::update` function
The
[documentation](https://stat.ethz.ch/R-manual/R-devel/library/stats/html/update.html)
for the `stats::update` function is a good starting place.  Of course, examples
are even better.  We'll use the `diamonds` data set from within the [ggplot2](
https://CRAN.R-project.org/package=ggplot2) package for the examples.  By
default, the `cut`, `color`, and `clarity` elements of the `diamonds` data set
are ordered factors.  I'm going to remove the order and level these variables as
just factors.


```r
data("diamonds", package = "ggplot2")

diamonds$cut     <- factor(diamonds$cut, ordered = FALSE)
diamonds$color   <- factor(diamonds$color, ordered = FALSE)
diamonds$clarity <- factor(diamonds$clarity, ordered = FALSE)

dplyr::glimpse(diamonds) 
## Observations: 53,940
## Variables: 10
## $ carat   <dbl> 0.23, 0.21, 0.23, 0.29, 0.31, 0.24, 0.24, 0.26, 0.22, ...
## $ cut     <fctr> Ideal, Premium, Good, Premium, Good, Very Good, Very ...
## $ color   <fctr> E, E, E, I, J, J, I, H, E, H, J, J, F, J, E, E, I, J,...
## $ clarity <fctr> SI2, SI1, VS1, VS2, SI2, VVS2, VVS1, SI1, VS2, VS1, S...
## $ depth   <dbl> 61.5, 59.8, 56.9, 62.4, 63.3, 62.8, 62.3, 61.9, 65.1, ...
## $ table   <dbl> 55, 61, 65, 58, 58, 57, 57, 55, 61, 61, 55, 56, 61, 54...
## $ price   <int> 326, 326, 327, 334, 335, 336, 336, 337, 337, 338, 339,...
## $ x       <dbl> 3.95, 3.89, 4.05, 4.20, 4.34, 3.94, 3.95, 4.07, 3.87, ...
## $ y       <dbl> 3.98, 3.84, 4.07, 4.23, 4.35, 3.96, 3.98, 4.11, 3.78, ...
## $ z       <dbl> 2.43, 2.31, 2.31, 2.63, 2.75, 2.48, 2.47, 2.53, 2.49, ...
```

A simple regression model, a `lm` object, will be used for our examples.  Let's
regress the `price` of diamonds as a function of `carat`, `cut`, `color`, and
`clarity`.


```r
original_fit <- lm(price ~ carat + cut + color + clarity, data = diamonds)
```

## Updating a `call`
The `stats::update` function works by calling `getCall` which calls
`getElement`.  While that is nice to know, the take away message is that if you
have an object with an element called `call`, then `stat::update` gives you
assess to modify the call.

Let's update the regression formula.  Say instead of the four predictors we
started with we want to regress `price` on the `depth` and `table` of the
diamonds

```r
updated_fit_1 <- update(original_fit, formula = . ~ depth + table)
```
In the code chunk above, the `update` call took the `original_fit` object as its
first argument and then we provided the `formula = . ~ depth + table` argent to
tell the interpreter that we want to modify the formula.  The `.` is shorthand
for "the current," i.e., use the current left hand side of the formula, and
replace the right hand side with the `depth + table`.

Using `getCall` we can see that the two objects have different calls, and the
calls we would expect them to have.

```r
getCall(original_fit)
## lm(formula = price ~ carat + cut + color + clarity, data = diamonds)
getCall(updated_fit_1)
## lm(formula = price ~ depth + table, data = diamonds)
```

One more sanity check: the regression coefficients for these two models are, at
least in name, as expected:

```r
coef(original_fit)
##  (Intercept)        carat      cutGood cutVery Good   cutPremium 
##   -7362.8022    8886.1289     655.7674     848.7169     869.3959 
##     cutIdeal       colorE       colorF       colorG       colorH 
##     998.2544    -211.6825    -303.3100    -506.1995    -978.6977 
##       colorI       colorJ   claritySI2   claritySI1   clarityVS2 
##   -1440.3019   -2325.2224    2625.9500    3573.6880    4217.8291 
##   clarityVS1  clarityVVS2  clarityVVS1    clarityIF 
##    4534.8790    4967.1994    5072.0276    5419.6468
coef(updated_fit_1)
##  (Intercept)        depth        table 
## -15084.96675     82.26159    242.58346
```

We can update more than just the `formula` in the call.  Perhaps you want to fit
the same regression model on a subset of the data, for example, use the same
regression formula as in `original_fit` but subset the data to only diamonds
with a `carat` weight under 2.

```r
updated_fit_2 <- update(original_fit, data = dplyr::filter(diamonds, carat < 2))
getCall(updated_fit_2)
## lm(formula = price ~ carat + cut + color + clarity, data = dplyr::filter(diamonds, 
##     carat < 2))

coef(original_fit)
##  (Intercept)        carat      cutGood cutVery Good   cutPremium 
##   -7362.8022    8886.1289     655.7674     848.7169     869.3959 
##     cutIdeal       colorE       colorF       colorG       colorH 
##     998.2544    -211.6825    -303.3100    -506.1995    -978.6977 
##       colorI       colorJ   claritySI2   claritySI1   clarityVS2 
##   -1440.3019   -2325.2224    2625.9500    3573.6880    4217.8291 
##   clarityVS1  clarityVVS2  clarityVVS1    clarityIF 
##    4534.8790    4967.1994    5072.0276    5419.6468
coef(updated_fit_2)
##  (Intercept)        carat      cutGood cutVery Good   cutPremium 
##   -6162.1964    8724.8278     527.6717     712.9909     744.0746 
##     cutIdeal       colorE       colorF       colorG       colorH 
##     861.5481    -223.8747    -313.4734    -508.2788    -996.4275 
##       colorI       colorJ   claritySI2   claritySI1   clarityVS2 
##   -1492.3535   -2210.0376    1635.1460    2580.1405    3240.4271 
##   clarityVS1  clarityVVS2  clarityVVS1    clarityIF 
##    3568.3900    3999.7614    4094.5453    4437.4743
```
A second way to achieve the same results as seen with `updated_fit_2` is to use
the `update` function to add to the call:

```r
updated_fit_3 <- update(original_fit, subset = carat < 2)
getCall(updated_fit_3)
## lm(formula = price ~ carat + cut + color + clarity, data = diamonds, 
##     subset = carat < 2)

coef(original_fit)
##  (Intercept)        carat      cutGood cutVery Good   cutPremium 
##   -7362.8022    8886.1289     655.7674     848.7169     869.3959 
##     cutIdeal       colorE       colorF       colorG       colorH 
##     998.2544    -211.6825    -303.3100    -506.1995    -978.6977 
##       colorI       colorJ   claritySI2   claritySI1   clarityVS2 
##   -1440.3019   -2325.2224    2625.9500    3573.6880    4217.8291 
##   clarityVS1  clarityVVS2  clarityVVS1    clarityIF 
##    4534.8790    4967.1994    5072.0276    5419.6468
all.equal(coef(updated_fit_3), coef(updated_fit_2))
## [1] TRUE
```
Lastly, before we move onto more interesting examples, you can always update
more than one part of a call with one `update` call


```r
updated_fit_4 <- update(original_fit, 
                        formula = . ~ depth, 
                        data    = dplyr::filter(diamonds, carat < 2), 
                        subset  = cut %in% c("Good", "Very Good"))
getCall(updated_fit_4)
## lm(formula = price ~ depth, data = dplyr::filter(diamonds, carat < 
##     2), subset = cut %in% c("Good", "Very Good"))
summary(updated_fit_4)
## 
## Call:
## lm(formula = price ~ depth, data = dplyr::filter(diamonds, carat < 
##     2), subset = cut %in% c("Good", "Very Good"))
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -3316.1 -2596.6  -954.8  1364.0 15272.6 
## 
## Coefficients:
##             Estimate Std. Error t value Pr(>|t|)    
## (Intercept)  5027.60     934.21   5.382 7.48e-08 ***
## depth         -24.33      15.07  -1.615    0.106    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 3187 on 16321 degrees of freedom
## Multiple R-squared:  0.0001598,	Adjusted R-squared:  9.852e-05 
## F-statistic: 2.608 on 1 and 16321 DF,  p-value: 0.1063
```

## Modifying a Variable on the Right Hand Side
Let's work with the `original_fit` object again and modify the right hand side
of the regression formula to use a centered and scaled version of `carat`.  (You
can quickly center and scale a variable by calling `scale`.  The default
behavior is to center and scale by subtracting off the mean and dividing by the
standard deviation.) If
you attempt to update the formula as `. ~ . + scale(carat)` where the `.` is a
reuse operator, the result will be nonsensical.


```r
scaled_fit_1 <- update(original_fit, formula = . ~ . + scale(carat))
getCall(scaled_fit_1)
## lm(formula = price ~ carat + cut + color + clarity + scale(carat), 
##     data = diamonds)
coef(scaled_fit_1)
##  (Intercept)        carat      cutGood cutVery Good   cutPremium 
##   -7362.8022    8886.1289     655.7674     848.7169     869.3959 
##     cutIdeal       colorE       colorF       colorG       colorH 
##     998.2544    -211.6825    -303.3100    -506.1995    -978.6977 
##       colorI       colorJ   claritySI2   claritySI1   clarityVS2 
##   -1440.3019   -2325.2224    2625.9500    3573.6880    4217.8291 
##   clarityVS1  clarityVVS2  clarityVVS1    clarityIF scale(carat) 
##    4534.8790    4967.1994    5072.0276    5419.6468           NA
```
Note that `carat` appears twice in the right hand side of the formula.  Further,
the regression coefficient for `scale(carat)` is `NA` as this vector in the
design matrix is a linear function of `carat`.  We've created a regression model
with a rank deficient design matrix.  Oops.

Obviously, we need to omit `carat` and replace with `scale(carat)` in the
updated formula.  Two ways to do this.  

1. Don't use the `.` on the right hand side
and write out the full right hand side yourself.  This option requires too much
effort and would suck to maintain.  

2. Continue to use the `.` on the right hand side and omit `carat` via `-`


```r
scaled_fit_2 <- update(original_fit, formula = . ~ . - carat + scale(carat))
getCall(scaled_fit_2)
## lm(formula = price ~ cut + color + clarity + scale(carat), data = diamonds)
coef(scaled_fit_2)
##  (Intercept)      cutGood cutVery Good   cutPremium     cutIdeal 
##    -272.2067     655.7674     848.7169     869.3959     998.2544 
##       colorE       colorF       colorG       colorH       colorI 
##    -211.6825    -303.3100    -506.1995    -978.6977   -1440.3019 
##       colorJ   claritySI2   claritySI1   clarityVS2   clarityVS1 
##   -2325.2224    2625.9500    3573.6880    4217.8291    4534.8790 
##  clarityVVS2  clarityVVS1    clarityIF scale(carat) 
##    4967.1994    5072.0276    5419.6468    4212.1250
```
Cool.  That worked well.

Now, what if we wanted to only center `carat` instead of centering and scaling?
This would require adding `scale(carat, scale = FALSE)` to the right hand side
of the formula.  Starting with the `scaled_fit_2` object we find that this task
can be difficult as the full text of `scale(carat)` needs to be omitted.  In the
following chunk you'll see that `scaled_fit_3` does not have the desired formula
whereas `scaled_fit_4` does.


```r
scaled_fit_3 <-
  update(scaled_fit_2, formula = . ~ . + scale(carat, scale = FALSE))
scaled_fit_4 <-
  update(scaled_fit_2, formula = . ~ . - scale(carat) + scale(carat, scale = FALSE))

getCall(scaled_fit_3)
## lm(formula = price ~ cut + color + clarity + scale(carat) + scale(carat, 
##     scale = FALSE), data = diamonds)
getCall(scaled_fit_4)
## lm(formula = price ~ cut + color + clarity + scale(carat, scale = FALSE), 
##     data = diamonds)
```

Okay, one more problem.  Let's start with `scaled_fit_4` and scale, but not
center `carat`.  In the chunk below, `scaled_fit_5` does not have the desired
result, but `scaled_fit_6` does.

```r
scaled_fit_5 <-
  update(scaled_fit_4,
         formula = . ~ . - scale(carat) + scale(carat, center = TRUE))
scaled_fit_6 <-p
## Error in eval(expr, envir, enclos): object 'p' not found
  update(scaled_fit_4,
         formula = . ~ . - scale(carat, scale = FALSE) + scale(carat, center = TRUE))
## 
## Call:
## lm(formula = price ~ cut + color + clarity + scale(carat, center = TRUE), 
##     data = diamonds)
## 
## Coefficients:
##                 (Intercept)                      cutGood  
##                      -272.2                        655.8  
##                cutVery Good                   cutPremium  
##                       848.7                        869.4  
##                    cutIdeal                       colorE  
##                       998.3                       -211.7  
##                      colorF                       colorG  
##                      -303.3                       -506.2  
##                      colorH                       colorI  
##                      -978.7                      -1440.3  
##                      colorJ                   claritySI2  
##                     -2325.2                       2625.9  
##                  claritySI1                   clarityVS2  
##                      3573.7                       4217.8  
##                  clarityVS1                  clarityVVS2  
##                      4534.9                       4967.2  
##                 clarityVVS1                    clarityIF  
##                      5072.0                       5419.6  
## scale(carat, center = TRUE)  
##                      4212.1
                                                                           
getCall(scaled_fit_5)
## lm(formula = price ~ cut + color + clarity + scale(carat, scale = FALSE) + 
##     scale(carat, center = TRUE), data = diamonds)
getCall(scaled_fit_6)
## Error in getCall(scaled_fit_6): object 'scaled_fit_6' not found
```

So, what do you think?  The `update` function is great, but is has some
limitations.  Imaging if you had a function of a variable with several options,
or just one option with a very long value.  Update might not be that useful.
For example, instead of scaling `carat`, let's move it into bins using the
`cut()` function.  (The fact that there is a variable and a function both called
`cut` on the right hand side could be confusing.  I selected this data set for
this example specifically because it had a meaningful variable name of cut.
We will see why this is important later.)


```r
cut_fit_1 <-
  update(original_fit,
         formula = . ~ . - carat + 
                       cut(carat, 
                           breaks = seq(0, 5.5, by = 0.5),
                           right = FALSE)
         )
names(coef(cut_fit_1))
##  [1] "(Intercept)"                                                     
##  [2] "cutGood"                                                         
##  [3] "cutVery Good"                                                    
##  [4] "cutPremium"                                                      
##  [5] "cutIdeal"                                                        
##  [6] "colorE"                                                          
##  [7] "colorF"                                                          
##  [8] "colorG"                                                          
##  [9] "colorH"                                                          
## [10] "colorI"                                                          
## [11] "colorJ"                                                          
## [12] "claritySI2"                                                      
## [13] "claritySI1"                                                      
## [14] "clarityVS2"                                                      
## [15] "clarityVS1"                                                      
## [16] "clarityVVS2"                                                     
## [17] "clarityVVS1"                                                     
## [18] "clarityIF"                                                       
## [19] "cut(carat, breaks = seq(0, 5.5, by = 0.5), right = FALSE)[0.5,1)"
## [20] "cut(carat, breaks = seq(0, 5.5, by = 0.5), right = FALSE)[1,1.5)"
## [21] "cut(carat, breaks = seq(0, 5.5, by = 0.5), right = FALSE)[1.5,2)"
## [22] "cut(carat, breaks = seq(0, 5.5, by = 0.5), right = FALSE)[2,2.5)"
## [23] "cut(carat, breaks = seq(0, 5.5, by = 0.5), right = FALSE)[2.5,3)"
## [24] "cut(carat, breaks = seq(0, 5.5, by = 0.5), right = FALSE)[3,3.5)"
## [25] "cut(carat, breaks = seq(0, 5.5, by = 0.5), right = FALSE)[3.5,4)"
## [26] "cut(carat, breaks = seq(0, 5.5, by = 0.5), right = FALSE)[4,4.5)"
## [27] "cut(carat, breaks = seq(0, 5.5, by = 0.5), right = FALSE)[4.5,5)"
## [28] "cut(carat, breaks = seq(0, 5.5, by = 0.5), right = FALSE)[5,5.5)"
```
Now, we have a regression model for price with the `cut`, `color`, and `clarity`
accounted for, and a 11 level factor for `carat`, the interval `[0, 0.5)` is the
reference level here.

If you only had the `cut_fit_1` object to start with, using the `update`
function to modify the options passed to `cut()` would be a pain.  Having to
type out the old `cut()` call exactly as provide and then replace with a new
`cut()` call. This is too much work and a pain to maintain.  Just start with a
fresh call to `lm`.  Or, let's be clever and build some new tools to do this
work.

## Modifying a `call` within a `formula`
First, let's look at the structure of a formula.  We'll use the object `f`,
defined below, as the primary object in this example.

```r
f <- price ~ color + cut + clarity + cut(carat, breaks = seq(0, 5.5, by = 0.5), right = FALSE) 
f
## price ~ color + cut + clarity + cut(carat, breaks = seq(0, 5.5, 
##     by = 0.5), right = FALSE)
str(f)
## Class 'formula'  language price ~ color + cut + clarity + cut(carat, breaks = seq(0, 5.5, by = 0.5),      right = FALSE)
##   ..- attr(*, ".Environment")=<environment: R_GlobalEnv>
is.list(f)
## [1] FALSE
is.recursive(f)
## [1] TRUE
```

We have a language object with it an
[environment](http://adv-r.had.co.nz/Environments.html) attribute. This object
is not a `list`, but it is recursive ("`is.recursive(x)` returns `TRUE` if `x`
has a recursive (list-like) structure and `FALSE` otherwise).  So, if we
recursively apply `as.list` to `f` object we see a controlled deconstruction of
the formula object.


```r
as.list(f)
## [[1]]
## `~`
## 
## [[2]]
## price
## 
## [[3]]
## color + cut + clarity + cut(carat, breaks = seq(0, 5.5, by = 0.5), 
##     right = FALSE)
lapply(as.list(f), as.list) 
## [[1]]
## [[1]][[1]]
## `~`
## 
## 
## [[2]]
## [[2]][[1]]
## price
## 
## 
## [[3]]
## [[3]][[1]]
## `+`
## 
## [[3]][[2]]
## color + cut + clarity
## 
## [[3]][[3]]
## cut(carat, breaks = seq(0, 5.5, by = 0.5), right = FALSE)
```

What happens when we apply `as.list` to the third element of the `.Last.value`?

```r
lapply(lapply(as.list(f), as.list)[[3]], as.list)
## [[1]]
## [[1]][[1]]
## `+`
## 
## 
## [[2]]
## [[2]][[1]]
## `+`
## 
## [[2]][[2]]
## color + cut
## 
## [[2]][[3]]
## clarity
## 
## 
## [[3]]
## [[3]][[1]]
## cut
## 
## [[3]][[2]]
## carat
## 
## [[3]]$breaks
## seq(0, 5.5, by = 0.5)
## 
## [[3]]$right
## [1] FALSE
```
Look at the elements of the third element, the `cut()` call is the first
sub-element followed by the arguments `x` (implicitly) and `breaks` (explicitly).

This is great!  If a formula is reconstructed recursively then we can access the
arguments of calls within the formula.

Let's start building a function to fully reconstruct a `formula` object into
it's parts.


```r
decon <- function(x) {
  if (is.recursive(x)) {
    lapply(as.list(x), decon)
  } else { 
    x
  }
}
```
The `decon` function rips apart any recursive object until only the
non-recursive elements remain.  Passing `f` to `decon` yields:

```r
decon(f) 
## [[1]]
## `~`
## 
## [[2]]
## price
## 
## [[3]]
## [[3]][[1]]
## `+`
## 
## [[3]][[2]]
## [[3]][[2]][[1]]
## `+`
## 
## [[3]][[2]][[2]]
## [[3]][[2]][[2]][[1]]
## `+`
## 
## [[3]][[2]][[2]][[2]]
## color
## 
## [[3]][[2]][[2]][[3]]
## cut
## 
## 
## [[3]][[2]][[3]]
## clarity
## 
## 
## [[3]][[3]]
## [[3]][[3]][[1]]
## cut
## 
## [[3]][[3]][[2]]
## carat
## 
## [[3]][[3]]$breaks
## [[3]][[3]]$breaks[[1]]
## seq
## 
## [[3]][[3]]$breaks[[2]]
## [1] 0
## 
## [[3]][[3]]$breaks[[3]]
## [1] 5.5
## 
## [[3]][[3]]$breaks$by
## [1] 0.5
## 
## 
## [[3]][[3]]$right
## [1] FALSE
```
where we can see a hierarchy for each element and sub-element.  Take careful
notice of element `[[3]][[3]][[1]]`.  When the deconstruction of the formula
runs into the `cut()` call, the first element is the name of the call itself,
followed by the arguments thereto.  This is illustrated again with respect to
the `seq` call.

Now, before we try to modify any arguments, we need `decon` to return a
`formual` object.  After all, the point of this is to modify a `formula`. By
wrapping the `lapply` in a `as.call` if `x` is recursive we gain the desired
behavior.


```r
decon <- function(x) {
  if (is.recursive(x)) {
    as.call(lapply(as.list(x), decon))
  } else { 
    x
  }
}
decon(f)
## price ~ color + cut + clarity + cut(carat, breaks = seq(0, 5.5, 
##     by = 0.5), right = FALSE)
all.equal(decon(f), f)
## [1] TRUE
```
Why?  Well, each deconstruction is a list with an operator, a named call, as the
first element followed by two arguments.  A simple example with addition:

```r
as.call(list(`+`, 1, 2))
## .Primitive("+")(1, 2)

eval(as.call(list(`+`, 1, 2))) 
## [1] 3
```
By wrapping the `lapply` in the `as.call` within the `decon` function we
preserve the unevaluated calls until the end of the recursion when the call is
implicitly evaluated.

The next step in our journey is to modify `decon` such that arguments to the
`cut` call can be updated.  Specifically, we want to change the value of the
`breaks` argument.  Just to be clear, the `stats::update` function can't do
this.


```r
update(cut(carat, breaks = c(1, 1)), breaks = c(3, 4))
## Error in cut(carat, breaks = c(1, 1)): object 'carat' not found
```

We are looking for a call named `cut`, and to modify the breaks argument.
Using `is.call` will differentiate between the `cut` call and the `cut`
variable.

```r
decon <- function(x, nb) {
  if (is.call(x) && grepl("cut", deparse(x[[1]]))) {
    x$breaks <- nb
    x
  } else if (is.recursive(x)) {
    as.call(lapply(as.list(x), decon, nb))
  } else {
    x
  }
}
f
## price ~ color + cut + clarity + cut(carat, breaks = seq(0, 5.5, 
##     by = 0.5), right = FALSE)
decon(f, c(0, 3))
## price ~ color + cut + clarity + cut(carat, breaks = c(0, 3), 
##     right = FALSE)
```
We're almost there.  The return from `decon` is a `formula`.  However, we have
not dealt with the environments.  Let's place
`decon` within another function, call it `newbreaks` and then handle
environments and calls.  While not necessary, to make it clear which functions
are being called we will give use the name `local_decon` within the `newbreaks`
function.

```r
newbreaks <- function(form, nb) {
  local_decon <- function(x, nb) {
    if (is.call(x) && grepl("cut", deparse(x[[1]]))) {
      x$breaks <- nb
      x
    } else if (is.recursive(x)) {
      as.call(lapply(as.list(x), local_decon, nb))
    } else {
      x
    }
  }

  out <- lapply(as.list(form), local_decon, nb)
  out <- eval(as.call(out))
  environment(out) <- environment(form)
  out
}

newbreaks(f, c(0, 3)) 
## price ~ color + cut + clarity + cut(carat, breaks = c(0, 3), 
##     right = FALSE)
```

This is great!  Now we are able to modify the `breaks` within a `cut` call
within a `formula`!  In practice we could do the following:


```r
new_fit_1 <-
  update(cut_fit_1, formula = newbreaks(formula(cut_fit_1), c(0, 1, 2)))
new_fit_2 <-
  update(cut_fit_1, formula = newbreaks(formula(cut_fit_1), c(0, 1, 3, 5)))
new_fit_3 <-
  update(cut_fit_1, formula = newbreaks(formula(cut_fit_1), seq(0, 5.5, by = 1.25)))

names(coef(new_fit_1))
##  [1] "(Intercept)"                                        
##  [2] "cutGood"                                            
##  [3] "cutVery Good"                                       
##  [4] "cutPremium"                                         
##  [5] "cutIdeal"                                           
##  [6] "colorE"                                             
##  [7] "colorF"                                             
##  [8] "colorG"                                             
##  [9] "colorH"                                             
## [10] "colorI"                                             
## [11] "colorJ"                                             
## [12] "claritySI2"                                         
## [13] "claritySI1"                                         
## [14] "clarityVS2"                                         
## [15] "clarityVS1"                                         
## [16] "clarityVVS2"                                        
## [17] "clarityVVS1"                                        
## [18] "clarityIF"                                          
## [19] "cut(carat, breaks = c(0, 1, 2), right = FALSE)[1,2)"
names(coef(new_fit_2))
##  [1] "(Intercept)"                                           
##  [2] "cutGood"                                               
##  [3] "cutVery Good"                                          
##  [4] "cutPremium"                                            
##  [5] "cutIdeal"                                              
##  [6] "colorE"                                                
##  [7] "colorF"                                                
##  [8] "colorG"                                                
##  [9] "colorH"                                                
## [10] "colorI"                                                
## [11] "colorJ"                                                
## [12] "claritySI2"                                            
## [13] "claritySI1"                                            
## [14] "clarityVS2"                                            
## [15] "clarityVS1"                                            
## [16] "clarityVVS2"                                           
## [17] "clarityVVS1"                                           
## [18] "clarityIF"                                             
## [19] "cut(carat, breaks = c(0, 1, 3, 5), right = FALSE)[1,3)"
## [20] "cut(carat, breaks = c(0, 1, 3, 5), right = FALSE)[3,5)"
names(coef(new_fit_3))
##  [1] "(Intercept)"                                                           
##  [2] "cutGood"                                                               
##  [3] "cutVery Good"                                                          
##  [4] "cutPremium"                                                            
##  [5] "cutIdeal"                                                              
##  [6] "colorE"                                                                
##  [7] "colorF"                                                                
##  [8] "colorG"                                                                
##  [9] "colorH"                                                                
## [10] "colorI"                                                                
## [11] "colorJ"                                                                
## [12] "claritySI2"                                                            
## [13] "claritySI1"                                                            
## [14] "clarityVS2"                                                            
## [15] "clarityVS1"                                                            
## [16] "clarityVVS2"                                                           
## [17] "clarityVVS1"                                                           
## [18] "clarityIF"                                                             
## [19] "cut(carat, breaks = c(0, 1.25, 2.5, 3.75, 5), right = FALSE)[1.25,2.5)"
## [20] "cut(carat, breaks = c(0, 1.25, 2.5, 3.75, 5), right = FALSE)[2.5,3.75)"
## [21] "cut(carat, breaks = c(0, 1.25, 2.5, 3.75, 5), right = FALSE)[3.75,5)"
```
         

## Why, Why would you ever need this?
I hope the above examples would have answered this question.  If not, here was
my motivation.  I have been working with B-splines, a lot.  My Ph.D.
dissertation focuses on B-spline regression models.  I needed to be able to
update a regression object with a new formula differing only by the internal
knot locations within a spline.  Originally, this meant using the `splines::bs`
call and adjusting the `knots` argument while preserving the values passed to
the `degree`, `intercept` and `Boundary.knots` arguments.  If you are familiar
with the `splines::bs` call then you'll know that the there are default
arguments to each of these after mentioned arguments.  

Further, the objects I really needed to update where more complex than just an
`lm` object and, as new software goes, had an ever changing API. Construction of
calls was a lot of overhead.  When the only thing that needed to be updated was
one argument in one call within a formula it seemed reasonable to find a
solution to do exactly what I needed and no more.

Once I publicly release my `cpr` package you'll find, if you dig into the source
code, functions `cpr:::newknots` and `cpr:::newdfs` to be critical functions in
the implementation.


*Acknowledgements:*
I didn't figure out how to do this completely on my own.  I had posed a question
on [stackoverflow](http://stackoverflow.com/q/25272387/1104685) which was the
basis for this post  and extensions. 
