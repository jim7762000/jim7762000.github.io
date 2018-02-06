---
layout: post
title: "RcppArmadillo call F77 blas/lapack"
---

前幾篇有提過在Rcpp, RcppEigen中call F77 blas/lapack

最近也大概抓到RcppArmadillo call F77的方法

因為Armadillo本身就有宣告BLAS跟LAPACK functions

其實只需要include Armadillo宣告的header file即可

要include的是這三個檔案：

``` cpp
#include <armadillo_bits/typedef_elem.hpp>
#include <armadillo_bits/def_blas.hpp>
#include <armadillo_bits/def_lapack.hpp>
```

不過我試了一下Armadillo並沒有import R全部的BLAS/LAPACK functions

所以遇到沒有的就要自己宣告，我這提供一個簡單例子宣告自己R有但Armadillo沒有的LAPACK function：

``` cpp
// [[Rcpp::depends(RcppArmadillo)]]
#include <RcppArmadillo.h>
#include <armadillo_bits/typedef_elem.hpp>
#include <armadillo_bits/def_blas.hpp>
#include <armadillo_bits/def_lapack.hpp>


#if !defined(ARMA_BLAS_CAPITALS)
#define arma_dposv dposv
#else
#define arma_dposv DPOSV
#endif

extern "C" {
  void arma_fortran(arma_dposv)(char* trans, blas_int* n, blas_int* nrhs, double* a, blas_int* lda, 
                    double*  b, blas_int* ldb, blas_int* info);
}

// [[Rcpp::export]]
arma::vec wlssolver(arma::mat X, arma::vec w, arma::vec y){
  blas_int info = 0, nrhs = 1, k = blas_int(X.n_cols);
  char uplo = 'L';
  arma::mat XWX = X.t() * (X.each_col() % w);
  arma::vec Xy = X.t() * (y % w);
  arma_fortran(arma_dposv)(&uplo, &k, &nrhs, XWX.memptr(), &k, Xy.memptr(), &k, &info);
  return(Xy);
}
```

上面這個script是用lapack中解正定對稱矩陣linear system的函數去求WLS的迴歸係數

簡單的R執行範例：

``` R
n <- 20
p <- 3
X <- matrix(rnorm(n * p), n , p)
beta <- rnorm(p)
w <- rgamma(nrow(X), 2, 0.5)
y <- 3 + X %*% beta + rnorm(n)
wlssolver(X, w, y)
```

