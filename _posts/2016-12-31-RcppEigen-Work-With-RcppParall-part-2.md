---
layout: post
title: "RcppEigen Work with RcppParallel (Part 2)"
---

我在把RcppEigen跟RcppParallel合體的時候

看了下面這篇的程式：

[C++ code](https://gist.github.com/JWiley/d9cba55603471f75d438)，[R code](https://gist.github.com/JWiley/8fe93ae105b1fb244f93)

我也有發現他在R mail群裡面回答說這個程式時不時會crash

我就想到我第一部分提到的Rcpp::NumericVector會在RcppParallel會crash的問題

那我就修改了一下，發現程式就穩定了，我也把我修改後的程式放上來 (稍微改的漂亮一些)

R code:

``` R
Rcpp::sourceCpp("parallel_boot_lm.cpp")  # could be found at http://pastebin.com/Xir4LYei
 
set.seed(1234)
N <- 1e3L
p <- 50L
R <- 500L
dmat <- cbind(1, matrix(rnorm(N * p), ncol = p))
beta <- c(2, runif(p))
yvec <- as.vector(dmat %*% beta + rnorm(N, sd = 3))
dall <- cbind(y = yvec, as.data.frame(dmat[,-1]))
myindex <- matrix(sample(0:(N - 1), N * R, TRUE), ncol = R)
 
system.time(res1 <- parallelFit(dmat, yvec, myindex))
#  user  system elapsed
#  1.00    0.00    0.14
system.time(res2 <- apply(myindex, 2, function(i) coef(lm(y ~ ., data = dall[i+1, ]))))
#  user  system elapsed
#  4.59    0.00    4.59
 
stopifnot(all(abs(res1 - res2) < 1e-8))
```

C++ code:

``` c++
// [[Rcpp::depends(RcppParallel)]]
// [[Rcpp::depends(RcppEigen)]]
#include <Rcpp.h>
#include <RcppEigen.h>
#include <RcppParallel.h>
 
using namespace std;
using namespace Rcpp;
using namespace RcppParallel;
 
using Eigen::Map;
using Eigen::MatrixXd;
using Eigen::VectorXd;
using Eigen::MatrixXi;
using Eigen::VectorXi;
 
template<typename Derived>
inline MatrixXd matIdx(const Eigen::MatrixBase<Derived>& X, const VectorXi& index)
{
  MatrixXd X2(index.size(), X.cols());
  for (unsigned int i = 0; i < index.size(); ++i)
    X2.row(i) = X.row(index(i));
  return X2;
}
 
struct CVLM : public Worker
{
  // design matrix and outcome
  const MatrixXd& X;
  const VectorXd& y;
  const MatrixXi& index;
 
  // output
  MatrixXd& betamat;
 
  // initialize with input and output
  CVLM(const MatrixXd& X, const VectorXd& y, const MatrixXi& index, MatrixXd& betamat)
    : X(X), y(y), index(index), betamat(betamat) {}
 
  void operator()(std::size_t begin, std::size_t end) {
    for(unsigned int i = begin; i < end; ++i)
    {
      MatrixXd Xi = matIdx(X, index.col(i));
      VectorXd yi = matIdx(y, index.col(i));
      betamat.col(i) = Xi.colPivHouseholderQr().solve(yi);
    }
  }
};
 
// [[Rcpp::export]]
Eigen::MatrixXd parallelFit(Rcpp::NumericMatrix xr,
                            Rcpp::NumericVector dvr,
                            Rcpp::IntegerMatrix indexr) {
  MatrixXd x = MatrixXd::Map(xr.begin(), xr.nrow(), xr.ncol());
  VectorXd dv = VectorXd::Map(dvr.begin(), dvr.size());
  MatrixXi index = MatrixXi::Map(indexr.begin(), indexr.nrow(), indexr.ncol());
 
  // allocate the output matrix
  MatrixXd betamat = MatrixXd::Zero(x.cols(), index.cols());
 
  // pass input and output
  CVLM cvLM(x, dv, index, betamat);
 
  // parallelFor to do it
  parallelFor(0, index.cols(), cvLM);
 
  return betamat;
}
```
