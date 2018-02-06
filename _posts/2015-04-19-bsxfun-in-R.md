---
layout: post
title: bsxfun in R -- matrix-vector arithmetic operation
---

The matrix-vector arithmetic operation in R is difficult to do before. We need for-loop or `replicate` to compute. Now, we have `sweep` to do. There are several examples to demonstrate.

``` R
M = 3
N = 5
(mat = replicate(N, 1:M))
#      [,1] [,2] [,3] [,4] [,5]
# [1,]    1    1    1    1    1
# [2,]    2    2    2    2    2
# [3,]    3    3    3    3    3
(vec = 1:N)
# [1] 1 2 3 4 5
(sweep(mat, MARGIN=2, vec,`+`))
#      [,1] [,2] [,3] [,4] [,5]
# [1,]    2    3    4    5    6
# [2,]    3    4    5    6    7
# [3,]    4    5    6    7    8
(sweep(mat, MARGIN=2, vec,`*`))
#      [,1] [,2] [,3] [,4] [,5]
# [1,]    1    2    3    4    5
# [2,]    2    4    6    8   10
# [3,]    3    6    9   12   15
(sweep(mat, MARGIN=2, vec,`/`))
#      [,1] [,2]      [,3] [,4] [,5]
# [1,]    1  0.5 0.3333333 0.25  0.2
# [2,]    2  1.0 0.6666667 0.50  0.4
# [3,]    3  1.5 1.0000000 0.75  0.6
(sweep(mat, MARGIN=2, vec,`-`))
#      [,1] [,2] [,3] [,4] [,5]
# [1,]    0   -1   -2   -3   -4
# [2,]    1    0   -1   -2   -3
# [3,]    2    1    0   -1   -2
(sweep(mat, MARGIN=2, vec, pmax))
#      [,1] [,2] [,3] [,4] [,5]
# [1,]    1    2    3    4    5
# [2,]    2    2    3    4    5
# [3,]    3    3    3    4    5
(sweep(mat, MARGIN=2, vec, pmin))
#      [,1] [,2] [,3] [,4] [,5]
# [1,]    1    1    1    1    1
# [2,]    1    2    2    2    2
# [3,]    1    2    3    3    3

## usage in scaling
N = 10; p = 3
dat = matrix(rnorm(N*p), N)

dat_standardized = sweep(sweep(dat, 2, colMeans(dat)), 2,
  apply(dat, 2, sd), '/')

dat_normalized = sweep(sweep(dat, 2, apply(dat, 2, min)), 2,
  apply(dat, 2, function(x) diff(range(x))), '/')

## usage in computing kernel matrix
kernelMatrix_f = function(X, center, sigma){
  exp(sweep(sweep(X %*% t(center), 1,
    rowSums(X**2)/2), 2, rowSums(center**2)/2) / (sigma**2))
}

## compare with Rcpp and kernlab
library(kernlab)
library(Rcpp)
library(RcppArmadillo)
sourceCpp(code = '
// [[Rcpp::depends(RcppArmadillo)]]
#include <RcppArmadillo.h>
using namespace Rcpp;
using namespace arma;

// [[Rcpp::export]]
NumericMatrix kernelMatrix_cpp(NumericMatrix Xr, NumericMatrix Centerr, double sigma) {
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

N = 8000
p = 500
b = 1000
X = matrix(rnorm(N*p), ncol = p)
center = X[sample(1:N, b),]
sigma = 3

all.equal(kernelMatrix(rbfdot(sigma=1/(2*sigma^2)), X, center)@.Data,
  kernelMatrix_cpp(X, center, sigma))
# TRUE
all.equal(kernelMatrix(rbfdot(sigma=1/(2*sigma^2)), X, center)@.Data,
  kernelMatrix_f(X, center, sigma))
# TRUE

library(microbenchmark)
microbenchmark(Rfun = kernelMatrix_f(X, center, sigma),
  kernlab = kernelMatrix(rbfdot(sigma=1/(2*sigma^2)), X, center),
  Rcpp = kernelMatrix_cpp(X, center, sigma),
  times = 20L
)
# Unit: milliseconds
#     expr      min       lq      mean    median        uq       max neval
#     Rfun 872.8854 981.1995 1036.2209 1074.0526 1098.9185 1132.9356    20
#  kernlab 773.7489 841.9028 1059.3098  862.7476  883.3874 2979.2541    20
#     Rcpp 490.2462 501.2993  520.1283  522.5060  532.5426  571.0949    20
```
