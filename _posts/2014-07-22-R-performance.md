---
layout: post
title: R Performance for calculating the maximum of two normal random variable
---

There is a performance test between R functions, R functions with byte compiler, Rcpp and RcppArmadillo. It is about the generating the normal random variables and the vectorized computations.

Code:

``` R
# 1
a = function(){
  sum = 0
  nreps = 1e7
  for(i in 1:nreps){
    xy = rnorm(2)
    sum = sum + max(xy)
  }
  print(sum/nreps)
}

# 2
b = function(){
  nreps = 1e7
  xymat = matrix(rnorm(2*nreps),nreps)
  maxs = pmax(xymat[,1],xymat[,2])
  print(mean(maxs))
}

# 3
d = function(){
  nreps = 1e7
  print(mean(sapply(1:nreps, function(i) max(rnorm(2)))))
}

# 4
e = function(){
  nreps = 1e7
  xymat = matrix(rnorm(2*nreps),nreps)
  maxs = apply(xymat, 1, max)
  print(mean(maxs))
}

# 5
f = function(){
  nreps = 1e7
  print(Reduce('+', lapply(1:nreps, function(i) max(rnorm(2)))) / nreps)
}

# 6
g = function(){
  nreps = 1e7
  library(snowfall)
  sfInit(TRUE, 8)
  sfExport('nreps')
  maxs = sfLapply(1:nreps, function(i) max(rnorm(2)))
  sfStop()
  print(Reduce('+', maxs) / nreps)
}

# cmpfun
library(compiler)
a_cmp = cmpfun(a)
b_cmp = cmpfun(b)
d_cmp = cmpfun(d)
e_cmp = cmpfun(e)
f_cmp = cmpfun(f)

# Rcpp
library(Rcpp)
## For windows user
# library(inline)
# settings <- getPlugin("Rcpp")
# settings$env$PKG_CXXFLAGS <- paste('-fopenmp', settings$env$PKG_CXXFLAGS)
# settings$env$PKG_LIBS <- paste('-fopenmp -lgomp', settings$env$PKG_LIBS)
# do.call(Sys.setenv, settings$env)
sourceCpp(code = '
#include <Rcpp.h>
#include <omp.h>
using namespace Rcpp;
// [[Rcpp::export]]
double mm(NumericVector x, NumericVector y) {
  int n = x.size();
  double sum_maxs = 0;
  for(int i = 0; i < n; ++i) {
    if(x[i] > y[i]) sum_maxs += x[i];
    else sum_maxs += y[i];
  }
  return sum_maxs / n;
}
// [[Rcpp::export]]
double mm2(NumericMatrix x) {
  int n = x.nrow();
  double sum_maxs = 0;
  for(int i = 0; i < n; ++i) {
    if(x[i] > x[i+n])
      sum_maxs += x[i];
    else
      sum_maxs += x[i+n];
  }
  return sum_maxs / n;
}
// [[Rcpp::export]]
double mm3(NumericVector x, NumericVector y) {
  omp_set_num_threads(omp_get_max_threads());
  int n = x.size();
  double sum_maxs = 0;
  #pragma omp parallel
  {
    #pragma omp for reduction( +:sum_maxs)
    for(int i = 0; i < n; ++i) {
      if(x[i] > y[i])
        sum_maxs += x[i];
      else
        sum_maxs += y[i];
    }
  }
  return sum_maxs / n;
}
// [[Rcpp::export]]
double mm4(NumericMatrix x) {
  omp_set_num_threads(omp_get_max_threads());
  int n = x.nrow();
  double sum_maxs = 0;
  #pragma omp parallel
  {
    #pragma omp for reduction( +:sum_maxs)
    for(int i = 0; i < n; ++i)
    {
      if(x[i] > x[i+n])
        sum_maxs += x[i];
      else
        sum_maxs += x[i+n];
    }
  }
  return sum_maxs / n;
}
// [[Rcpp::export]]
double mm5(int n) {
  omp_set_num_threads(omp_get_max_threads());
  Rcpp::NumericVector x(n);
  Rcpp::NumericVector y(n);
  RNGScope scope;
  x = rnorm(n, 0.0, 1.0);
  y = rnorm(n, 0.0, 1.0);
  double sum_maxs = 0;
  #pragma omp parallel
  {
    #pragma omp for reduction( +:sum_maxs)
    for(int i = 0; i < n; ++i) {
      if(x[i] > y[i]) sum_maxs += x[i];
      else sum_maxs += y[i];
    }
  }
  return sum_maxs / n;
}')
h = function(){
  nreps = 1e7
  x = rnorm(nreps)
  y = rnorm(nreps)
  print(mm(x, y))
}
i = function(){
  nreps = 1e7
  x = matrix(rnorm(2*nreps),nreps)
  print(mm2(x))
}
j = function(){
  nreps = 1e7
  x = rnorm(nreps)
  y = rnorm(nreps)
  print(mm3(x, y))
}
k = function(){
  nreps = 1e7
  x = matrix(rnorm(2*nreps),nreps)
  print(mm4(x))
}
l = function(){
  nreps = 1e7
  print(mm5(nreps))
}

library(RcppArmadillo)
sourceCpp(code = '
// [[Rcpp::depends(RcppArmadillo)]]
#include <RcppArmadillo.h>
#include <Rcpp.h>
using namespace Rcpp;
using namespace arma;
// [[Rcpp::export]]
double mm6(int n) {
  arma_rng::set_seed_random();
  colvec x = randn(n), y = randn(n);
  return as_scalar(mean(arma::max(x,y)));
}
// [[Rcpp::export]]
double mm7(int n) {
  arma_rng::set_seed_random();
  mat x = randn(n, 2);
  return as_scalar(mean(arma::max(x,1)));
}')
m = function(){
  nreps = 1e7
  print(mm6(nreps))
}
n = function(){
  nreps = 1e7
  print(mm7(nreps))
}
library(rbenchmark)
benchmark(a(),b(),d(),e(),f(),g(),h(),i(),j(),k(),l(),m(),n(), a_cmp(),b_cmp(),d_cmp(),e_cmp(),f_cmp(), replications = 10, columns=c('test', 'replications', 'elapsed','relative', 'user.self'), order='relative')

## linux
#       test replications elapsed relative user.self
# 12     m()           10   6.909    1.000     8.790
# 13     n()           10   7.161    1.036     7.091
# 11     l()           10  11.123    1.610    29.872
# 7      h()           10  17.842    2.582    17.826
# 9      j()           10  17.873    2.587    34.783
# 10     k()           10  20.029    2.899    37.757
# 8      i()           10  20.925    3.029    20.811
# 15 b_cmp()           10  24.484    3.544    24.282
# 2      b()           10  24.514    3.548    24.290
# 6      g()           10 242.593   35.113   114.829
# 4      e()           10 362.433   52.458   361.964
# 17 e_cmp()           10 370.546   53.632   370.284
# 14 a_cmp()           10 381.048   55.152   381.387
# 1      a()           10 434.574   62.900   434.929
# 5      f()           10 579.186   83.831   579.427
# 18 f_cmp()           10 621.092   89.896   621.314
# 3      d()           10 634.046   91.771   634.205
# 16 d_cmp()           10 686.426   99.352   686.650
```

Number 1, 2, 3, 4, 5 and 6 are the R functions. Number 14, 15, 16, 17 and 18 are R functions with byte compiler. Number 7, 8, 9, 10 and 11 are the Rcpp functions. Number 12 and 13 are RcppArmadillo functions.

Number 1 function is using for-loop in R. Number 2 function is using the function pmax (R vectorized maximum function). Number 3, 4 and 5 functions are the application of sapply, apply and lapply function, respectively. Number 6 function is using the parallel computing package snowfall in R.

Number 7 and 8 function is to locate the two vectors of normal r.v. and to use Rcpp function to calculate. The difference between two functions are the input element, number 7 function input two vectors and number 8 function input one matrix.

Number 9 and 10 functions is the parallel version of number 7 and 8 functions, respectively. Number 11 function is to use the c++ for locate the two vectors of normal r.v. and to calculate in the way of number 9 function. Number 12 and 13 functions are to use armadillo for locating the two vectors of normal r.v. and calculating with vector or matrix manipulation.

The results shows that generating normal r.v. in c++ is fast than generating in R, the vectorized function pmax is fast than other ?apply functions and calculation of two vectors are fast than a matrix (I think that the reason is searching address.).

My environment is ubuntu 14.04, R 3.1.1 compiled by intel c++, fortran compiler with MKL. My CPU is 3770K@4.3GHz.

Remark:
I think that the reason why the calculation with matrix is slow is that c++ is a row-major language, so I try another example:

``` R
library(Rcpp)
library(RcppArmadillo)
sourceCpp(code = '
// [[Rcpp::depends(RcppArmadillo)]]
#include <RcppArmadillo.h>
#include <Rcpp.h>
using namespace Rcpp;
using namespace arma;

// [[Rcpp::export]]
double mm6(int n) {
arma_rng::set_seed_random();
colvec x = randn(n), y = randn(n);
return as_scalar(mean(arma::max(x,y)));
}
// [[Rcpp::export]]
double mm7(int n) {
arma_rng::set_seed_random();
mat x = randn(n, 2);
return as_scalar(mean(arma::max(x,1)));
}
// [[Rcpp::export]]
double mm9(int n) {
arma_rng::set_seed_random();
mat x = randn(2, n);
return as_scalar(mean(arma::max(x,0), 1));
}')
m = function(){
  nreps = 1e7
  print(mm6(nreps))
}
n = function(){
  nreps = 1e7
  print(mm7(nreps))
}
o = function(){
  nreps = 1e7
  print(mm9(nreps))
}
library(rbenchmark)
benchmark(m(),n(),o(), replications = 10, columns=c('test', 'replications', 'elapsed','relative', 'user.self'), order='relative')
#   test replications elapsed relative user.self
# 1  m()           10   7.001    1.000     6.875
# 3  o()           10   7.090    1.013     6.977
# 2  n()           10   7.247    1.035     7.124
```

The results show that calculation in row and calculation in column is big difference. My environment is ubuntu 14.04, R 3.1.1 compiled by intel c++, fortran compiler with MKL. My CPU is 3770K@4.3GHz.

