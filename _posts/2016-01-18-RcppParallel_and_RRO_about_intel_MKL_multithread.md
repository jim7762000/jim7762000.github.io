---
layout: post
title: "RcppParallel and RRO: about Intel MKL multithread"
---

Revolution R Open (Now named Microsoft R Open) provides Intel MKL multi-threaded BLAS.
RcppParallel provides parallel tools for R via Intel TBB. (Note: On other platforms, it use TinyThread library.)
However, does RcppParallel conflict with Intel MKL multi-threaded BLAS?

We do a simple test on this issue. The cross-validation is simple but time-consuming which is a great method for testing the power of parallel.
Therefore, we use leave-one-out-cross-validation to compute the mean squared errors of regression model.
Leave-one-out-cross-validation is a important tool for measuring the performance of model.
We use Rcpp, RcppArmadillo and RcppParallel to Leave-one-out-cross-validation. You can find the code of test below.
Results:

  expr (Unit: seconds)  |       min  |        lq  |      mean  |    median  |        uq  |       max  | neval
-----------------------:|-----------:|-----------:|-----------:|-----------:|-----------:|-----------:|-------
         LOOCV_R(y, X)  | 44.094387  | 44.293443  | 45.217615  | 44.712347  | 45.149469  | 51.166242  |    20
-----------------------:|-----------:|-----------:|-----------:|-----------:|-----------:|-----------:|-------
  fastLOOCV_noRP(y, X)  | 26.710105  | 26.811101  | 26.869819  | 26.854140  | 26.905504  | 27.094317  |    20
-----------------------:|-----------:|-----------:|-----------:|-----------:|-----------:|-----------:|-------
       fastLOOCV(y, X)  |  6.411246  |  6.494743  |  6.720013  |  6.574210  |  6.697720  |  8.210375  |    20
-----------------------:|-----------:|-----------:|-----------:|-----------:|-----------:|-----------:|-------
 fastLOOCV_mkl_1(y, X)  |  6.527082  |  6.997851  |  7.099617  |  7.148261  |  7.327877  |  7.465877  |    20
-----------------------:|-----------:|-----------:|-----------:|-----------:|-----------:|-----------:|-------
 fastLOOCV_mkl_2(y, X)  |  6.364546  |  6.529459  |  6.564495  |  6.557765  |  6.637208  |  6.668344  |    20

According to our test, the performance of RcppArmadillo is about 1.6x as fast as R.
For our concern, the performance of using default number of MKL threads (12) is very close to using 2 MKL threads,
and is better than using 1 MKL threads. From this test, we think that the performance of RcppParallel is
deeply affected the number of threads Intel MKL used.
We suggest not to change the number of threads Intel MKL used, just keep it default.

``` R
library(Rcpp)
library(RcppArmadillo)
library(RcppParallel)

sourceCpp(code = '
// [[Rcpp::plugins(cpp11)]]
// [[Rcpp::depends(RcppArmadillo, RcppParallel)]]
#include <RcppArmadillo.h>
#include <RcppParallel.h>
using namespace RcppParallel;
using namespace arma;
using namespace Rcpp;

double fastMSE_RcppArma(vec y, mat X, vec yTest, mat XTest)
{
 vec coef = solve(join_rows(ones<vec>(y.n_elem), X), y);
 return as_scalar(pow(yTest - join_rows(ones<vec>(yTest.n_elem), XTest)*coef, 2));
}

struct MSE_Compute: public Worker {
 mat& X;
 vec& y;
 uvec& index;
 vec& mse;
 MSE_Compute(mat& X, vec& y, uvec& index, vec& mse):
   X(X), y(y), index(index), mse(mse) {}
 void operator()(std::size_t begin, std::size_t end)
 {
   for (uword row_index = begin; row_index < end; row_index++)
   {
     mse(row_index) = fastMSE_RcppArma(y.elem(find(index != row_index)),
       X.rows(find(index != row_index)), y.elem(find(index == row_index)),
       X.rows(find(index == row_index)));
   }
 }
};

// [[Rcpp::export]]
NumericVector fastLOOCV(NumericVector yr, NumericMatrix Xr)
{
 int n = Xr.nrow(), p = Xr.ncol();
 mat X(Xr.begin(), n, p, false);
 if (yr.size() != n)
   Rcpp::stop("The size of y must be equal to the number of rows of X." );
 vec y(yr.begin(), n, false),
 mse = zeros<vec>(n);
 uvec index = linspace<uvec>(0, y.n_elem - 1, y.n_elem);
 MSE_Compute mseResults(X, y, index, mse);
 parallelFor(0, n, mseResults);
 return wrap(mse);
}

// [[Rcpp::export]]
NumericVector fastLOOCV_noRP(NumericVector yr, NumericMatrix Xr)
{
 int n = Xr.nrow(), p = Xr.ncol();
 mat X(Xr.begin(), n, p, false);
 if (yr.size() != n)
   Rcpp::stop("The size of y must be equal to the number of rows of X." );
 vec y(yr.begin(), n, false),
 mse = zeros<vec>(n);
 uvec index = linspace<uvec>(0, y.n_elem - 1, y.n_elem);
 for (uword row_index = 0; row_index < y.n_elem; row_index++)
 {
   mse(row_index) = fastMSE_RcppArma(y.elem(find(index != row_index)),
       X.rows(find(index != row_index)), y.elem(find(index == row_index)),
       X.rows(find(index == row_index)));
 }
 return wrap(mse);
}')

start_RppParallel <- function(verbose=FALSE, numMKLthreads = 1){
  if (!identical(system.file(package="RevoUtilsMath"), ""))
  {
    RevoUtilsMath::setMKLthreads(numMKLthreads)
    RcppParallel::setThreadOptions(parallel::detectCores(FALSE, TRUE)/numMKLthreads)
    if (verbose)
    {
      cat(sprintf('RcppParallel uses %i threads.\n', parallel::detectCores(FALSE, FALSE)))
      cat('RevoUtilsMath is installed with multi-threaded BLAS, change the number of MKL threads.\n')
      cat(sprintf('Multithreaded BLAS/LAPACK libraries detected. Using %s cores for math algorithms.\n', RevoUtilsMath::getMKLthreads()))
    }
  }
}

end_RppParallel <- function(verbose=FALSE){
  if (!identical(system.file(package="RevoUtilsMath"), ""))
  {
    RevoUtilsMath::setMKLthreads(parallel::detectCores(FALSE, FALSE))
    if (verbose)
    {
      cat('RevoUtilsMath is installed with multi-threaded BLAS, recover the number of MKL threads.\n')
      cat(sprintf('Multithreaded BLAS/LAPACK libraries detected. Using %s cores for math algorithms.\n', RevoUtilsMath::getMKLthreads()))
    }
  }
}

fastLOOCV_mkl_1 <- function(y, X, verbose = FALSE){
  start_RppParallel(verbose, 1)
  mses <- fastLOOCV(y, X)
  end_RppParallel(verbose)
  return(mses)
}

fastLOOCV_mkl_2 <- function(y, X, verbose = FALSE){
  start_RppParallel(verbose, 2)
  mses <- fastLOOCV(y, X)
  end_RppParallel(verbose)
  return(mses)
}

LOOCV_R <- function(y, X){
  mses <- vector('numeric', length(y))
  for (i in seq_along(y))
  {
    lmCoefs <- coef(lm(y[-i] ~ X[-i, ]))
    mses[i] <- mean((y[i] - c(1, X[i, ]) %*% lmCoefs)**2)
  }
  return(mses)
}

set.seed(10)
N <- 500
p <- N*0.95
X <- matrix(rnorm(N*p), N)
lm_coef <- replicate(p, rnorm(1, rnorm(1, 2, 3), rgamma(1,10,2)))
addingCol = matrix(sample(p, p, TRUE), p/2, 2)
addingCol = addingCol[addingCol[ ,1] != addingCol[ ,2], ]
for (i in 1:ncol(addingCol))
  X[ , addingCol[i,1]] = X[ , addingCol[i,1]] + X[ , addingCol[i,2]]
y <- 9 + rowSums(sweep(X, 2, lm_coef, '*')) + rnorm(N, 0, 10)

st <- proc.time()
mses_R <- LOOCV_R(y, X)
proc.time() - st
#   user  system elapsed
#  39.61    0.92   43.97

st <- proc.time()
mses_cpp1 <- fastLOOCV_noRP(y, X)
proc.time() - st
#   user  system elapsed
#  38.88    6.54   27.02
all.equal(mses_R, as.vector(mses_cpp1)) # TRUE

st <- proc.time()
mses_cpp2 <- fastLOOCV(y, X)
proc.time() - st
#   user  system elapsed
#  52.77    7.91    6.52
all.equal(mses_R, as.vector(mses_cpp2)) # TRUE

st <- proc.time()
mses_cpp3 <- fastLOOCV_mkl_1(y, X)
proc.time() - st
#   user  system elapsed
#  63.21   12.31    7.18
all.equal(mses_R, as.vector(mses_cpp3)) # TRUE

st <- proc.time()
mses_cpp4 <- fastLOOCV_mkl_2(y, X)
proc.time() - st
#   user  system elapsed
#  53.32    7.62    6.47
all.equal(mses_R, as.vector(mses_cpp4)) # TRUE

library(microbenchmark)
microbenchmark(LOOCV_R(y, X), fastLOOCV_noRP(y, X), fastLOOCV(y, X),
               fastLOOCV_mkl_1(y, X), fastLOOCV_mkl_2(y, X), times = 20L)
# Unit: seconds
#                   expr       min        lq      mean    median        uq       max neval
#          LOOCV_R(y, X) 44.094387 44.293443 45.217615 44.712347 45.149469 51.166242    20
#   fastLOOCV_noRP(y, X) 26.710105 26.811101 26.869819 26.854140 26.905504 27.094317    20
#        fastLOOCV(y, X)  6.411246  6.494743  6.720013  6.574210  6.697720  8.210375    20
#  fastLOOCV_mkl_1(y, X)  6.527082  6.997851  7.099617  7.148261  7.327877  7.465877    20
#  fastLOOCV_mkl_2(y, X)  6.364546  6.529459  6.564495  6.557765  6.637208  6.668344    20

# The information of machine: i7-5820K@4.0GHz with 32GB ram.
# The information of session: RRO 3.2.3 on windows 7 64bit.
```
