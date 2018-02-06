---
layout: post
title: "Rcpp link system library"
---

這篇主要是介紹Linux環境下，如何直接在Rcpp裡面去link系統已有的library

這篇主要是因為我在測試Blaze為什麼效能遠不如直接跑blazemark的結果

所以就想說直接link系統裡面的armadillo, blaze試試看

我這裡讓Rcpp直接使用我系統安裝好的Intel MKL以及Armaillo當第一個例子

``` R
# armadillo
Sys.setenv(PKG_CPPFLAGS = paste("-I/opt/intel/compilers_and_libraries_2017.1.132/linux/mkl/include -I/usr/include",
                                "-L/opt/intel/compilers_and_libraries_2017.1.132/linux/mkl/lib/intel64 -L/usr/lib64",
                                "-lmkl_intel_lp64 -lmkl_intel_thread -lmkl_core  -liomp5 -lpthread -lm -larmadillo",
                                "-O3 -mavx -DNDEBUG -fpermissive"),
           PKG_LIBS = paste("-L/usr/lib64 -larmadillo"))

sourceCpp(code = "
#include <Rcpp.h>
#include <mkl_blas.h>
#include <armadillo>

// [[Rcpp::export]]
Rcpp::NumericMatrix arma_XTX2(Rcpp::NumericMatrix Xs) {
  int nr = Xs.nrow(), nc = Xs.ncol(), i_col, i_row, k;
  arma::mat X(Xs.begin(), nr, nc, false);
  arma::mat XTX = X.t() * X;
  Rcpp::NumericMatrix out(nc, nc);
  for( i_col=0, k=0 ; i_col < nc; ++i_col){
  	for( i_row = 0; i_row < nc ; ++i_row, ++k ){
  		out[k] = XTX(i_row,i_col) ;
  	}
  }
  return out;
}", verbose = TRUE, rebuild = TRUE)
```

下面是Rcpp link Blaze and MKL的例子

``` R
library(Rcpp)
Sys.setenv(PKG_CPPFLAGS = paste("-I/opt/intel/compilers_and_libraries_2017.1.132/linux/mkl/include -I/usr/local/include/blaze",
                                "-L/opt/intel/compilers_and_libraries_2017.1.132/linux/mkl/lib/intel64",
                                "-lmkl_intel_lp64 -lmkl_intel_thread -lmkl_core  -liomp5 -lpthread -lm",
                                "-Wall -Wextra -Werror -Woverloaded-virtual -ansi -O3 -mavx -DNDEBUG -fpermissive -std=c++14"))
sourceCpp(code = "
#include <mkl_cblas.h>
#include <Blaze.h>
#include <Rcpp.h>

// [[Rcpp::export]]
Rcpp::NumericMatrix blaze3_XTX3(Rcpp::NumericMatrix Xs) {
  int nr = Xs.nrow(), nc = Xs.ncol(), i_col, i_row, k;
  blaze::DynamicMatrix<double, blaze::columnMajor> X(nr, nc);
  for( i_col=0; i_col < nc; ++i_col){
  	for( i_row = 0; i_row < nc ; ++i_row ){
          X(i_row, i_col) = Xs(i_row,i_col) ;
    }
  }


  blaze::DynamicMatrix<double, blaze::rowMajor> XT = blaze::trans(X);
  blaze::DynamicMatrix<double, blaze::columnMajor> XTX( XT * X );
  Rcpp::NumericMatrix out(nc, nc);
  for( i_col=0, k=0 ; i_col < nc; ++i_col){
  	for( i_row = 0; i_row < nc ; ++i_row, ++k ){
  		out[k] = XTX(i_row,i_col) ;
  	}
  }
  return out;
}", verbose = TRUE, rebuild = TRUE)
```




