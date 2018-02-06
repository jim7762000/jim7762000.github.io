---
layout: post
title: Splitting characters in Rcpp
---

We usually need to process the raw data by ourself, the character type of data is the most common type of raw data. I demonstrate a example to simply split the characters.

``` R
library(data.table)
library(magrittr)
library(Rcpp)
library(inline)
sourceCpp(code = '
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
CharacterMatrix dat_split_f( std::vector< std::string > strings,
NumericVector loc) {
  int loc_len = loc.size(), num_strings = strings.size();
  CharacterMatrix output(num_strings, loc_len);
  for( int j=0; j < num_strings; j++ )
  {
    for (int i=0; i < loc_len-1; i++)
      output(j, i) = strings[j].substr(loc[i], loc[i+1] - loc[i]);
  }
  return output;
}')

# size is 400000
dat = fread(paste0(rep("001female2019920404\n002male  3019920505\n003male  4019920606\n004female5019920707\n", 100000), collapse=""),
  sep="\n", sep2="", header=FALSE)

tt = proc.time()
dat_split = dat_split_f(dat[[1]], c(0, 3, 9, 11, 19)) %>% data.table()
dat_split[, ':='(V1 = as.numeric(V1), V3 = as.numeric(V3))]
proc.time() - tt
#   user  system elapsed
#   0.97    0.06    1.03

# size is 4000000
dat = fread(paste0(rep("001female2019920404\n002male  3019920505\n003male  4019920606\n004female5019920707\n", 1000000), collapse=""),
  sep="\n", sep2="", header=FALSE)

tt = proc.time()
dat_split = dat_split_f(dat[[1]], c(0, 3, 9, 11, 19)) %>% data.table()
dat_split[, ':='(V1 = as.numeric(V1), V3 = as.numeric(V3))]
proc.time() - tt
#   user  system elapsed
#   7.73    0.21    7.98

## by using regular expression
dat = fread(paste0(rep("001female2019920404\n002male  3019920505\n003male
4019920606\n004female5019920707\n", 100000), collapse=""),sep="\n",
sep2="",header=FALSE)

tt = proc.time()
dat_regex = dat %>% select(V1) %>% extract2(1) %>%
  regexec("([0-9]{3})(female|male\\s{2})([0-9]{2})([0-9]{8})", text = .)
dat_split = dat %>% select(V1) %>% extract2(1) %>%
  regmatches(dat_regex) %>% do.call(rbind, .) %>% data.table() %>%
  select(2:ncol(.)) %>% setnames(c("id", "gender", "age", "birthday"))
proc.time() - tt
```
