---
layout: post
title: Rcpp Attributes with lasso
---

I try to write a lasso algorithm by Rcpp attributes.

Reference:
Friedman, J., Hastie, T. and Tibshirani, R. (2008) Regularization Paths for Generalized Linear Models via Coordinate Descent, [http://www.stanford.edu/~hastie/Papers/glmnet.pdf](http://www.stanford.edu/~hastie/Papers/glmnet.pdf), Journal of Statistical Software, Vol. 33(1), 1-22 Feb 2010, [http://www.jstatsoft.org/v33/i01/](http://www.jstatsoft.org/v33/i01/)

Code:

``` R
library(Rcpp)
library(RcppArmadillo)
sourceCpp(code = '
// [[Rcpp::depends(RcppArmadillo)]]
#include <RcppArmadillo.h>
#include <ctime>
using namespace Rcpp;
using namespace arma;
// [[Rcpp::export]]
List lasso_fit_f(NumericMatrix Xr, NumericVector yr, int n_penalty = 100){
  int n = Xr.nrow(), p = Xr.ncol();
  mat X(Xr.begin(), n, p, false);
  colvec y(yr.begin(), n, false);
  double z0 = mean(y), penalty;
  colvec xydot = (y.t() * X).t();
  mat xxdot(p, p);
  xxdot = X.t() * X;
  double penalty_max_log = log(max(xydot) / n);
  colvec penalties = exp(linspace<colvec>(penalty_max_log, penalty_max_log+log(0.05), n_penalty));
  mat coef_m(p, n_penalty);
  int MaxIter = 1e5;
  colvec coef(p), coef_new(p);
  for(int k = 1; k < n_penalty; k++)
  {
    coef = coef_m.col(k-1);
    penalty = penalties(k);
    for(int i = 0; i < MaxIter; i++)
    {
      coef_new = (xydot - xxdot * coef) / n + coef;
      coef_new(find(abs(coef_new) < penalty)).zeros();
      coef_new(find(coef_new > penalty)) -= penalty;
      coef_new(find(coef_new < -penalty)) += penalty;
      if( as_scalar((coef - coef_new).t() * (coef - coef_new)) / k < 1e-7)
        break;
      coef = coef_new;
    }
    coef_m.col(k) = coef;
  }
  return List::create(Named("intercept") = z0,
                      Named("coefficients") = coef_m,
                      Named("penalties") = penalties);
}')

d = 1000; N = 10000
x = matrix(rnorm(N*d), N)
x[,3] = x[,1] - 2*x[,2] + rnorm(N)
x[,30] = x[,10] - 2*x[,20] + rnorm(N)
y = cbind(1, x) %*% c(-1, 2, -0.3, 0.7, sample(10**seq(-10, 1, length = N), d-3)) + rnorm(N, 0, 2)
x = scale(x)

t1 = Sys.time()
a = lasso_fit_f(x, y)
t_cpp = Sys.time() - t1

library(glmnet)
t1 = Sys.time()
fit = glmnet(x,y, lambda = a$penalties)
t_glmnet = Sys.time() - t1
c(t_glmnet, t_cpp)
# [1] 0.7171087 0.2032089
```

It works well!!

My environment is ubuntu 14.04, R 3.1.1 compiled by intel c++, fortran compiler with MKL. My CPU is 3770K@4.3GHz.

