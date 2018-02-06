---
layout: post
title: "RcppEigen Work with RcppParallel"
---

最近在比較RcppEigen跟RcppArmadillo，比較的結果寫到下一篇好了

就想說也來試試看RcppEigen跟RcppParallel結合看看有什麼火花

這裡是改之前的RcppArmadillo with RcppParallel的程式 - fastLOOCV

順便也把RcppEigen vs RcppArmadillo 跟 Openmp vs RcppParallel 一起放入混合比較一下

R code: 

``` R
Rcpp::sourceCpp("RcppLOOCV.cpp")
 
N <- 600L
p <- 150L
X <- matrix(rnorm(N*p), N)
b <- rnorm(p, rnorm(2L), rgamma(2L,3,7))
y <- 1.5 + as.vector(X %*% b) + rnorm(N)
 
system.time({
  out <- sapply(1L:N, function(i){
    idx <- setdiff(1L:N, i)
    b <- coef(lm(y[idx] ~ X[idx , ]))
    return((y[i] - crossprod(c(1, X[i, ]), b))**2)
  })
})
#  user  system elapsed
#  8.83    0.28    9.20
 
# check results
for (i in 1L:5L) stopifnot(abs(mean(out) - arma_fastLOOCV1(y, X)) < 1e-8)
for (i in 1L:5L) stopifnot(abs(mean(out) - arma_fastLOOCV2(y, X)) < 1e-8)
for (i in 1L:5L) stopifnot(abs(mean(out) - eigen_fastLOOCV1(y, X)) < 1e-8)
for (i in 1L:5L) stopifnot(abs(mean(out) - eigen_fastLOOCV2(y, X)) < 1e-8)
for (i in 1L:5L) stopifnot(abs(mean(out) - eigen_fastLOOCV3(y, X)) < 1e-8)
for (i in 1L:5L) stopifnot(abs(mean(out) - eigen_fastLOOCV4(y, X)) < 1e-8)
 
library(microbenchmark)
microbenchmark(
  arma_fastLOOCV1(y, X),  # RcppArmadillo with RcppPrallel
  arma_fastLOOCV2(y, X),  # RcppArmadillo with openmp
  eigen_fastLOOCV1(y, X), # RcppEigen with RcppPrallel (Reduce)
  eigen_fastLOOCV2(y, X), # RcppEigen with RcppPrallel (For)
  eigen_fastLOOCV3(y, X), # RcppEigen with openmp
  eigen_fastLOOCV4(y, X), # RcppEigen without parallism
  times = 30L
)
# Unit: milliseconds
#                    expr       min        lq      mean    median        uq       max neval
#   arma_fastLOOCV1(y, X) 1009.9239 1037.4614 1052.9272 1045.9733 1068.5668 1101.5399    30
#   arma_fastLOOCV2(y, X) 1020.5068 1033.6180 1051.7171 1046.8751 1064.4512 1139.6371    30
#  eigen_fastLOOCV1(y, X)  623.7900  628.0837  639.1062  635.0657  644.8178  716.8571    30
#  eigen_fastLOOCV2(y, X)  620.1337  624.2227  634.4193  630.3217  641.1066  682.0993    30
#  eigen_fastLOOCV3(y, X)  620.0021  630.6396  643.2877  638.7085  655.9284  706.5759    30
#  eigen_fastLOOCV4(y, X) 2627.1159 2648.5870 2657.8932 2654.8032 2663.2681 2707.9168    30
```

C++ code:

``` c++
// [[Rcpp::depends(RcppArmadillo, RcppEigen, RcppParallel)]]
// [[Rcpp::plugins(openmp)]]
#include <RcppArmadillo.h>
#include <RcppEigen.h>
#include <omp.h>
#include <RcppParallel.h>
using arma::mat;
using arma::vec;
using arma::uvec;
using arma::uword;
using Eigen::Map;
using Eigen::MatrixXd;
using Eigen::VectorXd;
 
struct arma_MSE_Compute: public RcppParallel::Worker {
  const mat& X;
  const vec& y;
  const uvec& index;
  double mse;
 
  arma_MSE_Compute(const mat& X, const vec& y, const uvec& index):
    X(X), y(y), index(index), mse(0.0) {}
 
  arma_MSE_Compute(const arma_MSE_Compute& arma_MSE_worker, RcppParallel::Split):
    X(arma_MSE_worker.X), y(arma_MSE_worker.y), index(arma_MSE_worker.index), mse(0.0) {}
 
  void operator()(std::size_t begin, std::size_t end) {
    for (uword i = begin; i < end; ++i) {
      uvec idx = arma::find(index != i);
      mse += pow(y(i) - arma::dot(X.row(i), arma::solve(X.rows(idx), y.elem(idx))), 2.0);
    }
  }
 
  void join(const arma_MSE_Compute& rhs) {
    mse += rhs.mse;
  }
};
 
// [[Rcpp::export]]
double arma_fastLOOCV1(const arma::vec& y, const arma::mat& X) {
  mat X_with_ones = arma::join_rows(arma::ones<vec>(X.n_rows), X);
  uvec index = arma::linspace<uvec>(0, y.n_elem - 1, y.n_elem);
 
  arma_MSE_Compute mseResults(X_with_ones, y, index);
  RcppParallel::parallelReduce(0, y.n_elem, mseResults);
  return mseResults.mse / y.n_elem;
}
 
// [[Rcpp::export]]
double arma_fastLOOCV2(const arma::vec& y, const arma::mat& X) {
  mat X_with_ones = arma::join_rows(arma::ones<vec>(X.n_rows), X);
  uvec index = arma::linspace<uvec>(0, y.n_elem - 1, y.n_elem);
  vec mse = arma::zeros<vec>(y.n_elem);
 
  uword i = 0;
  #pragma omp parallel for private(i)
  for (i = 0; i < y.n_elem; ++i) {
    uvec idx = arma::find(index != i);
    mse(i) = pow(y(i) - arma::dot(X_with_ones.row(i), arma::solve(X_with_ones.rows(idx), y.elem(idx))), 2.0);
  }
  return mean(mse);
}
 
struct eigen_MSE_Compute: public RcppParallel::Worker {
  MatrixXd X;
  VectorXd y;
  double mse;
 
  eigen_MSE_Compute(MatrixXd X, VectorXd y):
    X(X), y(y), mse(0.0) {}
 
  eigen_MSE_Compute(const eigen_MSE_Compute& eigen_MSE_worker, RcppParallel::Split):
    X(eigen_MSE_worker.X), y(eigen_MSE_worker.y), mse(0.0) {}
 
  void operator()(std::size_t begin, std::size_t end) {
    for (unsigned int i = begin; i < end; ++i) {
      MatrixXd tmpX(X.rows() - 1, X.cols());
      VectorXd tmpY(y.size() - 1);
      if (i == 0) {
        tmpX = X.bottomRows(X.rows() - 1);
        tmpY = y.tail(y.size() - 1);
      } else if (i == X.rows() - 1) {
        tmpX = X.topRows(X.rows() - 1);
        tmpY = y.head(y.size() - 1);
      } else {
        tmpX << X.topRows(i),
                X.bottomRows(X.rows() - i - 1);
        tmpY << y.head(i),
                y.tail(y.size() - i - 1);
      }
      mse += pow(y(i) - (tmpX.colPivHouseholderQr().solve(tmpY)).dot(X.row(i)), 2.0);
    }
  }
 
  void join(const eigen_MSE_Compute& rhs) {
    mse += rhs.mse;
  }
};
 
// [[Rcpp::export]]
double eigen_fastLOOCV1(const Eigen::Map<VectorXd>& y,
                        const Eigen::Map<MatrixXd>& X) {
  MatrixXd X_with_ones(X.rows(), X.cols() + 1);
  X_with_ones << MatrixXd::Ones(X.rows(), 1), X;
 
  eigen_MSE_Compute mseResults(X_with_ones, y);
  RcppParallel::parallelReduce(0, y.size(), mseResults);
  return mseResults.mse / y.size();
}
 
struct eigen_MSE_Compute2: public RcppParallel::Worker {
  const MatrixXd& X;
  const VectorXd& y;
  VectorXd& mse;
 
  eigen_MSE_Compute2(const MatrixXd& X, const VectorXd& y, VectorXd& mse):
    X(X), y(y), mse(mse) {}
 
  void operator()(std::size_t begin, std::size_t end) {
    for (unsigned int i = begin; i < end; ++i) {
      MatrixXd tmpX(X.rows() - 1, X.cols());
      VectorXd tmpY(y.size() - 1);
      if (i == 0) {
        tmpX = X.bottomRows(X.rows() - 1);
        tmpY = y.tail(y.size() - 1);
      } else if (i == X.rows() - 1) {
        tmpX = X.topRows(X.rows() - 1);
        tmpY = y.head(y.size() - 1);
      } else {
        tmpX << X.topRows(i),
                X.bottomRows(X.rows() - i - 1);
        tmpY << y.head(i),
                y.tail(y.size() - i - 1);
      }
      mse(i) = pow(y(i) - (tmpX.colPivHouseholderQr().solve(tmpY)).dot(X.row(i)), 2.0);
    }
  }
};
 
// [[Rcpp::export]]
double eigen_fastLOOCV2(Rcpp::NumericVector yin,
                        const Eigen::Map<MatrixXd>& X) {
  // Eigen::Map<VectorXd>& object in RcppParall::Worker would cause crash
  // but we can input Rcpp::NumericVector, and use VectorXd::Map to convert to VectorXd
  VectorXd y = VectorXd::Map(yin.begin(), yin.size());
  MatrixXd X_with_ones(X.rows(), X.cols() + 1);
  X_with_ones << MatrixXd::Ones(X.rows(), 1), X;
  VectorXd mse = VectorXd::Zero(y.size());
 
  eigen_MSE_Compute2 mseResults(X_with_ones, y, mse);
  RcppParallel::parallelFor(0, y.size(), mseResults);
  return mse.mean();
}
 
// [[Rcpp::export]]
double eigen_fastLOOCV3(const Eigen::Map<VectorXd>& y,
                        const Eigen::Map<MatrixXd>& X) {
  MatrixXd X_with_ones(X.rows(), X.cols() + 1);
  X_with_ones << MatrixXd::Ones(X.rows(), 1), X;
 
  VectorXd mse = VectorXd::Zero(y.size());
  unsigned int i = 0;
 
  #pragma omp parallel for private(i)
  for (i = 0; i < X.rows(); ++i) {
    MatrixXd tmpX(X.rows() - 1, X_with_ones.cols());
    VectorXd tmpY(y.size() - 1);
    if (i == 0) {
      tmpX = X_with_ones.bottomRows(X_with_ones.rows() - 1);
      tmpY = y.tail(y.size() - 1);
    } else if (i == X_with_ones.rows() - 1) {
      tmpX = X_with_ones.topRows(X_with_ones.rows() - 1);
      tmpY = y.head(y.size() - 1);
    } else {
      tmpX << X_with_ones.topRows(i),
              X_with_ones.bottomRows(X_with_ones.rows() - i - 1);
      tmpY << y.head(i),
              y.tail(y.size() - i - 1);
    }
    mse(i) = pow(y(i) - (tmpX.colPivHouseholderQr().solve(tmpY)).dot(X_with_ones.row(i)), 2.0);
  }
  return mse.mean();
}
 
// [[Rcpp::export]]
double eigen_fastLOOCV4(const Eigen::Map<VectorXd>& y,
                        const Eigen::Map<MatrixXd>& X) {
  MatrixXd X_with_ones(X.rows(), X.cols() + 1);
  X_with_ones << MatrixXd::Ones(X.rows(), 1), X;
 
  double mse = 0.0;
  for (unsigned int i = 0; i < X.rows(); ++i) {
    MatrixXd tmpX(X.rows() - 1, X_with_ones.cols());
    VectorXd tmpY(y.size() - 1);
    if (i == 0) {
      tmpX = X_with_ones.bottomRows(X_with_ones.rows() - 1);
      tmpY = y.tail(y.size() - 1);
    } else if (i == X_with_ones.rows() - 1) {
      tmpX = X_with_ones.topRows(X_with_ones.rows() - 1);
      tmpY = y.head(y.size() - 1);
    } else {
      tmpX << X_with_ones.topRows(i),
              X_with_ones.bottomRows(X_with_ones.rows() - i - 1);
      tmpY << y.head(i),
              y.tail(y.size() - i - 1);
    }
    mse += pow(y(i) - (tmpX.colPivHouseholderQr().solve(tmpY)).dot(X_with_ones.row(i)), 2.0);
  }
  return mse / y.size();
}
```

結論：

Armadillo的程式雖然簡短漂亮，但是Performance真的不如Eigen來的好

至於Openmp與RcppParallel這裡比起來根本沒什麼差異

回到這篇的重點，`Eigen::Map<VectorXd>`這個的背後還是Rcpp::NumericVector或是R的SEXP

在input進去RcppParallel的時候，會出問題，所以都必須確定轉成`Eigen::VectorXd`才行

不然R會crash或是得到錯誤的答案，我為了測試這個花了好多心力Orz

