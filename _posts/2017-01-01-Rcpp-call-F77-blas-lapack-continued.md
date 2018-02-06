---
layout: post
title: "Rcpp call F77 blas/lapack (continued)"
---

想一想還是繼續把上一篇補完

所以這次就活用了LAPACK查詢，去得到the optimal sizes of the WORK array and IWORK array

R code沒什麼更動，只是多了一些check results的動作

也試試看第2,3個input得到的結果是否正確

R code:

``` R
Sys.setenv("PKG_LIBS" = "$(LAPACK_LIBS) $(BLAS_LIBS) $(FLIBS)")
Rcpp::sourceCpp("eigen_sym_cpp.cpp")

n_row <- 1e4L
n_col <- 1e1L
A <- matrix(rnorm(n_row * n_col), n_row, n_col)
S <- cov(A)

eig_res_cpp1 <- eigen_sym_cpp(S)
eig_res_cpp2 <- eigen_sym_cpp(S, 5L)
eig_res_cpp3 <- eigen_sym_cpp(S, -1L, TRUE)

eig_res_r <- eigen(S)

# check eigenvalues
all.equal(eig_res_cpp1$values, eig_res_r$values) # TRUE
all.equal(eig_res_cpp2$values, eig_res_r$values[1L:5L]) # TRUE
all.equal(eig_res_cpp3$values, eig_res_r$values) # TRUE

# check eigenvectors
sign_vectors_cpp1 <- colSums(eig_res_r$vectors * eig_res_cpp1$vectors)
eig_res_cpp1$vectors <- sweep(eig_res_cpp1$vectors, 2, sign_vectors_cpp1, '*')
all.equal(eig_res_cpp1$vectors, eig_res_r$vectors) # TRUE

# check eigenvectors
sign_vectors_cpp2 <- colSums(eig_res_r$vectors[ , 1L:5L] * eig_res_cpp2$vectors)
eig_res_cpp2$vectors <- sweep(eig_res_cpp2$vectors, 2, sign_vectors_cpp2, '*')
all.equal(eig_res_cpp2$vectors, eig_res_r$vectors[ , 1L:5L]) # TRUE

library(microbenchmark)
microbenchmark(
  Rcpp_BLAS = eigen_sym_cpp(S),
  R = eigen(S),
  times = 100L
)
# Unit: microseconds
#       expr      min       lq      mean    median       uq      max neval
#  Rcpp_BLAS  788.180  852.546  872.9262  860.4455  874.196 1000.293   100
#          R 1203.336 1271.505 1363.7922 1281.4520 1391.457 2432.417   100
```

C++程式部分採用了RcppEigen做一些輔助的動作

然後加上註解讓整個程式比較容易被讀懂

再來是，增加上面提到的查詢，然後再使用查詢結果去計算eigenvalues跟eigenvectors

並且簡化掉一些不會用到的參數，ex: `lda`, `ldz`, `iu`等

C++ code:

``` c++
// [[Rcpp::depends(RcppEigen)]]
#include <RcppEigen.h>
#include <R_ext/Lapack.h>
#include <R_ext/BLAS.h>

using Eigen::Map;
using Eigen::MatrixXd;
using Eigen::VectorXd;
using Eigen::VectorXi;

//[[Rcpp::export]]
Rcpp::List eigen_sym_cpp(Rcpp::NumericMatrix xr, int num_eig = -1, bool eigenvalues_only = false, double tol = 1.5e-8)
{
  MatrixXd A = MatrixXd::Map(xr.begin(), xr.nrow(), xr.ncol());
  
  // settings
  char jobz = eigenvalues_only ? 'N' : 'V', range = (num_eig == -1) ? 'A' : 'I', uplo = 'U';
  int N = A.rows(), info = 0;
  // deside the number of eigenvalues
  int il = (num_eig == -1) ? 1 : (N - num_eig + 1), M = N - il + 1;
  // the tolerance and not used arguments vl, vu
  double abstol = tol, vl = 0.0, vu = 0.0;
  // query the optimal size of the WORK array and IWORK array
  int lwork = -1, liwork = -1, iwork_query;
  
  VectorXi isuppz(2 * M);
  VectorXd W(M);
  MatrixXd Z(N, M);
  // perform dsyerv and get the optimal size of the WORK array and IWORK array
  double work_qeury;
  F77_CALL(dsyevr)(&jobz, &range, &uplo, &N, A.data(), &N, &vl, &vu, 
           &il, &N, &abstol, &M, W.data(), Z.data(), &N, isuppz.data(), 
           &work_qeury, &lwork, &iwork_query, &liwork, &info);
  
  // get the optimal sizeㄋ of the WORK array and IWORK array
  lwork = (int) work_qeury;
  liwork = iwork_query;
  VectorXd work(lwork);
  VectorXi iwork(liwork);
  // perform dsyerv and get the results of eigen decomposition
  F77_CALL(dsyevr)(&jobz, &range, &uplo, &N, A.data(), &N, &vl, &vu, 
           &il, &N, &abstol, &M, W.data(), Z.data(), &N, isuppz.data(), 
           work.data(), &lwork, iwork.data(), &liwork, &info);

  // reverse the eigenvalues to sort in the descending order
  W.reverseInPlace();
  
  // return eigenvalues only
  if (eigenvalues_only)
    return Rcpp::List::create(
      Rcpp::Named("LAPACK_info") = info,
      Rcpp::Named("values") = W
    );
  
  // reverse the eigenvectors to sort in the order of eigenvalues
  MatrixXd Z2 = Z.rowwise().reverse();
  
  // reutrn eigenvalues and eigenvectors
  return Rcpp::List::create(
    Rcpp::Named("LAPACK_info") = info,
    Rcpp::Named("vectors") = Z2,
    Rcpp::Named("values") = W
  );
}
```

這樣整個就算完工了~~
