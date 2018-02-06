---
layout: post
title: "RcppArmadillo Uses NVBLAS"
---

Set an environmental variables for CUDA_HOME.

For Linux:

``` bash
export CUDA_HOME=/path/to/cuda/home
tee -a `Rscript -e "R.home()"`/etc/Renviron << EOF
export LD_PRELOAD=$CUDA_HOME/lib64/libnvblas.so
EOF

tee -a `Rscript -e "R.home()"`/etc/Rprofile.site << EOF
Sys.setenv(PKG_CXXFLAgs = sprintf("-I%s/include", Sys.getenv("CUDA_HOME")))
Sys.setenv(PKG_LIBS = sprintf("-L%s/lib64 -lnvblas", Sys.getenv("CUDA_HOME")))
EOF
```

For Windows:

``` bash
tee -a `Rscript -e "R.home()"`/etc/Rprofile.site << EOF
dyn.load(paste0(Sys.getenv("CUDA_HOME"), "/bin/nvblas64_90.dll"))
Sys.setenv(PKG_CXXFLAGS = sprintf("-I\"%s/include\"", Sys.getenv("CUDA_HOME")))
Sys.setenv(PKG_LIBS = sprintf("-L\"%s/lib64\" -lnvblas", Sys.getenv("CUDA_HOME")))
EOF
```


Run R:

``` R
# NVBLAS_CONFIG_FILE
write(c("NVBLAS_LOGFILE nvblas.log", "NVBLAS_CPU_BLAS_LIB Rblas.dll", 
        "NVBLAS_GPU_LIST ALL"), "nvblas.conf")
		
# library Rcpp and RcppArmadillo
library(Rcpp)
library(RcppArmadillo)

# Rcpp code
code <- '
// [[Rcpp::depends(RcppArmadillo)]]
#include <RcppArmadillo.h>
using Rcpp::_;
using Rcpp::List;
// [[Rcpp::export]]
Rcpp::List arma_fastLm_chol(const arma::mat& X, const arma::vec& y) {
  arma::mat xtx = X.t() * X;
  arma::vec coef = arma::solve(xtx, X.t() * y);
  arma::vec res  = y - X*coef;
  arma::uword df = X.n_rows - X.n_cols;
  double s2 = arma::dot(res, res) / (double) df;
  arma::colvec se = arma::sqrt(s2 * arma::diagvec(arma::inv_sympd(xtx)));
  return List::create(_["coefficients"] = coef, _["stderr"] = se, 
                      _["df.residual"]  = df);
}'
# compile Rcpp code
Rcpp::sourceCpp(code = code)

# data generation
n <- 2e5L
p <- 300L
mm <- cbind(1, matrix(rnorm(n * (p - 1L)), nc = p-1L))
y <- mm %*% rnorm(p, sd = 3) + rnorm(n, sd = 5)

# test function
system.time({
  arma_fastLm_chol(mm, y)
})
## cpu
# user  system elapsed
# 7.03    0.22    0.45
## gpu
# user  system elapsed
# 0.60    0.25    0.38
```

My computer: AMD Threadripper 1950X with 128GB Ram and GTX 1080 SLI
