---
layout: post
title: Rcpp Attributes for Sparse Matrix
---

Thanks to the R package `Matrix`, we can use the sparse matrix in R. But to use the sparse matrix in the `Rcpp`, we need create a S4 object to get sparse matrix in `Rcpp`. However, `RcppArmadillo` provides a convenient API for interfacing the armadillo class `SpMat` and R class `dgCMatrix`. I provide several methods to use the sparse matrix in R.

Code:

``` R
library(Rcpp)
library(Matrix)
library(RcppArmadillo)
sourceCpp(code = '
// [[Rcpp::depends(Matrix, RcppArmadillo)]]
#include <RcppArmadillo.h>
using namespace Rcpp;
using namespace arma;

// [[Rcpp::export]]
mat SparseDenseMatrixMulti(sp_mat m, mat n) {
    mat R = m* n;
  return R;
}

// [[Rcpp::export]]
mat SparseDenseMatrixMulti2(sp_mat m, S4 n) {
  uvec dim(as<uvec>(n.slot("Dim")));
  NumericVector n_elem = n.slot("x");
  mat n_mat(n_elem.begin(), dim(0), dim(1), false);
    mat R = m * n_mat;
  return R;
}

// [[Rcpp::export]]
mat SparseDenseMatrixMulti3(sp_mat m, NumericMatrix n) {
    mat n_mat(n.begin(), n.nrow(), n.ncol(), false);
    mat R = m* n_mat;
  return R;
}

// [[Rcpp::export]]
mat SparseDenseMatrixMulti4(S4 m, S4 n) {
  uvec i(as<uvec>(m.slot("i"))),
       p(as<uvec>(m.slot("p"))),
     dim_m(as<uvec>(m.slot("Dim"))),
       dim_n(as<uvec>(n.slot("Dim")));
  NumericVector n_elem = n.slot("x");
  colvec m_elem = as<colvec>(m.slot("x"));
  sp_mat m_mat(i, p, m_elem, dim_m(0), dim_m(1));
  mat n_mat(n_elem.begin(), dim_n(0), dim_n(1), false);
    mat R = m_mat * n_mat;
  return R;
}')

i = c(1:196,3:800, 1005:2107, 5009:7011, 2001:8000)
j = c(2:200, 9:160, 11:9700, 9942:10000)
x = runif(length(i))
A = sparseMatrix(i, j, x = x)
B_S4 = Matrix(rnorm(3e7), 10000)
B = as.matrix(B_S4)

library(rbenchmark)
benchmark(CC1 = SparseDenseMatrixMulti(A, B),
        CC2 = SparseDenseMatrixMulti2(A, B_S4),
        CC3 = SparseDenseMatrixMulti3(A, B),
        CC4 = SparseDenseMatrixMulti4(A, B_S4),
        columns=c("test", "replications","elapsed", "relative"),
        replications=20, order="relative")
#   test replications elapsed relative
# 4  CC4           20   5.892    1.000
# 3  CC3           20   5.922    1.005
# 2  CC2           20   5.933    1.007
# 1  CC1           20   6.820    1.158
```

The above result show that if you use the interface of `RcppArmadillo` or `Rcpp`, then you need suffer from delay of memory copy.

My environment is ubuntu 14.04, R 3.1.1 compiled by intel c++, fortran compiler with MKL. My CPU is 3770K@4.3GHz.

