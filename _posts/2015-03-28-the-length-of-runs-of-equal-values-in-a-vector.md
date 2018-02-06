---
layout: post
title: The length of runs of equal values in a vector
---

We usually encounter the problem for counting the length of equal values repeatedly. `rle` is the build-in command in R to solve this problem which is consist of `diff` and `which`. But it is not so fast, I write another version of `rle` in Rcpp.

``` R
library(Rcpp)
library(RcppArmadillo)
sourceCpp(code = '
// [[Rcpp::depends(RcppArmadillo)]]
#include <RcppArmadillo.h>
using namespace Rcpp;
using namespace arma;

template <int RTYPE>
List rle_template(const Vector<RTYPE>& x) {
 IntegerVector tmp = seq_len(x.size()-1);
 LogicalVector loc = head(x, x.size()-1) != tail(x, x.size()-1);
 IntegerVector y = tmp[loc | is_na(loc)];
 y.push_back(x.size());
 Col<int> y2(y.begin(), y.size());
 y2.insert_rows(0, zeros< Col<int> >(1));
 IntegerVector y3 = wrap(y2);
 return List::create(Named("lengths") = diff(y3),
   Named("values") = x[y-1]);
}

// [[Rcpp::export]]
List rle_cpp(SEXP x){
 switch( TYPEOF(x) ) {
 case INTSXP: return rle_template<INTSXP>(x);
 case REALSXP: return rle_template<REALSXP>(x);
 case STRSXP: return rle_template<STRSXP>(x);
 }
 return R_NilValue;
}')

x <- rev(rep(6:10, 1:5))
all.equal(rle(x), rle_cpp(x), check.attributes = FALSE)
# TRUE

N = 100000
testVector = rep(sample(1:150, round(N/10), TRUE),
  sample(1:25, round(N/10), TRUE))
all.equal(rle(testVector), rle_cpp(testVector),
  check.attributes = FALSE)
# TRUE
library(rbenchmark)
benchmark(rle(testVector), rle_cpp(testVector),
  columns = c("test", "replications","elapsed",
  "relative"), replications=100, order="relative")
#         test replications elapsed relative
# 2 rle_cpp(x)          100    0.34    1.000
# 1     rle(x)          100    2.53    7.441
```
