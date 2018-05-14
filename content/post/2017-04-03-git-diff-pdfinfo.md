---
title:  "git diff pdfs"
date:   2017-04-03 00:00:00
tags: [git]
---

Binary files and git repositories can be a pain, especially when looking at the
`git diff` output.  Here are two helpful things to do with git to make working
with .pdfs less painful.

The standard output from `git diff` when a tracked `.pdf` has changed is

    diff --git a/dissertation.pdf b/dissertation.pdf
    index 2a9ec08..d3aaa79 100644
    Binary files a/dissertation.pdf and b/dissertation.pdf differ

Not the most helpful output.  The two things I have found to be helpful are

1. A `git difftool` to call [`diffpdf`](https://packages.debian.org/sid/diffpdf).
2. Have the differences in the `pdfinfo` returned when calling `git diff`.

Note, I'm writing this on a Debian machine.  Set up for other Linux
distributions, Winblows and Mac may differ.

Setting up git to use `diffpdf` is simple.  In either you project or global config
file add the following lines

    [difftool "diffpdf"]
      cmd = diffpdf \"$LOCAL\" \"$REMOTE\"

From the command line

    git difftool --no-prompt --tool=diffpdf dissertation.pdf  

will open the `diffpdf` gui and show the differences, changes, in the
`dissertation.pdf` file.  I find this to be the most helpful when comparing
different versions of the `.pdf` file with/for non-git users.  For example,
version-0.9.1 versus the current version

    git difftool --no-prompt --tool=diffpdf version-0.9.1:dissertation.pdf dissertation.pdf

Now, from time to time, I know that the `.pdf` changes are limited, or can
easily be assess by looking at the metadata.   To have the output of `git diff`
show the changes in the pdf metadata do the following.

Add, or add to, `.gitattributes` file.  Setting one up for all your repos is done
by adding the following to you global git config file

    git config --global core.attributesfile ~/.gitattributes
    echo "*.pdf diff=diffpdfinfo" >> ~/.gitattributes 

If you only want this to work in one particular repository you need only create
the `.gitattributes` file in the project root directory.

We need to write the `diffpdfinfo` script and save it somewhere in your `PATH`.

    #!/bin/bash

    pdfinfo $1 > .localpdfmetadata
    pdfinfo $2 > .remotepdfmetadata

    diff .localpdfmetadata .remotepdfmetadata

Remember to made the script executable via `chmod a+x`.

Now `git diff` has meaningful output for the `.pdf` file.  For example, a quick
recompile of the `.pdf` document results in the `diffpdf` tool telling me the
two files are the same.  However, `git diff` tells me they are different,
specifically in that the creation date and modification dates have change.


    $ git diff

    diff --git a/dissertation.pdf b/dissertation.pdf
    index 2a9ec08..edd66d8 100644
    --- a/dissertation.pdf
    +++ b/dissertation.pdf
    @@ -4,8 +4,8 @@ Keywords:
     Author:         
     Creator:        LaTeX with hyperref package
     Producer:       pdfTeX-1.40.15
    -CreationDate:   Wed Mar 29 14:55:27 2017
    -ModDate:        Wed Mar 29 14:55:27 2017
    +CreationDate:   Mon Apr  3 00:41:05 2017
    +ModDate:        Mon Apr  3 00:41:05 2017
     Tagged:         no
     UserProperties: no
     Suspects:       no

Diffing a draft version to the current version there should be some changes in
the metadata.  The creation and modification dates have changed, but so has the
number of pages and the file size.

    $ git diff version-0.9.1:dissertation.pdf dissertation.pdf

    diff --git a/dissertation.pdf b/dissertation.pdf
    index bac8a42..edd66d8 100644
    --- a/dissertation.pdf
    +++ b/dissertation.pdf
    @@ -4,17 +4,17 @@ Keywords:
     Author:         
     Creator:        LaTeX with hyperref package
     Producer:       pdfTeX-1.40.15
    -CreationDate:   Sat Mar 18 16:03:34 2017
    -ModDate:        Sat Mar 18 16:03:34 2017
    +CreationDate:   Mon Apr  3 00:41:05 2017
    +ModDate:        Mon Apr  3 00:41:05 2017
     Tagged:         no
     UserProperties: no
     Suspects:       no
     Form:           none
     JavaScript:     no
    -Pages:          130
    +Pages:          140
     Encrypted:      no
     Page size:      612 x 792 pts (letter)
     Page rot:       0
    -File size:      13127074 bytes
    +File size:      23775044 bytes
     Optimized:      no
     PDF version:    1.5


