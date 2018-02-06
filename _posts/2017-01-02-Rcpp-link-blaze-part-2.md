---
layout: post
title: "Rcpp Link Blaze (part 2)"
---

如果include `blaze/Blaze.h`會發現出了好幾個錯誤，這裡提供一些解法：

1. 改`blaze/util/Time.h`

原本的43行~55行改成下面這樣：

``` c++
#if defined(_MSC_VER)
#  ifndef NOMINMAX
#    define NOMINMAX
#  endif
#  include <windows.h>
#  include <winsock.h>
#  include <time.h>
#  include <sys/timeb.h>
#else
#if defined(__MINGW32__)
#  include <windows.h>
#  include <winsock.h>
#  include <time.h>
#  include <sys/timeb.h>
#else
#  include <sys/resource.h>
#  include <sys/time.h>
#  include <sys/types.h>
#endif
#endif
```

2. 在pre-processor中efine 三個變數

``` c++
#define STRICT_R_HEADERS
#define BOOST_ERROR_CODE_HEADER_ONLY
#define RCPP_HAS_LONG_LONG_TYPES
```

第一個只是避免掉C++11的編譯錯誤，第二個則是避免去linking boost::system

`RCPP_HAS_LONG_LONG_TYPES`是為了讓Blaze可以輸出`unsigned long long int`的維度

3. 不要使用`boost::thread`

可以在pre-processor中加入下面幾行，提醒自己：

``` c++
#if defined(BLAZE_USE_BOOST_THREADS)
#error "Boost threads could not be used!"
#endif
```

設定完之後，我們測試看看：

``` R
Sys.setenv("PKG_CXXFLAGS" = sprintf('-I"%s"', normalizePath(".", winslash = "/")))
Rcpp::sourceCpp(code = '
#define STRICT_R_HEADERS
#define BOOST_ERROR_CODE_HEADER_ONLY
// [[Rcpp::depends(BH)]]
// [[Rcpp::plugins(cpp11)]]
#include <Rcpp.h>
#include <blaze/Blaze.h>

using blaze::unaligned; 
using blaze::unpadded;
using blaze::columnMajor;
using blaze::CustomMatrix;
using blaze::CustomVector;
using blaze::DynamicVector;

//[[Rcpp::export]]
Rcpp::NumericVector blaze_mv(Rcpp::NumericMatrix X, Rcpp::NumericVector Y) {
  // input cheching
  if (X.ncol() != Y.size())
    Rcpp::stop("The size of Y must be equal to the number of columns of X.");
 
  // map and perform matrix-vector multiplication
  CustomMatrix<double, unaligned, unpadded, columnMajor> a(&X[0], X.nrow(), X.ncol());
  CustomVector<double, unaligned, unpadded> b(&Y[0], Y.size());
  DynamicVector<double> c = a * b;

  // output
  Rcpp::NumericVector out(c.size());
  for (auto i = 0; i < c.size(); ++i) out[i] = c[i];
  return out;
}')

library(microbenchmark)
N <- 5000L
X <- matrix(rnorm(N**2), N, N)
y <- rnorm(N)
microbenchmark(
  blaze_mv = blaze_mv(X, y),
      R_mv = as.vector(X %*% y),
     times = 50L
)
# Unit: milliseconds
#      expr      min       lq     mean   median       uq      max neval
#  blaze_mv 10.09042 10.32564 11.70235 10.93375 12.88094 15.67995    50
#      R_mv 32.24726 33.52491 35.97026 35.66461 37.98600 42.05827    50
```

矩陣乘法快了三倍呢！
