---
layout: post
title: Tables combining
---

A simple log for doing a job of mapreduce in python.

There are two tables:

``` R
library(data.table)
library(plyr)
library(dplyr)
library(magrittr)
library(reshape2)
ref_m = data.table(
  "gene_a" = c("A", "B", "C"),
  "Chromosome" = c("1", "X", "2"),
  "gene_start" = c(25000, 1000, 0),
  "gene_end" = c(50000, 2000, 800)
) %>% tbl_dt(FALSE)
ref_m
#   gene_a Chromosome gene_start gene_end
# 1      A          1      25000    50000
# 2      B          X       1000     2000
# 3      C          2          0      800

dat = data.table(
  "Probe_b" = c("a1", "a2", "a3", "a4", "a5"),
  "Chromosome" = c("2", "4", "1", "X", "1"),
  "Chr_s" = as.integer(c(175, 600, 23575, 1010, 30000)),
  "Chr_e" = as.integer(c(200, 625, 23600, 1035, 30025)),
  stringsAsFactors = FALSE
) %>% tbl_dt(FALSE)
dat
#   Probe_b  Chromosome Chr_s Chr_e
# 1      a1           2   175   200
# 2      a2           4   600   625
# 3      a3           1 23575 23600
# 4      a4           X  1010  1035
# 5      a5           1 30000 30025

# we want combine these two table like following by referring the Chromosome and checking that the range of Chr_s and Chr_e is fallen into range of gene_start and gene_end .
#   gene_a  match
# 1      A  a5
# 2      B  a4
# 3      C  a1

## first method
combine_f = function(dat, ref_m){
  check_f = function(v, ref_m){
    loc <- match(v$Chromosome, ref_m$Chromosome, nomatch = 0)
    if(length(loc) != 0 && loc > 0)  # 避免loc出現integer(0)的情況
    {
      if(v$Chr_s >= ref_m[loc,]$gene_s && v$Chr_e <= ref_m[loc,]$gene_e)
        return(as.character(ref_m[loc,]$gene_a))
      else
        return(NA)
    }
    else
      return(NA)
  }
  result = rep(NA, nrow(dat))
  aaply(1:nrow(dat), 1, function(v) check_f(dat[v,], ref_m)) %>%
    {tapply(as.character(dat$Probe_b), ., c)}
}
combine_f(dat, ref_m)
#    A    B    C
# "a5" "a4" "a1"

## second method
combine_f2 = function(dat, ref_m){
  loc_ref_v = match(dat$Chromosome, ref_m$Chromosome)
  loc_dat_v = loc_ref_v > 0
  loc_dat_v = loc_dat_v[!is.na(loc_dat_v)]
  loc_ref_v = loc_ref_v[loc_dat_v]
  loc = dat$Chr_s[loc_dat_v] >= ref_m$gene_start[loc_ref_v] &
        dat$Chr_e[loc_dat_v] <= ref_m$gene_end[loc_ref_v]
  tapply(dat$Probe_b[loc_dat_v][loc], ref_m$gene_a[loc_ref_v[loc]], c)
}
combine_f2(dat, ref_m)
#    A    B    C
# "a5" "a4" "a1"

## third method
combine_f3 = function(dat, ref_m){
  merge(dat, ref_m, by="Chromosome") %>%
    filter((Chr_s > gene_start) & (Chr_e < gene_end)) %>%
    select(one_of(c("gene_a", "Probe_b"))) %>%
    dlply(.(gene_a), function(x) x$Probe_b)
}
combine_f3(dat, ref_m)

## test the performance
N = 5000; k = 20
ref_m = data.table(
  gene_a = LETTERS[1:k],
  Chromosome = sample(c(as.character(1:25), "X"), k),
  gene_start = sample(seq(0, 50000, 200), k)) %>%
  mutate(gene_end = gene_start+sample(seq(600, 15000, 200), k)) %>%
  tbl_dt(FALSE)
dat = data.table(
  Probe_b = paste0("a", 1:N),
  Chromosome = sample(c(as.character(1:25), "X"), N, replace=TRUE),
  Chr_s = as.integer(sample(seq(0, 50000, 5), N, replace=TRUE))) %>%
  mutate(Chr_e = Chr_s + sample(seq(1000, 10000, 100), N,
    replace=TRUE)) %>% tbl_dt(FALSE)

all.equal( combine_f(dat, ref_m),  combine_f2(dat, ref_m))
# TRUE
all.equal( combine_f2(dat, ref_m),  combine_f3(dat, ref_m), check.attributes =FALSE)
# TRUE

library(rbenchmark)
benchmark(combine_f(dat, ref_m), combine_f2(dat, ref_m),
  combine_f3(dat, ref_m), replications = 20,
  columns = c("test", "replications", "elapsed", "relative"),
  order = "relative")
#                     test replications elapsed relative
# 2 combine_f2(dat, ref_m)           20    0.11    1.000
# 3 combine_f3(dat, ref_m)           20    0.38    3.455
# 1  combine_f(dat, ref_m)           20  406.35 3694.091
```

The second method has the best performance, but it is a hard coding. Therefore, I recommend the third method.
