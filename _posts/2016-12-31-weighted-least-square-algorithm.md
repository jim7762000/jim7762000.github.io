---
layout: post
title: "Weighted Least-Square Algorithm"
---

最近一直work on locally weighted least-square

結果發現使用`gausvar`這個kernel的時候，weight會出現負的

我原本的解法是直接在input跟output都乘上根號的weight

結果這招就行不通了

另外還有再一些case下，解不是穩健的，有可能跑出虛數，但是虛數的係數其實很小...

所以就下定決心來研究一下各種WLS的解法


稍微GOOGLE了一下，發現不外乎下面四種解法：

1. 直接解，就是利用`(X^T * W * X)^(-1) * X^T * W * y`去解出迴歸係數
1. 再來就是把inverse部分用pseudo inverse代替，以避免rank不足的問題出現
1. Cholesky Decomposition (LDL Decomposition)
1. QR Decomposition

效率的話，`4 > 3 > 1 > 2`，但是QR在一些情況下會跑出虛數

所以我這裡會偏向以3為主


下面是用程式去實作各種解法：

R code:

``` R
Rcpp::sourceCpp("wls.cpp")
n <- 500
p <- 150
X <- matrix(rnorm(n * p), n , p)
beta <- rnorm(p)
w <- sqrt(rowMeans(X**2) - (n / (n-1)) * rowMeans(X)**2)
y <- 3 + X %*% beta + rnorm(n)

library(microbenchmark)
microbenchmark(
  eigen_llt = eigen_llt(X, w, y),
  eigen_ldlt = eigen_ldlt(X, w, y),
  eigen_fullLU = eigen_fullLU(X, w, y),
  eigen_HHQR = eigen_HHQR(X, w, y),
  eigen_colPivHHQR = eigen_colPivHHQR(X, w, y),
  eigen_fullPivHHQR = eigen_fullPivHHQR(X, w, y),
  eigen_chol_llt1 = eigen_chol_llt1(X, w, y),
  eigen_chol_llt2 = eigen_chol_llt2(X, w, y),
  eigen_chol_llt3 = eigen_chol_llt3(X, w, y),
  arma_qr = arma_qr(X, w, y), # can't run if n is too big
  arma_pinv = arma_pinv(X, w, y),
  arma_chol1 = arma_chol1(X, w, y),
  arma_chol2 = arma_chol2(X, w, y),
  arma_direct1 = arma_direct1(X, w, y),
  arma_direct2 = arma_direct2(X, w, y),
  r_lm = coef(lm(y ~ -1 + X, weights = w)),
  times = 30L
)
# Unit: microseconds
#               expr      min       lq       mean    median       uq        max neval
#          eigen_llt 1787.601 1814.225  2341.2993 2287.1645 2889.126   2912.239    30
#         eigen_ldlt 1812.763 1846.408  2292.7815 2089.9725 2928.916   3020.197    30
#       eigen_fullLU 3112.649 3133.129  3673.1350 3242.1115 4610.021   4725.294    30
#         eigen_HHQR 2334.999 2401.120  3095.5537 3073.1525 3820.669   3920.142    30
#   eigen_colPivHHQR 2411.945 2423.941  2969.3488 2691.4950 3756.888   4029.855    30
#  eigen_fullPivHHQR 3449.397 3477.776  4179.7097 3585.0035 5282.639   5363.389    30
#    eigen_chol_llt1 3302.234 3359.286  4111.7262 3828.5675 5297.852   5362.510    30
#    eigen_chol_llt2 3268.004 3308.379  4065.1882 3418.3850 5253.674   5296.390    30
#    eigen_chol_llt3 3354.020 3397.027  4130.9969 3857.0925 4929.800   5425.121    30
#            arma_qr 1868.936 2167.357  2330.0549 2316.5675 2419.552   2829.442    30
#          arma_pinv 4137.229 4723.245  5024.8460 4877.1375 5474.272   5722.957    30
#         arma_chol1  702.167  959.337  1042.1535 1041.1100 1147.167   1291.404    30
#         arma_chol2  780.869 1046.523  1121.4886 1125.5160 1234.645   1423.645    30
#       arma_direct1 4473.977 4636.645  4956.8919 4701.4490 5481.294   5867.193    30
#       arma_direct2  676.129  898.482   963.5788  965.4805 1060.565   1184.615    30
#               r_lm 6257.189 6817.459 11962.0246 8113.9820 9301.084 123398.876    30
```

C++ code:

``` c++
// [[Rcpp::depends(RcppArmadillo, RcppEigen)]]
#include <RcppArmadillo.h>
#include <RcppEigen.h>
using namespace arma;

// [[Rcpp::export]]
Eigen::VectorXd eigen_fullPivHHQR(const Eigen::Map<Eigen::MatrixXd> & X,
                                  const Eigen::Map<Eigen::VectorXd> & w,
                                  const Eigen::Map<Eigen::VectorXd> & y) {
  Eigen::VectorXd beta = (X.transpose() * w.asDiagonal() * X).fullPivHouseholderQr().solve(X.transpose() * w.asDiagonal() * y);;
  return beta;
}

// [[Rcpp::export]]
Eigen::VectorXd eigen_colPivHHQR(const Eigen::Map<Eigen::MatrixXd> & X,
                                 const Eigen::Map<Eigen::VectorXd> & w,
                                 const Eigen::Map<Eigen::VectorXd> & y) {
  Eigen::VectorXd beta = (X.transpose() * w.asDiagonal() * X).colPivHouseholderQr().solve(X.transpose() * w.asDiagonal() * y);;
  return beta;
}

// [[Rcpp::export]]
Eigen::VectorXd eigen_HHQR(const Eigen::Map<Eigen::MatrixXd> & X,
                           const Eigen::Map<Eigen::VectorXd> & w,
                           const Eigen::Map<Eigen::VectorXd> & y) {
  Eigen::VectorXd beta = (X.transpose() * w.asDiagonal() * X).householderQr().solve(X.transpose() * w.asDiagonal() * y);
  return beta;
}

// [[Rcpp::export]]
Eigen::VectorXd eigen_fullLU(const Eigen::Map<Eigen::MatrixXd> & X, 
                             const Eigen::Map<Eigen::VectorXd> & w, 
                             const Eigen::Map<Eigen::VectorXd> & y) {
  Eigen::VectorXd beta = (X.transpose() * w.asDiagonal() * X).fullPivLu().solve(X.transpose() * w.asDiagonal() * y);
  return beta;
}
  
// [[Rcpp::export]]
Eigen::VectorXd eigen_llt(const Eigen::Map<Eigen::MatrixXd> & X, 
                          const Eigen::Map<Eigen::VectorXd> & w, 
                          const Eigen::Map<Eigen::VectorXd> & y) {
  Eigen::VectorXd beta = (X.transpose() * w.asDiagonal() * X).llt().solve(X.transpose() * w.asDiagonal() * y);
  return beta;
}

// [[Rcpp::export]]
Eigen::VectorXd eigen_ldlt(const Eigen::Map<Eigen::MatrixXd> & X, 
                           const Eigen::Map<Eigen::VectorXd> & w, 
                           const Eigen::Map<Eigen::VectorXd> & y) {
  Eigen::VectorXd beta = (X.transpose() * w.asDiagonal() * X).ldlt().solve(X.transpose() * w.asDiagonal() * y);
  return beta;
}

// [[Rcpp::export]]
Eigen::VectorXd eigen_chol_llt1(const Eigen::Map<Eigen::MatrixXd> & X,
                                const Eigen::Map<Eigen::VectorXd> & w,
                                const Eigen::Map<Eigen::VectorXd> & y) {
  Eigen::MatrixXd XWX = X.transpose() * w.asDiagonal() * X;
  Eigen::MatrixXd R = XWX.llt().matrixU();
  Eigen::VectorXd XWY = X.transpose() * (w.array() * y.array()).matrix();
  Eigen::VectorXd beta = R.householderQr().solve(R.transpose().householderQr().solve(XWY));
  return beta;
}

// [[Rcpp::export]]
Eigen::VectorXd eigen_chol_llt2(const Eigen::Map<Eigen::MatrixXd> & X,
                                const Eigen::Map<Eigen::VectorXd> & w,
                                const Eigen::Map<Eigen::VectorXd> & y) {
  Eigen::MatrixXd XW = X.transpose() * w.asDiagonal();
  Eigen::MatrixXd R = (XW * X).llt().matrixU();
  Eigen::VectorXd beta = R.householderQr().solve(R.transpose().householderQr().solve(XW * y));
  return beta;
}

// [[Rcpp::export]]
Eigen::VectorXd eigen_chol_llt3(const Eigen::Map<Eigen::MatrixXd> & X,
                                const Eigen::Map<Eigen::VectorXd> & w,
                                const Eigen::Map<Eigen::VectorXd> & y) {
  Eigen::MatrixXd XW(X.cols(), X.rows());
  for (unsigned int i = 0; i < X.cols(); ++i)
    XW.row(i) = X.col(i).array() * w.array();
  Eigen::MatrixXd R = (XW * X).llt().matrixU();
  Eigen::VectorXd beta = R.householderQr().solve(R.transpose().householderQr().solve(XW * y));
  return beta;
}

// [[Rcpp::export]]
Eigen::VectorXd eigen_colPivHHQR2(const Eigen::Map<Eigen::MatrixXd> & X,
                                  const Eigen::Map<Eigen::VectorXd> & w,
                                  const Eigen::Map<Eigen::VectorXd> & y) {
  Eigen::VectorXd sw = w.cwiseSqrt();
  Eigen::MatrixXd XW(X.rows(), X.cols());
  for (unsigned int i = 0; i < X.cols(); ++i)
    XW.col(i) = X.col(i).array() * sw.array();
  Eigen::VectorXd beta = XW.colPivHouseholderQr().solve(y.cwiseProduct(sw));
  return beta;
}

// [[Rcpp::export]]
arma::vec arma_qr(const arma::mat& X, const arma::vec& w, const arma::vec& y) {
  mat Q, R;
  arma::qr_econ(Q, R, X.each_col() % sqrt(w));
  vec p = solve(R, Q.t() * (y % sqrt(w)));
  return p;
}

// [[Rcpp::export]]
arma::vec arma_pinv(const arma::mat& X, const arma::vec& w, const arma::vec& y) {
  vec p = pinv(X.t() * (repmat(w, 1, X.n_cols) % X)) * X.t() * (w % y);
  return p;
}

// [[Rcpp::export]]
arma::vec arma_chol1(const arma::mat& X, const arma::vec& w, const arma::vec& y) {
  mat R = chol((X.each_col() % w).t() * X);
  vec p = solve(R, solve(R.t(), X.t() * (w % y), solve_opts::fast), solve_opts::fast);
  return p;
}

// [[Rcpp::export]]
arma::vec arma_chol2(const arma::mat& X, const arma::vec& w, const arma::vec& y) {
  mat XW = (X.each_col() % w).t();
  mat R = chol(XW * X);
  vec p = solve(R, solve(R.t(), XW * y, solve_opts::fast), solve_opts::fast);
  return p;
}

// [[Rcpp::export]]
arma::vec arma_direct1(const arma::mat& X, const arma::vec& w, const arma::vec& y) {
  vec sw = sqrt(w);
  vec p = solve(X.each_col() % sw, y % sw);
  return p;
}

// [[Rcpp::export]]
arma::vec arma_direct2(const arma::mat& X, const arma::vec& w, const arma::vec& y) {
  vec p = solve((X.each_col() % w).t() * X, X.t() * (w % y));
  return p;
}
```

這裡的QR解得很慢，我不知道要怎麼樣讓armadillo只輸出跟rank一樣多的Q，R矩陣就好

而直接解會是最快的，我猜這原因是裡面某部分有被優化過了...

不然以程式來看，Cholesky Decomposition的performance是最好的

只是我也不解的是Eigen也用一樣的方法去做

卻比Armadillo手動去寫慢了好幾倍 (eigen_chol_llt vs arma_chol)

不確定是不是Eigen在solve linear system時用不一樣的LAPACK函數

或是Eigen在這做了比較多check

這裡就留給後人慢慢玩賞QQ


2017/4/20補充：

我後來試了一些簡單的case

發現其實在p不大(大概小於80)，n也不大(小於200)的情況

RcppEigen擁有比較好的performance，我的猜測是SSE指令集帶來的好處

![](/images/wls_performace_comparison1.png)
![](/images/wls_performace_comparison2.png)

測試script如下：

``` R
library(iterators)
library(foreach)
library(data.table)
library(pipeR)
library(Rcpp)
library(RcppArmadillo)
library(RcppEigen)
library(microbenchmark)

Rcpp::sourceCpp("wls.cpp")

trainFunc <- function(n, p){
  X <- matrix(rnorm(n * p), n , p)
  beta <- rnorm(p)
  w <- rgamma(nrow(X), 2, 0.5)
  y <- 3 + X %*% beta + rnorm(n)
  
  microRes <- microbenchmark(
    eigen_llt = eigen_llt(X, w, y),
    eigen_ldlt = eigen_ldlt(X, w, y),
    eigen_chol_llt1 = eigen_chol_llt1(X, w, y),
    eigen_chol_llt2 = eigen_chol_llt2(X, w, y),
    eigen_chol_llt3 = eigen_chol_llt3(X, w, y),
    arma_chol1 = arma_chol1(X, w, y),
    arma_chol2 = arma_chol2(X, w, y),
    arma_direct1 = arma_direct1(X, w, y),
    arma_direct2 = arma_direct2(X, w, y),
    r_lm = coef(lm(y ~ -1 + X, weights = w)),
    times = 30L
  )
  m <- tapply(microRes$time, microRes$expr, median) / 1000
  return(data.table(n = n, p = p, method = names(m), median_time = m))
}

paraList <- CJ(p = c(1:20, seq(25, 100, 5)), n = c(20, 30, 50, 75, 100, 200)) %>>% `[`(n > p)
paraResDT <- foreach(v = iter(paraList, by = "row"), .final = rbindlist) %do% trainFunc(v$n, v$p)

library(lattice)

xyplot(median_time ~ p | factor(n, c(20, 30, 50, 75, 100, 200)), paraResDT, groups = method, type = "o", 
       auto.key = list(points = TRUE, columns = 3), 
       scales = list(x = list(relation = "free"), y = list(relation = "free")))

xyplot(median_time ~ p | factor(n, c(20, 30, 50, 75, 100, 200)), 
       paraResDT[!method %in% c("r_lm", "arma_direct1", "eigen_chol_llt1",
                                "eigen_chol_llt2", "eigen_chol_llt3")], 
       groups = method, type = "o", auto.key = list(points = TRUE, columns = 3), 
       scales = list(x = list(relation = "free"), y = list(relation = "free")))
```

基於上面的結論，所以我會建議這樣去寫wls的solver:

```
// [[Rcpp::depends(RcppArmadillo, RcppEigen)]]
#include <RcppArmadillo.h>
#include <RcppEigen.h>

// [[Rcpp::export]]
arma::vec fastSolve(arma::mat X, arma::vec w, arma::vec y) {
  if (X.n_rows * X.n_cols <= 6000) {
    Eigen::MatrixXd X2 = Eigen::MatrixXd::Map(X.memptr(), X.n_rows, X.n_cols);
    Eigen::VectorXd w2 = Eigen::VectorXd::Map(w.memptr(), w.size());
    Eigen::VectorXd y2 = Eigen::VectorXd::Map(y.memptr(), y.size());
    Eigen::VectorXd out = (X2.transpose() * w2.asDiagonal() * X2).llt().solve(X2.transpose() * w2.asDiagonal() * y2);
    arma::vec p(out.data(), out.rows(), false);
    return p;
  } else {
    return arma::solve((X.each_col() % w).t() * X, X.t() * (w % y));
  }
}
```