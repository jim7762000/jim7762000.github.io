---
layout: post
title: "Rcpp Link Blaze"
---

補充：我把程式放到GitHub，可以到這下載來玩：[Link](https://github.com/ChingChuan-Chen/rcpp_link_blaze)

Blaze claim it is fast. [Link](https://bitbucket.org/blaze-lib/blaze/wiki/Benchmarks)

所以我就來試試看Rcpp去link玩玩看

看看有沒有大神會把它弄成一個套件，叫做RcppBlaze之類

我試了一下blaze-3.0，怎樣都無法編譯Orz (後來有找到要用`-std=c++1y`才能編譯成功)

我就改用上一版的blaze-2.6，不過還是有些地方需要更動

要變動的地方有兩個，一個是`blaze/util/Memory.h`，另一個是`blaze/math/adaptors/hermitianmatrix/HermitianValue.h`

Memory.h:

``` c++
// 在57行增加下面幾行
#ifdef __MINGW32__ 
#define _aligned_malloc __mingw_aligned_malloc 
#define _aligned_free  __mingw_aligned_free 
#endif

// 84行跟112行原本是#if defined(_MSC_VER)改成：
#if defined(_MSC_VER) || defined(__MINGW32__)
```

HermitianValue.h: (blaze-3.0不用改)

``` c++
// 279, 299, 323, 347, 371, 395, 513行原本是 pos_->index() == index改成：
pos_->index() == index_
```

然後就可以快樂的開始玩blaze了

(Note: 其實還有一些要改才能完整使用Blaze，請參考[下一篇文章](http://chingchuan-chen.github.io/posts/2017/01/02/Rcpp-link-blaze-part-2))

R code:

``` R
useBlaze26 <- FALSE
if (useBlaze26 && dir.exists("blaze26")) {
  file.rename("blaze", "blaze3")
  file.rename("blaze26", "blaze")
  Sys.setenv("PKG_CXXFLAGS" = '-I.')
} else if (!useBlaze26 && dir.exists("blaze3")) {
  file.rename("blaze", "blaze26")
  file.rename("blaze3", "blaze")
  Sys.setenv("PKG_CXXFLAGS" = "-I. -DUSE_BLAZE3")
} else {
  stop("Please use correct tag!")
}
Rcpp::sourceCpp("test_blaze.cpp", rebuild = TRUE)

test_blaze1(1:5)
test_blaze1(rnorm(5))

test_blaze2(matrix(1:9, 3, 3))
test_blaze2(matrix(rnorm(9), 3, 3))

X <- matrix(rnorm(9), 3, 3)
y <- rnorm(3)
all.equal(test_blaze3(X, y), as.vector(X %*% y)) # TRUE

library(microbenchmark)
M <- 6e3L
N <- 3e4L
X <- matrix(rnorm(M * N), M, N)
y <- rnorm(N)
microbenchmark(
  blaze = test_blaze3(X, y),
  R = as.vector(X %*% y),
  times = 50L
)
# Unit: microseconds
#   expr     min      lq     mean  median      uq      max neval
#  blaze 168.228 169.983 195.6315 173.640 202.750  312.172    30
#      R 554.126 559.099 632.9047 562.757 660.328 1506.730    30
```

test_blaze.cpp:

``` c++
// [[Rcpp::depends(BH)]]
#ifdef USE_BLAZE3
// [[Rcpp::plugins(cpp14)]]
#else
// [[Rcpp::plugins(cpp11)]]
#endif
#include <Rcpp.h>
#include <blaze/Math.h>

//[[Rcpp::export]]
Rcpp::NumericVector test_blaze1(Rcpp::NumericVector X){
  blaze::CustomVector<double, blaze::unaligned, blaze::unpadded> v( &X[0], X.size() );
  Rcpp::Rcout << v[0] << std::endl;
  Rcpp::Rcout << v[1] << std::endl;
  Rcpp::Rcout << v[2] << std::endl;
  Rcpp::Rcout << v[3] << std::endl;
  Rcpp::Rcout << v[4] << std::endl;
  return X;
}

//[[Rcpp::export]]
Rcpp::NumericMatrix test_blaze2(Rcpp::NumericMatrix X){
  blaze::CustomMatrix<double, blaze::unaligned, blaze::unpadded, blaze::columnMajor> v( &X[0], X.nrow(), X.ncol() );
  Rcpp::Rcout << v(0, 0) << std::endl;
  Rcpp::Rcout << v(0, 1) << std::endl;
  Rcpp::Rcout << v(0, 2) << std::endl;
  Rcpp::Rcout << v(1, 0) << std::endl;
  Rcpp::Rcout << v(1, 1) << std::endl;
  Rcpp::Rcout << v(1, 2) << std::endl;
  return X;
}

//[[Rcpp::export]]
Rcpp::NumericVector test_blaze3(Rcpp::NumericMatrix X, Rcpp::NumericVector y){
  blaze::CustomMatrix<double, blaze::unaligned, blaze::unpadded, blaze::columnMajor> a( &X[0], X.nrow(), X.ncol() );
  blaze::CustomVector<double, blaze::unaligned, blaze::unpadded> b( &y[0], y.size() );
  
  blaze::DynamicVector<double> c = a * b;
  Rcpp::NumericVector out(c.size());
  for (auto i = 0; i < c.size(); ++i) 
    out[i] = c[i];
  return out;
}
```

後記：

blaze-3.0的話，其實只要改好`blaze/util/Memory.h`

然後在cpp file中使用`// [[Rcpp::plugins(cpp14)]]`即可 

Note: 其實還有`cpp1y` for C++14 and C++17 standard under development

Rcpp的plugins可以看[GitHub Rcpp Issue Link](https://github.com/RcppCore/Rcpp/issues/524)

我這裡測試是blaze-3.0跟blaze-2.6效能滿接近的

個人是覺得因為R的Policy問題，所以還是用2.6就好

用太新的C++ std通常都會吃土
