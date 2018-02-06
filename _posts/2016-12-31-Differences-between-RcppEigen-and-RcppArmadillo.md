---
layout: post
title: "Differences between RcppEigen and RcppArmadillo"
---

R maillist有討論過這個問題，而且回的非常長

我稍微看了一下，發問者試了RcppArmadillo跟RcppEigen

發現RcppEigen的SVD decomposition不是那麼合用，而且不太好去做coding

RcppArmadillo比較好去coding


接下來的回文就是RcppEigen會快一點(我的文章也證實了這點)

而RcppEigen是用自己的BLAS and LAPACK，RcppArmadillo會使用R session所使用的

然後接下來就跳到RcppArmadillo的svd表現不好的問題：

``` R
library(inline)
library(RcppArmadillo)
library(RcppEigen)

arma_body <- 'using namespace arma; NumericMatrix Xr(Xs); 
mat X = Rcpp::as<mat>(Xr), U, V; 
vec s; svd(U, s, V, X); 
return wrap(s);'
arma_svd <- cxxfunction(signature(Xs = "numeric"), body = arma_body, plugin = "RcppArmadillo")
eigen_body <- 'using Eigen::MatrixXd; using Eigen::Map;
Map<MatrixXd> x(Rcpp::as<Map<MatrixXd> >(Xs));
Eigen::JacobiSVD<MatrixXd> svd(x, Eigen::ComputeThinU | Eigen::ComputeThinV);
return wrap(svd.singularValues());'
eigen_svd <- cxxfunction(signature(Xs = "numeric"), body = eigen_body, plugin = "RcppEigen")

library(microbenchmark)
N <- 1000L
A <- matrix(rnorm(N^2), N)
microbenchmark(svd(A), arma_svd(A), eigen_svd(A), times = 20L)
# Unit: milliseconds
#          expr        min         lq       mean     median         uq        max neval
#        svd(A)   436.0527   441.6736   444.1783   443.7081   445.4228   454.5612    10
#   arma_svd(A)   354.7630   359.3738   365.7064   365.5194   367.7162   383.8402    10
#  eigen_svd(A) 56648.3330 56700.3132 57499.2423 57345.0418 58318.3001 58883.5604    10
```

我在我電腦(i7-3770K@4.2GHz with MRO 3.3.1)上測試

其實RcppArmadillo是最快的

不知道是不是Armadillo做了一些改進

因為討論中提到R只有算sigular values沒算U跟V，而Armadillo有

這造就了底層的LAPACK用的函數就不同了，所以performance會差很多

而RcppEigen因為用自帶的LAPACK比我R用的MKL慢不少，所以performance很差


再來，我也測試看看兩個套件中fastLmPure的表現

``` R
n <- 3e3
p <- 250
X <- matrix(rnorm(n * p), n , p)
beta <- rnorm(p)
y <- 3 + X %*% beta + rnorm(n)

library(microbenchmark)
microbenchmark(
eigen_colPivQR = RcppEigen::fastLmPure(X, y, method = 0L),
    eigen_HHQR = RcppEigen::fastLmPure(X, y, method = 1L),
     eigen_LLT = RcppEigen::fastLmPure(X, y, method = 2L),
    eigen_LLDT = RcppEigen::fastLmPure(X, y, method = 3L),
     eigen_SVD = RcppEigen::fastLmPure(X, y, method = 4L),
     eigen_eig = RcppEigen::fastLmPure(X, y, method = 5L),
     armadillo = RcppArmadillo::fastLmPure(X, y),
         times = 30L
)
# Unit: milliseconds
#            expr        min         lq       mean     median         uq        max neval
#  eigen_colPivQR   77.37665   78.46618   85.91465   82.41118   89.88866  120.38286    30
#      eigen_HHQR   47.47491   47.83243   52.40245   48.70604   54.42752   70.18852    30
#       eigen_LLT   17.68025   18.07902   37.30092   18.50793   21.12671  544.98383    30
#      eigen_LDLT   19.52899   19.95702   22.62692   20.12774   26.57289   36.44651    30
#       eigen_SVD 1019.39032 1051.11184 1072.05842 1061.67417 1080.98286 1201.88356    30
#       eigen_eig   62.97932   63.51531   71.18810   64.34445   79.16513   99.23830    30
#            arma   47.26046   48.48105   50.61176   49.99042   51.71892   57.41407    30
```

這裡也可以看出來RcppArmadillo的performace平平，RcppEigen的llt, ldlt速度真的很快，比RcppArmadillo快上不少

可是這裡沒辦法加weights，所以我也測試了一下WLS，RcppArmadillo就快上不少，可以看一下我今天發的其他篇文章


看了兩個Benchmark之後，總結一下：

RcppEigen有自己的LAPACK，在某些case下可以比RcppArmadillo快

而SVD方面的表現卻是不盡人意，非常的慢

再來是連結到我的文章，WLS的計算上RcppArmadillo快([連結](http://chingchuan-chen.github.io/posts/2016/12/31/weighted-least-square-algorithm)，LOOCV with RcppParallel/OpenMP則是RcppEigen快([連結](http://chingchuan-chen.github.io/posts/2016/12/31/RcppEigen-Work-With-RcppParall)

所以求速度的話，使用RcppEigen基本上沒錯，只是svd的performance真的很差，這部分需要謹記(其他雷目前未知)

但是RcppArmadillo有不少方便的功能

submatrix view可以支援用uword vector拉出

element-wise, column/row-wise的操作也相較起RcppEigen來的強大 

舉例來說，`mat m = p.each_col() % v`，其中`v`是vector，`m`, `p`都是矩陣

而這個在RcppEigen就要用迴圈來處理了，但是performance還是RcppEigen快一些

所以求coding方便性，希望有很多簡單的方式去處理資料，使用RcppArmadillo就好


延伸閱讀：

1. [kernal matrix computation in Rcpp](http://chingchuan-chen.github.io/posts/2017/01/01/kernal-matrix-computation-in-Rcpp)
1. [kernal matrix computation in Rcpp (continued)](http://chingchuan-chen.github.io/posts/2017/01/01/kernal-matrix-computation-in-Rcpp-continued)
1. [pdate RcppEigen to 3.3.1](http://chingchuan-chen.github.io/posts/2017/01/02/update-RcppEigen-to-3-3-1)
