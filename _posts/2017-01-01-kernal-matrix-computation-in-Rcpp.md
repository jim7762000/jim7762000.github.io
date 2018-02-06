---
layout: post
title: "kernal matrix computation in Rcpp"
---

會有這篇是剛好想到之前在PTT PO的文章

[文章1](https://www.ptt.cc/bbs/R_Language/M.1429191476.A.F82.html)，[文章2](https://www.ptt.cc/bbs/C_and_CPP/M.1392479791.A.222.html)

就想說順便來把這兩個拿來比較一下，於是就有了這一篇的誕生

這篇剛好也是我試到一個RcppEigen比較慢的例子

當做之前的RcppArmadillo vs RcppEigen的延伸

我這次也觀察到一個現象是RcppArmadillo在這個計算上會使用BLAS

所以它這裡會用我R的Multi-threaded MKL BLAS去算

而RcppEigen機制不太確定，但是有趣的事情是Eigen還可以透過include omp配上`Eigen::initParallel`來加速

但是這裡RcppArmadillo毫無疑問的直接打趴RcppEigen(攤手

下一篇會再把RcppParallel拉進來一起玩

R code:

``` R
library(kernlab)
Rcpp::sourceCpp("kernel_matrix_arma.cpp")
Rcpp::sourceCpp("kernel_matrix_eigen1.cpp")
Rcpp::sourceCpp("kernel_matrix_eigen2.cpp")
 
N <- 5000L
p <- 1000L
b <- 500L
X <- matrix(rnorm(N*p), ncol = p)
center <- X[sample(N, b),]
sigma <- 2.5

kernelMatrix_R <- function(X, center, sigma){
  exp(sweep(sweep(X %*% t(center), 1,
                  rowSums(X**2)/2), 2, rowSums(center**2)/2) / (sigma**2))
}

res_kernlab <- kernelMatrix(rbfdot(sigma=1/(2*sigma^2)), X, center)@.Data

# check results with kernlab::kernelMatrix
all.equal(kernelMatrix_R(X, center, sigma), res_kernlab)             # TRUE
all.equal(kernelMatrix_arma_cpp(X, center, sigma), res_kernlab)      # TRUE
all.equal(kernelMatrix_eigen_cpp(X, center, sigma), res_kernlab)     # TRUE
all.equal(kernelMatrix_eigen_omp_cpp(X, center, sigma), res_kernlab) # TRUE

library(microbenchmark)
microbenchmark(
  Rfun = kernelMatrix_R(X, center, sigma),
  kernlab = kernelMatrix(rbfdot(sigma=1/(2*sigma^2)), X, center),
  RcppArmadillo = kernelMatrix_arma_cpp(X, center, sigma),
  RcppEigen = kernelMatrix_eigen_cpp(X, center, sigma),
  RcppEigen_Openmp = kernelMatrix_eigen_omp_cpp(X, center, sigma),
  times = 30L
)
# Unit: milliseconds
#              expr      min       lq     mean   median       uq      max neval
#              Rfun 224.3423 245.0635 258.5608 256.1671 267.4266 324.6571    30
#           kernlab 229.4637 239.9462 265.5297 265.6573 282.5231 348.2940    30
#     RcppArmadillo 165.3531 174.5219 188.2199 184.6343 200.6425 226.9207    30
#         RcppEigen 416.0304 417.4070 424.4944 418.7338 426.4974 464.8256    30
#  RcppEigen_Openmp 202.4775 208.1299 248.6041 229.6451 290.2074 333.7440    30
```

kernel_matrix_arma.cpp:

``` c++
// [[Rcpp::depends(RcppArmadillo)]]
#include <RcppArmadillo.h>
using arma::mat;
using arma::vec;
using arma::rowvec;

// [[Rcpp::export]]
mat kernelMatrix_arma_cpp(mat x, mat center, double sigma) {
  mat kernelMat(x * center.t());
  vec x_square_sum = sum(square(x), 1) / 2.0;
  rowvec center_square_sum = (sum(square(center), 1)).t() / 2.0;
  kernelMat.each_row() -= center_square_sum;
  kernelMat.each_col() -= x_square_sum;
  kernelMat /= pow(sigma, 2.0);
  return exp(kernelMat);
}
```

kernel_matrix_eigen1.cpp:

``` c++
// [[Rcpp::depends(RcppEigen)]]
#include <RcppEigen.h>
using Eigen::MatrixXd;
using Eigen::VectorXd;
using Eigen::VectorXi;

// [[Rcpp::export]]
MatrixXd kernelMatrix_eigen_cpp(MatrixXd x, MatrixXd center, double sigma) {
  MatrixXd kernelMat = x * center.transpose();
  VectorXd x_square_sum = x.array().pow(2.0).rowwise().sum() / 2.0;
  VectorXd center_square_sum = center.array().pow(2.0).rowwise().sum()  / 2.0;
  kernelMat.rowwise() -= center_square_sum.transpose();
  kernelMat.colwise() -= x_square_sum;
  kernelMat /=  pow(sigma, 2.0);
  return kernelMat.array().exp().matrix();
}
```

kernel_matrix_eigen2.cpp:

``` c++
// [[Rcpp::depends(RcppEigen)]]
// [[Rcpp::plugins(openmp, cpp11)]]
#include <RcppEigen.h>
#include <omp.h>
using Eigen::MatrixXd;
using Eigen::VectorXd;
using Eigen::VectorXi;

// [[Rcpp::export]]
MatrixXd kernelMatrix_eigen_omp_cpp(MatrixXd x, MatrixXd center, double sigma) {
  int n = omp_get_max_threads();
  omp_set_num_threads(n);
  Eigen::setNbThreads(n);
  Eigen::initParallel();
  MatrixXd kernelMat = x * center.transpose();
  VectorXd x_square_sum = x.array().pow(2.0).rowwise().sum() / 2.0;
  VectorXd center_square_sum = center.array().pow(2.0).rowwise().sum()  / 2.0;
  kernelMat.rowwise() -= center_square_sum.transpose();
  kernelMat.colwise() -= x_square_sum;
  kernelMat /=  pow(sigma, 2.0);
  return kernelMat.array().exp().matrix();
}
```
