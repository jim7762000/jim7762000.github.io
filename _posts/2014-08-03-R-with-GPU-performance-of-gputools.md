---
layout: post
title: R with GPU performance of gputools
---

I can't wait to test the performance of R with GPU. I test the performance of gputools and OpenCL.

The test code is showed in following:

``` R
library(Rcpp)
sourceCpp(code = '
// [[Rcpp::depends(RcppArmadillo)]]
#define ARMA_USE_BLAS
#define ARMA_USE_ARPACK
#define ARMA_USE_MKL_ALLOC
#include <RcppArmadillo.h>
using namespace Rcpp;
using namespace arma;

// [[Rcpp::export]]
List fastLm_RcppArma_cpp_f(NumericVector yr, S4 Xr) {
  uvec dim(as<uvec>(Xr.slot("Dim")));
  NumericVector Xr_elem = Xr.slot("x");
  mat X(Xr_elem.begin(), dim(0), dim(1), false);
  colvec y(yr.begin(), yr.size(), false);
  colvec coef = solve(X, y);
  colvec resid = y - X*coef;
  double sig2 = as_scalar(resid.t()*resid/(dim(0) - dim(1)));
  colvec stderrest = sqrt(sig2 * diagvec( inv(X.t()*X)));
  return List::create(Named("coefficients") = coef,
              Named("stderr") = stderrest);
}')

sourceCpp(code = '
// [[Rcpp::depends(RcppEigen)]]
#define EIGEN_USE_MKL_ALL
#include <RcppEigen.h>
using namespace Rcpp;
using Eigen::Map;
using Eigen::MatrixXd;
using Eigen::VectorXd;

// [[Rcpp::export]]
List fastLm_RcppEigen_cpp_f(NumericVector yr, NumericMatrix Xr) {
  const Map<MatrixXd> X(as<Map<MatrixXd> >(Xr));
  const Map<VectorXd> y(as<Map<VectorXd> >(yr));
  int n = Xr.nrow(), k = Xr.ncol();
  VectorXd coef = (X.transpose() * X).llt().solve(X.transpose() * y.col(0));
  VectorXd resid = y - X*coef;
  double sig2 = resid.squaredNorm() / (n - k);
  VectorXd stderrest = (sig2 * ((X.transpose() * X).inverse()).diagonal()).array().sqrt();
  return List::create(Named("coefficients") = coef,
            Named("stderr") = stderrest);
}')

library(Matrix)
library(MatrixModels)

fastLm_RcppArma = function(formula, data){
  inputs = model.Matrix(formula, data, sparse = FALSE)
  output = data[,as.character(formula[[2]]), with = FALSE][[1]]
  return(fastLm_RcppArma_cpp_f(output, inputs))
}

fastLm_RcppEigen = function(formula, data){
  inputs = model.matrix(formula, data)
  output = data[,as.character(formula[[2]]), with = FALSE][[1]]
  return(fastLm_RcppEigen_cpp_f(output, inputs))
}

library(data.table)
library(dplyr)
N = 25000
p = 1200
X = as.matrix(Matrix(rnorm(N*p), ncol = p))
Beta = (10**(sample(seq(-5, 1, length = N+p), p)))
y = X %*% Beta + rnorm(N)
dat = data.table(X)
set(dat, j = "y", value = y)

library(gputools)
library(rbenchmark)
benchmark(result_RcppArma = fastLm_RcppArma(y ~ ., data = dat),
        result_RcppEigen = fastLm_RcppEigen(y ~ ., data = dat),
        result_gpuLM = gpuLm(y ~ ., data = dat),
        result_Rlm = lm(y~., data = dat),
        columns=c("test", "replications","elapsed", "relative"),
        replications=10, order="relative")
#               test replications elapsed relative
# 3     result_gpuLM           10  35.137    1.000
# 2 result_RcppEigen           10  37.511    1.068
# 1  result_RcppArma           10  40.170    1.143
# 4       result_Rlm           10 146.959    4.182

sourceCpp(code = '
// [[Rcpp::depends(RcppArmadillo)]]
#define ARMA_USE_BLAS
#define ARMA_USE_ARPACK
#define ARMA_USE_MKL_ALLOC
#include <RcppArmadillo.h>
using namespace Rcpp;
using namespace arma;

// [[Rcpp::export]]
NumericMatrix RcppMatMult_Arma(NumericMatrix Xr, NumericMatrix Yr) {
  int m = Xr.nrow(), n = Xr.ncol(), k = Yr.ncol();
  if(n != Yr.ncol()) exit(1);
  mat X(Xr.begin(), m, n, false);
  mat Y(Xr.begin(), n, k, false);
  return wrap(X*Y);
}')

sourceCpp(code = '
// [[Rcpp::depends(RcppEigen)]]
#define EIGEN_USE_MKL_ALL
#include <RcppEigen.h>
using namespace Rcpp;
using Eigen::Map;
using Eigen::MatrixXd;
using Eigen::VectorXd;

// [[Rcpp::export]]
List RcppMatMult_Eigen(NumericMatrix Xr, NumericMatrix Yr) {
  const Map<MatrixXd> X(as<Map<MatrixXd> >(Xr));
  const Map<MatrixXd> Y(as<Map<MatrixXd> >(Yr));
  return wrap(X*Y);
}')

m = 10000
n = 5000
k = 10000
matA = matrix(runif(m*n), m)
matB = matrix(runif(n*k), n)
s = proc.time()
a = matA %*% matB
proc.time() - s
#   user  system elapsed
# 57.513   0.330  15.525
s = proc.time()
a = cpuMatMult(matA, matB) # same as matA %*% matB
proc.time() - s
#   user  system elapsed
# 56.287   0.252  14.573
s = proc.time()
a = gpuMatMult(matA, matB)
proc.time() - s
#   user  system elapsed
#  6.587   2.381   8.958
s = proc.time()
a = RcppMatMult_Arma(matA, matB)
proc.time() - s
# same BLAS, so the result is the same.
#   user  system elapsed
# 56.763   0.308  14.876
s = proc.time()
a = RcppMatMult_Eigen(matA, matB)
proc.time() - s
# same BLAS, so the result is the same.
#    user  system elapsed
# 123.486   1.766  88.519
```

R with GPU is 2 times faster than R with CPU!! My environment is ubuntu 14.04. My CPU is 3770K@4.3GHz and GPU is GTX 670. If you have some questions, you can reference following urls:



