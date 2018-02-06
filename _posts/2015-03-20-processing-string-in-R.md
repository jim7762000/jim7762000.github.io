---
layout: post
title: Processing string in R (Rcpp and rJava)
---

There is a example for processing string in R.

Before we use rJava, we need a class file first. The `regex_java.java` is shown in below and compilation can be done with command `javac regex_java.java`.

``` java
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class regex_java {
  public static String[] string_split_1d(String[] string_array, int[] loc) {
    String[] out = new String[string_array.length*(loc.length - 1)];
    for (int j = 0; j < loc.length - 1; j++)
      for (int i = 0; i < string_array.length; i++)
        out[i+j*string_array.length] = string_array[i].subSequence(loc[j], loc[j+1]).toString();
    return out;
  }
  public static String[][] string_split(String[] string_array, int[] loc) {
    String[][] out = new String[string_array.length][loc.length - 1];
    for (int i = 0; i < string_array.length; i++)
      for (int j = 0; j < loc.length - 1; j++)
        out[i][j] = string_array[i].subSequence(loc[j], loc[j+1]).toString();
    return out;
  }
  public static String[] regex_java(String[] string_array, String patternStr, int pattern_reconize) {
    String[] out = new String[string_array.length*pattern_reconize];
    Pattern pattern = Pattern.compile(patternStr);
    for (int j = 0; j < string_array.length; j++)
    {
      Matcher matcher = pattern.matcher(string_array[j]);
      boolean matchFound = matcher.find();
      while(matchFound)
      {
        for(int i = 1; i <= matcher.groupCount(); i++)
          out[j*pattern_reconize+i-1] = matcher.group(i);
        if(matcher.end() + 1 <= string_array[j].length())
          matchFound = matcher.find(matcher.end());
        else
          break;
      }
    }
    return out;
  }
  public static void main(String[] args) {
  }
}
```

Next, we demonstrate the performance between R, Rcpp and rJava. However, Rcpp cannot process string in regular expression because the standard of C++ does not support regular expression before C++14. Note that g++ have `regexp.h` to support regular expression in linux.

``` R
library(data.table)
library(plyr)
library(dplyr)
library(magrittr)
dat = fread(paste0(rep("001female2019920404\n002male  3019920505\n003male  4019920606\n004female5019920707\n", 100000), collapse=""), sep="\n", sep2="",header=FALSE)

## using R
tt = proc.time()
dat_regex = dat %>% select(V1) %>% extract2(1) %>%
  regexec("([0-9]{3})(female|male\\s{2})([0-9]{2})([0-9]{8})", text = .)
dat_split2 = dat %>% select(V1) %>% extract2(1) %>%
  regmatches(dat_regex) %>% do.call(rbind, .) %>% data.table() %>%
  select(2:ncol(.)) %>% setnames(c("id", "gender", "age", "birthday"))
proc.time() - tt
#   user  system elapsed
#  10.19    0.14   10.96

## using Rcpp
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

tt = proc.time()
dat_split = dat_split_f(dat[[1]], c(0, 3, 9, 11, 19)) %>% data.table() %>%
  select(1:4) %>% setnames(c("id", "gender", "age", "birthday"))
proc.time() - tt
#   user  system elapsed
#   1.12    0.08    1.20

## using rJava
library(rJava)
.jinit()
.jaddClassPath( "D:\\Program\\R\\some_code\\regex_java")
regex_java = .jnew("regex_java")

tt = proc.time()
output = dat %>% select(V1) %>% extract2(1) %>%
  .jcall(regex_java, "[S", "string_split_1d",  .,
    as.integer(c(0, 3, 9, 11, 19))) %>% matrix(ncol=4) %>% data.table() %>%
  setnames(c("id", "gender", "age", "birthday"))
proc.time() - tt
#   user  system elapsed
#   3.24    0.04    2.55

tt = proc.time()
output2 = dat %>% select(V1) %>% extract2(1) %>%
  .jcall(regex_java, "[[Ljava/lang/String;", "string_split",  .,
    as.integer(c(0, 3, 9, 11, 19)), simplify = TRUE) %>% data.table() %>%
  setnames(c("id", "gender", "age", "birthday"))
proc.time() - tt
#   user  system elapsed
#   3.46    0.05    2.92

tt = proc.time()
pattern = "(\\d{3})(female|male\\s{2})(\\d{2})(\\d{8})"
size_recognize = "(\\((?>[^()]+|(?R))*\\))" %>% gregexpr(pattern, perl = TRUE) %>%
  extract2(1) %>% length() %>% as.integer()
output3 = dat %>% select(V1) %>% extract2(1) %>%
  .jcall(regex_java, "[S", "regex_java",  ., pattern, size_recognize) %>%
    matrix(ncol = 4, byrow=TRUE) %>%
    data.table() %>% setnames(c("id", "gender", "age", "birthday"))
proc.time() - tt
#   user  system elapsed
#   4.66    0.13    3.38

all.equal(output, output2)    # TRUE
all.equal(output, dat_split)  # TRUE
all.equal(output, dat_split2) # TRUE
all.equal(output, output3) # TRUE
```
