---
layout: post
title: "library多個套件，並自動安裝沒安裝的套件"
---

廢話不多說，直接上code

``` R
library(pipeR)
library_mul <- function(..., lib.loc = NULL, quietly = FALSE, warn.conflicts = TRUE){
  pkgs <- as.list(substitute(list(...))) %>>% sapply(as.character) %>>% setdiff("list")
  if (any(!pkgs %in% installed.packages()))
    install.packeges(pkgs[!pkgs %in% installed.packages()])
  sapply(pkgs, library, character.only = TRUE, lib.loc = lib.loc, quietly = quietly) %>>% invisible
}
library_mul(httr, pipeR, data.table)
```




