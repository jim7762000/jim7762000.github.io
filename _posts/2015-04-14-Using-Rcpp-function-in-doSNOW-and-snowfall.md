---
layout: post
title: Using Rcpp function in doSNOW and snowfall
---

I had been searching how to use Rcpp function in snowfall or doSNOW for a long time, but there is still not a solution. I recently come up an idea to implement. Since the error is printed when exporting the Rcpp function to nodes, I compile Rcpp function in nodes. Surprisingly, I success.

``` R
library(data.table)
library(plyr)
library(dplyr)
library(magrittr)
library(foreach)
library(doSNOW)
library(snowfall)

library(Rcpp)
library(RcppArmadillo)
RcppCode = '
// [[Rcpp::depends(RcppArmadillo)]]
#include <RcppArmadillo.h>
using namespace Rcpp;
using namespace arma;

// [[Rcpp::export]]
List fastRidgeLM_Arma(NumericVector yr, NumericMatrix Xr, double lambda) {
  int n = Xr.nrow(), k = Xr.ncol();
  mat X(Xr.begin(), n, k, false);
  mat X2 = join_rows(ones<mat>(n, 1), X);
  colvec y(yr.begin(), n, false);
  colvec coef = solve(join_cols(X2, lambda*eye<mat>(k,k+1)),
    join_cols(y, zeros<colvec>(k)));
  colvec resid = y - X2*coef;
  double sig2 = sum(square(resid)) / (n-k);
  colvec stderrest = sqrt(sig2 * diagvec(inv((trans(X2) * X2))));
  return List::create(Named("coefficients") = coef,
                      Named("stderr") = stderrest);
}'

sourceCpp(code = RcppCode)

n = 10000
p = 1000
X = matrix(rnorm(n*p), n)
tmp_B = matrix(0, p, p)
tmp_B[sample(1:p, 2*p, replace = TRUE), sample(1:p, 2*p, replace = TRUE)] = runif(2*p)*2-1
diag(tmp_B) = 1
X = X %*% tmp_B
Y = X %*% sample(seq(-5, 5, length=p*3), p) + rnorm(n, 0, 10)
X.N = scale(X)
Y.N = scale(Y)
lambda = seq(0, 1, length = 51)

# Example by selecting regularizer in the ridge regression
cv_index_f = function(n, fold = 10){
  fold_n = floor(n / fold)
  rem = n - fold_n * fold
  size = rep(fold_n, fold)
  if(rem > 0)
    size[1:rem] = fold_n + 1
  cv_index = 1:fold %>% sapply(function(i) rep(i, size[i])) %>%
    sample(length(.))
  return(cv_index)
}

sfInit(TRUE, 2)
sfExport("Y.N", "X.N", "lambda", "RcppCode")
sfLibrary("Rcpp", character.only=TRUE)
sfLibrary("RcppArmadillo", character.only=TRUE)
sfClusterEval(sourceCpp(code = RcppCode))

cl = makeCluster(2, type = "SOCK")
registerDoSNOW(cl)
clusterExport(cl, c("Y.N", "X.N", "lambda", "RcppCode"))
clusterEvalQ(cl, library(Rcpp))
clusterEvalQ(cl, library(RcppArmadillo))
clusterEvalQ(cl, sourceCpp(code = RcppCode))

CV_f = function(X, Y, lambda, n_fold, parallel, parPackage = "foreach"){
  cvIndex = cv_index_f(nrow(X), n_fold)
  if ((parPackage == "foreach" && parallel) || !parallel)
  {
    if (parallel)
      clusterExport(cl, "cvIndex", environment())
    CVScores = aaply(1:n_fold, 1, function(i){
      aaply(lambda, 1, function(l){
        tmp = fastRidgeLM_Arma(Y.N[cvIndex != i], X.N[cvIndex != i,], l)
        mean((Y.N[cvIndex == i] - cbind(1, X.N[cvIndex == i,]) %*%
          tmp$coefficients)^2)
      }, .parallel = parallel)
    })
    if (parallel)
      clusterEvalQ(cl, rm(cvIndex))
  }
  else if(parPackage == "snowfall")
  {
    if (parallel)
      sfExport("cvIndex")
    CVScores = aaply(1:n_fold, 1, function(i){
      sfSapply(lambda, function(l){
        tmp = fastRidgeLM_Arma(Y.N[cvIndex != i], X.N[cvIndex != i,], l)
        mean((Y.N[cvIndex == i] - cbind(1, X.N[cvIndex == i,]) %*%
          tmp$coefficients)^2)
      })
    })
    if (parallel)
      sfRemove("cvIndex")
  }
  aaply(CVScores, 2, mean)
}
library(rbenchmark)
benchmark(
  CV_f(X, Y, lambda, 10, FALSE)            ,
  CV_f(X, Y, lambda, 10, TRUE, "foreach")  ,
  CV_f(X, Y, lambda, 10, TRUE, "snowfall") ,
  columns = c("test", "replications", "user.self", "elapsed"),
  replications = 10, order = "relative")
stopCluster(cl)
sfStop()

```
