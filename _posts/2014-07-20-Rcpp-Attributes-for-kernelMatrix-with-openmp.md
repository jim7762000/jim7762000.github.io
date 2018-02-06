---
layout: post
title: Rcpp Attributes for kernelMatrix with openmp
---

I use Rcpp Attributes to compute the kernel matrix and show how to link omp to speedup your Rcpp code. I also present that the speed of Rcpp Attributes is competitive with the function kernelMatrix written in C in the R package kernlab.

Code:

``` R
library(kernlab)
library(Rcpp)
library(RcppArmadillo)
## For windows user
# library(inline)
# settings <- getPlugin("Rcpp")
# settings$env$PKG_CXXFLAGS <- paste('-fopenmp', settings$env$PKG_CXXFLAGS)
# settings$env$PKG_LIBS <- paste('-fopenmp -lgomp', settings$env$PKG_LIBS)
# do.call(Sys.setenv, settings$env)
sourceCpp(code = '
// [[Rcpp::depends(RcppArmadillo)]]
#include <RcppArmadillo.h>
#include <omp.h>
using namespace Rcpp;
using namespace arma;

// [[Rcpp::export]]
NumericMatrix kernelMatrix_cpp(NumericMatrix Xr, NumericMatrix Centerr, double sigma) {
  omp_set_num_threads(omp_get_max_threads());
  uword n = Xr.nrow(), b = Centerr.nrow(), row_index, col_index;
  mat X(Xr.begin(), n, Xr.ncol(), false);
  mat Center(Centerr.begin(), b, Centerr.ncol(), false);
  mat KerX(n, b);
  #pragma omp parallel private(row_index, col_index)
  for (row_index = 0; row_index < n; row_index++)
  {
    #pragma omp for nowait
    for (col_index = 0; col_index < b; col_index++)
    {
      KerX(row_index, col_index) = exp(sum(square(X.row(row_index)
        - Center.row(col_index))) / (-2.0 * sigma * sigma));
    }
  }
    return wrap(KerX);
}')
N = 1000
p = 100
b = 300
X = matrix(rnorm(N*p), ncol = p)
center = X[sample(1:N, b),]
sigma = 3
kernel_X <- kernelMatrix(rbfdot(sigma=1/(2*sigma^2)), X, center)
kernel_X_cpp = kernelMatrix_cpp(X, center, sigma)
## test
all.equal(kernel_X@.Data, kernel_X_cpp)
# TRUE

library(rbenchmark)
benchmark(cpp = kernelMatrix_cpp(X, center, sigma),
  kernlab = kernelMatrix(rbfdot(sigma=1/(2*sigma^2)), X, center),
  columns=c("test", "replications","elapsed", "relative"),
  replications=10, order="relative")
#      test replications elapsed relative
# 1     cpp           10   0.131    1.000
# 2 kernlab           10   0.199    1.519
```

With openmp, it is faster than kernlab with small input matrix size, however, the efficiency is too low when the size increase. Another efficient method is listed below:

``` R
library(kernlab)
library(Rcpp)
library(RcppArmadillo)
## For windows user
# library(inline)
# settings <- getPlugin("Rcpp")
# settings$env$PKG_CXXFLAGS <- paste('-fopenmp', settings$env$PKG_CXXFLAGS)
# settings$env$PKG_LIBS <- paste('-fopenmp -lgomp', settings$env$PKG_LIBS)
# do.call(Sys.setenv, settings$env)
sourceCpp(code = '
// [[Rcpp::depends(RcppArmadillo)]]
#include <RcppArmadillo.h>
using namespace Rcpp;
using namespace arma;

// [[Rcpp::export]]
NumericMatrix kernelMatrix_cpp2(NumericMatrix Xr, NumericMatrix Centerr, double sigma) {
  uword n = Xr.nrow(), b = Centerr.nrow(), row_index, col_index;
  mat X(Xr.begin(), n, Xr.ncol(), false),
      Center(Centerr.begin(), b, Centerr.ncol(), false),
      KerX(X*Center.t());
  colvec X_sq = sum(square(X), 1) / 2;
  rowvec Center_sq = (sum(square(Center), 1)).t() / 2;
  KerX.each_row() -= Center_sq;
  KerX.each_col() -= X_sq;
  KerX *= 1 / (sigma * sigma);
  KerX = exp(KerX);
  return wrap(KerX);
}')

N = 10000
p = 1000
b = 3000
X = matrix(rnorm(N*p), ncol = p)
center = X[sample(1:N, b),]
sigma = 3
t1 = Sys.time()
kernel_X_cpp = kernelMatrix_cpp2(X, center, sigma)
 Sys.time() - t1
 t1 = Sys.time()
kernel_X <- kernelMatrix(rbfdot(sigma=1/(2*sigma^2)), X, center)
Sys.time() - t1
## Test
all.equal(kernel_X@.Data, kernel_X_cpp)
# TRUE

library(rbenchmark)
benchmark(cpp = kernelMatrix_cpp(X, center, sigma),
          cpp2 = kernelMatrix_cpp2(X, center, sigma),
          kernlab = kernelMatrix(rbfdot(sigma=1/(2*sigma^2)), X, center),
          columns=c("test", "replications","elapsed", "relative"),
          replications=10, order="relative")
#      test replications elapsed relative
# 2    cpp2           10  13.810    1.000
# 3 kernlab           10  24.978    1.809
# 1     cpp           10 207.192   15.003

N = 1000
p = 100
b = 300
X = matrix(rnorm(N*p), ncol = p)
center = X[sample(1:N, b),]
sigma = 3
benchmark(cpp = kernelMatrix_cpp(X, center, sigma),
          cpp2 = kernelMatrix_cpp2(X, center, sigma),
          kernlab = kernelMatrix(rbfdot(sigma=1/(2*sigma^2)), X, center),
          columns=c("test", "replications","elapsed", "relative"),
          replications=10, order="relative")
#      test replications elapsed relative
# 2    cpp2           10   0.059    1.000
# 1     cpp           10   0.179    3.034
# 3 kernlab           10   0.230    3.898
```

The above result show that there is a trick you should know to speedup the code: if you can directly use matrix manipulation to finish the calculation, then please do it! Or you get low efficiency if you use for loop.

My environment is ubuntu 14.04, R 3.1.1 compiled by intel c++, fortran compiler with MKL. My CPU is 3770K@4.3GHz.

