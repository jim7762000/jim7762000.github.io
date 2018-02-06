---
layout: post
title: "Update RcppEigen to 3.3.1"
---

我花了一點時間把RcppEigen升級到3.3.1，然後我發現一件滿有趣的事情(細節請看 [Using BLAS/LAPACK from Eigen](https://eigen.tuxfamily.org/dox-devel/TopicUsingBlasLapack.html))

因為是我手動升級的，檔案放在我的github: [連結請點我](https://github.com/ChingChuan-Chen/RcppEigen)

就是`#define EIGEN_USE_BLAS`可以用了，設定之後，RcppEigen會使用R的BLAS去做計算

如此一來，之前講到的RcppEigen的慢就可以被改善了

拿前一篇kernel matrix computation來測試看看

新增兩個cpp檔案，其他之前幾篇就有，不再附上

R code:

``` R
library(kernlab)
Rcpp::sourceCpp("kernel_matrix_arma.cpp")
Rcpp::sourceCpp("kernel_matrix_eigen1.cpp")
Rcpp::sourceCpp("kernel_matrix_eigen2.cpp")
Rcpp::sourceCpp("kernel_matrix_arma_para.cpp")
Rcpp::sourceCpp("kernel_matrix_eigen_para1.cpp")
Rcpp::sourceCpp("kernel_matrix_eigen_para2.cpp")

Sys.setenv("PKG_LIBS" = "$(LAPACK_LIBS) $(BLAS_LIBS) $(FLIBS)")
Rcpp::sourceCpp("kernel_matrix_eigen3.cpp")
Rcpp::sourceCpp("kernel_matrix_eigen_para3.cpp")
 
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
all.equal(kernelMatrix_eigen_cpp2(X, center, sigma), res_kernlab)    # TRUE
all.equal(kernelMatrix_arma_para_cpp(X, center, sigma), res_kernlab) # TRUE
all.equal(kernelMatrix_eigen_para_cpp(X, center, sigma), res_kernlab) # TRUE
all.equal(kernelMatrix_eigen_para_omp_cpp(X, center, sigma), res_kernlab) # TRUE
all.equal(kernelMatrix_eigen_para_cpp2(X, center, sigma), res_kernlab) # TRUE

library(microbenchmark)
microbenchmark(
  Rfun = kernelMatrix_R(X, center, sigma),
  kernlab = kernelMatrix(rbfdot(sigma=1/(2*sigma^2)), X, center),
  RcppArmadillo = kernelMatrix_arma_cpp(X, center, sigma),
  RcppEigen = kernelMatrix_eigen_cpp(X, center, sigma),
  RcppEigen_Openmp = kernelMatrix_eigen_omp_cpp(X, center, sigma),
  RcppEigen2 = kernelMatrix_eigen_cpp2(X, center, sigma),
  RcppArmadillo_RcppParallel = kernelMatrix_arma_cpp(X, center, sigma),
  RcppEigen_RcppParallel = kernelMatrix_eigen_para_cpp(X, center, sigma),
  RcppEigen_RcppParallel_Openmp = kernelMatrix_eigen_para_omp_cpp(X, center, sigma),
  RcppEigen_RcppParallel2 = kernelMatrix_eigen_para_cpp2(X, center, sigma),
  times = 30L
)
# Unit: milliseconds
#                           expr      min       lq     mean   median       uq      max neval
#                           Rfun 215.8470 241.8069 272.6916 264.2367 302.9716 335.1085    30
#                        kernlab 224.6712 245.8116 278.1123 264.4116 314.4725 351.3037    30
#                  RcppArmadillo 162.1123 167.2791 185.8838 172.7111 199.5971 257.8169    30
#                      RcppEigen 414.0345 416.6387 424.3981 418.9367 426.3183 489.0802    30
#               RcppEigen_Openmp 193.6153 195.3303 218.5779 200.1698 236.1647 313.2402    30
#                     RcppEigen2 136.9555 140.3265 148.6040 144.2964 150.6931 203.9994    30
#     RcppArmadillo_RcppParallel 164.3119 171.0287 187.1966 182.7915 197.8019 251.6530    30
#         RcppEigen_RcppParallel 408.2332 410.5831 415.1298 412.5009 415.5977 447.3581    30
#  RcppEigen_RcppParallel_Openmp 188.2270 191.7382 241.4736 237.4829 271.3958 337.3198    30
#        RcppEigen_RcppParallel2 129.1949 135.4055 151.7909 148.0432 160.7522 193.2683    30
```

kernel_matrix_eigen3.cpp:

``` c++
#define EIGEN_USE_BLAS
// [[Rcpp::depends(RcppEigen)]]
#include <RcppEigen.h>
using Eigen::MatrixXd;
using Eigen::VectorXd;
using Eigen::VectorXi;

// [[Rcpp::export]]
MatrixXd kernelMatrix_eigen_cpp2(MatrixXd x, MatrixXd center, double sigma) {
  MatrixXd kernelMat = x * center.transpose();
  VectorXd x_square_sum = x.array().pow(2.0).rowwise().sum() / 2.0;
  VectorXd center_square_sum = center.array().pow(2.0).rowwise().sum()  / 2.0;
  kernelMat.rowwise() -= center_square_sum.transpose();
  kernelMat.colwise() -= x_square_sum;
  kernelMat /=  pow(sigma, 2.0);
  return kernelMat.array().exp().matrix();
}
```

kernel_matrix_eigen_para3.cpp:

``` c++
#define EIGEN_USE_BLAS
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
MatrixXd kernelMatrix_eigen_para_cpp2(MatrixXd x, MatrixXd center, double sigma) {
  MatrixXd kernelMat = x * center.transpose();
  VectorXd x_square_sum = x.array().pow(2.0).rowwise().sum() / 2.0;
  VectorXd center_square_sum = center.array().pow(2.0).rowwise().sum()  / 2.0;
  
  double sigma2 = pow(sigma, 2.0);
  KernelComputeWorker worker_kc(x_square_sum, center_square_sum, sigma2, kernelMat);
  RcppParallel::parallelFor(0, kernelMat.size(), worker_kc);
  return kernelMat;
}
```

可以看到用了multi-threaded BLAS之後的Eigen更加強大

直接從原本的平均424.3981 ms表現衝到平均只要148.6040 ms

果然用對了武器就是不一樣啊(茶

至於`#define EIGEN_USE_LAPACKE`是無效的，畢竟R用的是`Lapack.h`不是`lapacke.h`(攤手

(LAPACK與LAPACKE的差異請看[stackoverflow](http://stackoverflow.com/questions/26875415/difference-between-lapacke-and-lapack)的解釋)

也許需要有人去更改Eigen的include file，讓Eigen能用RLAPACK的函數去處理~~
