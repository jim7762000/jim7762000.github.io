---
layout: post
title: "kernal matrix computation in Rcpp (continued)"
---

kernal matrix computation這個主題不只被我用了一次

其實我在PTT分享RcppParallel也是用了這個當例子，[文章連結](https://www.ptt.cc/bbs/R_Language/M.1438914924.A.0B0.html)

那這裡就延續上篇的程式把RcppParallel一起拉進來比較一下吧

因為我已經知道p大的時候，我每一個都逐一算其實很慢

那我這裡的RcppParallel就改變一下做法

讓RcppParallel不會因為p改變而使得計算效率改變

這裡新增三個cpp檔案，分別是：RcppArmadillo

1.RcppArmadillo with RcppParallel
1. RcppEigen with RcppParallel
1. RcppEigen and OpenMP with RcppParallel

其他三個就在前一篇都有了，這裡就不重複貼了

R code:

``` R
library(kernlab)
Rcpp::sourceCpp("kernel_matrix_arma.cpp")
Rcpp::sourceCpp("kernel_matrix_eigen1.cpp")
Rcpp::sourceCpp("kernel_matrix_eigen2.cpp")
Rcpp::sourceCpp("kernel_matrix_arma_para.cpp")
Rcpp::sourceCpp("kernel_matrix_eigen_para1.cpp")
Rcpp::sourceCpp("kernel_matrix_eigen_para2.cpp")
 
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
all.equal(kernelMatrix_arma_para_cpp(X, center, sigma), res_kernlab) # TRUE
all.equal(kernelMatrix_eigen_para_cpp(X, center, sigma), res_kernlab) # TRUE
all.equal(kernelMatrix_eigen_para_omp_cpp(X, center, sigma), res_kernlab) # TRUE

library(microbenchmark)
microbenchmark(
  Rfun = kernelMatrix_R(X, center, sigma),
  kernlab = kernelMatrix(rbfdot(sigma=1/(2*sigma^2)), X, center),
  RcppArmadillo = kernelMatrix_arma_cpp(X, center, sigma),
  RcppEigen = kernelMatrix_eigen_cpp(X, center, sigma),
  RcppEigen_Openmp = kernelMatrix_eigen_omp_cpp(X, center, sigma),
  RcppArmadillo_RcppParallel = kernelMatrix_arma_cpp(X, center, sigma),
  RcppEigen_RcppParallel = kernelMatrix_eigen_para_cpp(X, center, sigma),
  RcppEigen_RcppParallel_Openmp = kernelMatrix_eigen_para_omp_cpp(X, center, sigma),
  times = 30L
)
# Unit: milliseconds
#                           expr      min       lq     mean   median       uq      max neval
#                           Rfun 238.5369 255.9128 273.9832 268.1477 284.9538 348.0322    30
#                        kernlab 226.1284 248.5015 267.4762 261.4748 275.2627 372.2815    30
#                  RcppArmadillo 173.2054 188.7308 199.1266 193.7259 207.7513 284.9052    30
#                      RcppEigen 416.1434 419.6085 425.7891 422.5549 429.4397 456.2896    30
#               RcppEigen_Openmp 204.2332 210.2853 225.0568 214.8849 224.6437 324.5009    30
#     RcppArmadillo_RcppParallel 170.5573 196.8640 204.4135 207.8597 212.0638 224.6665    30
#         RcppEigen_RcppParallel 411.3365 416.3367 425.0875 419.2989 429.4684 459.8628    30
#  RcppEigen_RcppParallel_Openmp 203.5772 211.5603 246.5694 224.0399 277.4804 377.0851    30
```

kernel_matrix_arma_para.cpp:

``` c++
// [[Rcpp::depends(RcppArmadillo, RcppParallel)]]
#include <RcppArmadillo.h>
#include <RcppParallel.h>
using arma::mat;
using arma::vec;
using arma::rowvec;

struct KernelComputeWorker: public RcppParallel::Worker {
  const vec& x_square_sum;
  const rowvec& center_square_sum;
  const double& sigma2;
  mat& kernelMat;
  
  KernelComputeWorker(const vec& x_square_sum, const rowvec& center_square_sum, 
                      const double& sigma2, mat& kernelMat):
    x_square_sum(x_square_sum), center_square_sum(center_square_sum), 
    sigma2(sigma2), kernelMat(kernelMat) {}
  
  void operator()(std::size_t begin, std::size_t end) {
    for (int k = (int) begin; k < (int) end; ++k) {
      int j = k / kernelMat.n_rows;
      int i = k - j * kernelMat.n_rows;
      kernelMat(i, j) -= (x_square_sum(i) + center_square_sum(j));
      kernelMat(i, j) = exp(kernelMat(i, j) / sigma2);
    }
  }
};

// [[Rcpp::export]]
mat kernelMatrix_arma_para_cpp(mat x, mat center, double sigma) {
  mat kernelMat(x * center.t());
  vec x_square_sum = sum(square(x), 1) / 2.0;
  rowvec center_square_sum = (sum(square(center), 1)).t() / 2.0;
  double sigma2 = pow(sigma, 2.0);
  
  KernelComputeWorker worker_kc(x_square_sum, center_square_sum, sigma2, kernelMat);
  RcppParallel::parallelFor(0, kernelMat.n_elem, worker_kc);
  return kernelMat;
}
```

kernel_matrix_eigen_para1.cpp:

``` c++
// [[Rcpp::depends(RcppEigen, RcppParallel)]]
#include <RcppEigen.h>
#include <RcppParallel.h>
using Eigen::MatrixXd;
using Eigen::VectorXd;
using Eigen::VectorXi;

struct KernelComputeWorker: public RcppParallel::Worker {
  const VectorXd& x_square_sum;
  const VectorXd& center_square_sum;
  const double& sigma2;
  MatrixXd& kernelMat;
  
  KernelComputeWorker(const VectorXd& x_square_sum, const VectorXd& center_square_sum, 
                      const double& sigma2, MatrixXd& kernelMat):
    x_square_sum(x_square_sum), center_square_sum(center_square_sum), 
    sigma2(sigma2), kernelMat(kernelMat) {}
  
  void operator()(std::size_t begin, std::size_t end) {
    for (int k = (int) begin; k < (int) end; ++k) {
      int j = k / kernelMat.rows();
      int i = k - j * kernelMat.rows();
      kernelMat(i, j) -= (x_square_sum(i) + center_square_sum(j));
      kernelMat(i, j) = exp(kernelMat(i, j) / sigma2);
    }
  }
};

// [[Rcpp::export]]
MatrixXd kernelMatrix_eigen_para_cpp(MatrixXd x, MatrixXd center, double sigma) {
  MatrixXd kernelMat = x * center.transpose();
  VectorXd x_square_sum = x.array().pow(2.0).rowwise().sum() / 2.0;
  VectorXd center_square_sum = center.array().pow(2.0).rowwise().sum()  / 2.0;
  
  double sigma2 = pow(sigma, 2.0);
  KernelComputeWorker worker_kc(x_square_sum, center_square_sum, sigma2, kernelMat);
  RcppParallel::parallelFor(0, kernelMat.size(), worker_kc);
  return kernelMat;
}
```

kernel_matrix_eigen_para2.cpp:

``` c++
// [[Rcpp::depends(RcppEigen, RcppParallel)]]
// [[Rcpp::plugins(openmp)]]
#include <RcppEigen.h>
#include <omp.h>
#include <RcppParallel.h>
using Eigen::MatrixXd;
using Eigen::VectorXd;
using Eigen::VectorXi;

struct KernelComputeWorker: public RcppParallel::Worker {
  const VectorXd& x_square_sum;
  const VectorXd& center_square_sum;
  const double& sigma2;
  MatrixXd& kernelMat;
  
  KernelComputeWorker(const VectorXd& x_square_sum, const VectorXd& center_square_sum, 
                      const double& sigma2, MatrixXd& kernelMat):
    x_square_sum(x_square_sum), center_square_sum(center_square_sum), 
    sigma2(sigma2), kernelMat(kernelMat) {}
  
  void operator()(std::size_t begin, std::size_t end) {
    for (int k = (int) begin; k < (int) end; ++k) {
      int j = k / kernelMat.rows();
      int i = k - j * kernelMat.rows();
      kernelMat(i, j) -= (x_square_sum(i) + center_square_sum(j));
      kernelMat(i, j) = exp(kernelMat(i, j) / sigma2);
    }
  }
};

// [[Rcpp::export]]
MatrixXd kernelMatrix_eigen_para_omp_cpp(MatrixXd x, MatrixXd center, double sigma) {
  int n = omp_get_max_threads();
  omp_set_num_threads(n);
  Eigen::setNbThreads(n);
  Eigen::initParallel();
  MatrixXd kernelMat = x * center.transpose();
  VectorXd x_square_sum = x.array().pow(2.0).rowwise().sum() / 2.0;
  VectorXd center_square_sum = center.array().pow(2.0).rowwise().sum()  / 2.0;
  
  double sigma2 = pow(sigma, 2.0);
  KernelComputeWorker worker_kc(x_square_sum, center_square_sum, sigma2, kernelMat);
  RcppParallel::parallelFor(0, kernelMat.size(), worker_kc);
  return kernelMat;
}
```

這個結果可以看出幾件事情：

1. 直接用Multi-threaded BLAS的RcppArmadillo在這個問題上，用了RcppParallel還是略輸一籌，BLAS還是比較強大XD
1. 相較RcppEigen_RcppParallel跟RcppEigen_RcppParallel_Openmp來看，因為沒平行的部分只剩下一開始計算`kernelMat`、`x_square_sum`以及`center_square_sum`，所以這裡RcppEigen用它本身的BLAS庫來做乘除就慢上了不少，而後面的計算其實也是用colwise, rowwise以及element-wise的操作，那邊已經被平行做掉了，還是跟只單用RcppEigen差不多快，所以其實element-wise, colwise以及rowwise這幾個操作應該不至於造成瓶頸，因此，最大的瓶頸是在大矩陣的乘法，需要openmp使用multi-thread來加速
1. RcppEigen_RcppParallel_Openmp還是比RcppEigen_Openmp比，所以其實後段的處理是白費的，用原本的方法比較快 (RcppArmadillo的部分同理)


