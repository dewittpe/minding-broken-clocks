---
title: "Sccm"
date: 2018-05-11T14:46:45-06:00
draft: false
---



# <a name="sccm"></a> sccm: Schwarz-Christoffel Conformal Mapping

[back to top](#top)

<a href="https://www.github.com/dewittpe/sccm" title="sccm on GitHub" target="_blank">
  <i class="fa fa-github fa-2x"></i>
  <span class="label">sccm on GitHub</span>
</a>

[![Project Status: Active – The project has reached a stable, usable state and is being actively developed.](http://www.repostatus.org/badges/latest/active.svg)](http://www.repostatus.org/#active)
[![Build Status](https://travis-ci.org/dewittpe/sccm.svg?branch=master)](https://travis-ci.org/dewittpe/sccm)
[![Coverage Status](https://img.shields.io/codecov/c/github/dewittpe/sccm/master.svg)](https://codecov.io/github/dewittpe/sccm?branch=master)
[![CRAN_Status_Badge](http://www.r-pkg.org/badges/version/sccm)](https://cran.r-project.org/package=sccm)

An R package Providing a conformal mapping of one 2D polygon to a rectangular
region via the Schwarz-Christoffel theorem.

Methods are provide to find a convex hull for an arbitrary set (x, y)
coordinates.  This hull, and the points within, are, via an (inverse)
Schwarz-Christoffel mapping, mapped to the unit disk.  A second
Schwarz-Christoffel mapping from the unit disk to an arbitrary rectangle is use
to finish the conformal mapping.

This package builds hulls via Andrew's monotone chain algorithm implemented in
C++. The Schwarz-Christoffel mappings are provided by the FORTRAN SCPACK by
Lloyd N. Trefethen.

**Not on CRAN** The FORTRAN code writes to the terminal directly.  It is
possible that when the FORTRAN code errors that the R session will be terminated
as well.  For these reasons the package does not pass CRAN checks.  My FORTRAN
skills are minimal, and it is a low priority, to fix this problem.  If you'd
like to help fix see [Issue #1](https://github.com/dewittpe/sccm/issues/1).

