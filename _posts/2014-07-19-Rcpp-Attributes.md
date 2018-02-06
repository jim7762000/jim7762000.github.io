---
layout: post
title: Rcpp Attributes
---

Recently, I went to the 23th STSC, I got some information about the new API of `Rcpp`, `Rcpp attributes`. I had tried some examples and it worked well. Here I demonstrate some examples.

First example: call the `pnorm` function in Rcpp:

``` R
require(Rcpp)
sourceCpp(code = '
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
DataFrame mypnorm(NumericVector x){
  int n = x.size();
  NumericVector y1(n), y2(n), y3(n);
  for (int i=0; i<n; i++){
    y1[i] = ::Rf_pnorm5(x[i], 0.0, 1.0, 1, 0);
    y2[i] = R::pnorm(x[i], 0.0, 1.0, 1, 0);
  }
  y3 = pnorm(x);
  return DataFrame::create(
    Named("R") = y1,
    Named("Rf_") = y2,
    Named("sugar") = y3);
}')
mypnorm(runif(10, -3, 3))
```

Rcpp attributes allows user to write Rcpp in a simple way. User does not need to learn about how to write R extension. Just write a cpp script and add the line `// [[Rcpp::export]]`, then user can use the function in R.

Next two example is about the two extension packages of Rcpp, RcppArmadillo and RcppEigen. The two packages provide Rcpp to link the C++ linear algebra libraries, armadillo and Eigen.

``` R
library(Rcpp)
library(RcppArmadillo)
library(RcppEigen)
sourceCpp(code = '
// [[Rcpp::depends(RcppArmadillo)]]
#include <RcppArmadillo.h>
using namespace Rcpp;

// [[Rcpp::export]]
List fastLm_RcppArma(NumericVector yr, NumericMatrix Xr) {
  int n = Xr.nrow(), k = Xr.ncol();
  arma::mat X(Xr.begin(), n, k, false);
  arma::colvec y(yr.begin(), yr.size(), false);
  arma::colvec coef = arma::solve(X, y);
  arma::colvec resid = y - X*coef;
  double sig2 = arma::as_scalar(arma::trans(resid)*resid/(n-k));
  arma::colvec stderrest = arma::sqrt(sig2 * arma::diagvec( arma::inv(arma::trans(X)*X)));
  return List::create(
    Named("coefficients") = coef,
    Named("stderr") = stderrest);
}')

sourceCpp(code = '
// [[Rcpp::depends(RcppEigen)]]
#include <RcppEigen.h>
using namespace Rcpp;
using Eigen::Map;
using Eigen::MatrixXd;
using Eigen::VectorXd;

// [[Rcpp::export]]
List fastLm_RcppEigen(NumericVector yr, NumericMatrix Xr) {
  const Map<MatrixXd> X(as<Map<MatrixXd> >(Xr));
  const Map<VectorXd> y(as<Map<VectorXd> >(yr));
  int n = Xr.nrow(), k = Xr.ncol();
  VectorXd coef = (X.transpose() * X).llt().solve(X.transpose() * y.col(0));
  VectorXd resid = y - X*coef;
  double sig2 = resid.squaredNorm() / (n - k);
  VectorXd stderrest = (sig2 * ((X.transpose() * X).inverse()).diagonal()).array().sqrt();
  return List::create(
    Named("coefficients") = coef,
    Named("stderr") = stderrest);
}')
N = 20000
p = 1000
X = matrix(rnorm(N*p), ncol = p)
y = X %*% 10**(sample(seq(-5, 3, length = N+p), p)) + rnorm(N)

system.time(fastLm_RcppArma(y, X))
#   user  system elapsed
#   5.49    0.06    1.46
system.time(fastLm_RcppEigen(y, X))
#   user  system elapsed
#   9.32    0.06    8.96
system.time(lm(y~X - 1))
#   user  system elapsed
# 145.13   14.76   91.84
```

The cpp functions are faster 63 times than R function `lm`. My environment is ubuntu 14.04, R 3.1.1 compiled by intel c++, fortran compiler with MKL. My CPU is 3770K@4.3GHz. I think that Rcpp is the package which is the worthiest to learn if you want to use R to do statistical computing or machine learning. Rcpp attributes had changed the way to source C++ code in R, it let `Rcpp` is more convenient and more powerful.


