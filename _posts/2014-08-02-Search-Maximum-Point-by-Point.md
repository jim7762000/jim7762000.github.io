---
layout: post
title: Search Maximum Point by Point
---

We have an image data, it is discrete and I want to find the local maximum. I write a Rcpp function for searching maximum point by point.

I have two strategies to search, one is in circle, another one is in square. Besides, I add a radius parameter to control the search region. Figure 1 shows that there are 34 by 34 points which each point contains a value of height, it searches maximum point by point with an unit circle. Figure 2, 3 and 4 show that searching maximum in circle with radius 2, square with radius 1 and square with radius 2, respectively.

![](/images/max_search_cr_1.png)
Figure 1: Searching in circle with radius 1
![](/images/max_search_cr_2.png)
Figure 2: Searching in circle with radius 2
![](/images/max_search_sq_1.png)
Figure 3: Searching in square with radius 1
![](/images/max_search_sq_2.png)
Figure 4: Searching in square with radius 2

I have three versions for searching maximum point by point. First one rely on Rcpp function completely, second one use sapply and Rcpp function which searching one point and third one use parallel and R function. Code is present as following:

``` R
x.range.v = 1:300
y.range.v = 1:300
data.m = expand.grid(x.range.v, y.range.v)
data.m[,3] = rnorm(nrow(data.m))

# square search
sq_search_f = function(i, data.m, loc.v, obj, mar){
  x = data.m[i,loc.v[1]]; y = data.m[i,loc.v[2]]
  near_int = data.m[abs(data.m[,loc.v[1]] - x) <= mar & abs(data.m[,loc.v[2]] - y) <= mar,obj]
  if( length(near_int) >= 1){
     all(near_int <= data.m[i,obj])
  }
  else{
     FALSE
  }
}

library(Rcpp)
sourceCpp(code = '
// [[Rcpp::depends(RcppArmadillo)]]
#include <RcppArmadillo.h>
using namespace Rcpp;
using namespace arma;

// [[Rcpp::export]]
uword sq_search_f_cpp_i_f(int i, NumericVector int_mr, NumericVector Xr, NumericVector Yr, double mar) {
  int n = Xr.size();
  colvec X(Xr.begin(), n, false),
         Y(Yr.begin(), n, false),
     int_m(int_mr.begin(), n, false);
  double x_loc = X( (uword) i), y_loc = Y( (uword) i);
  colvec temp_y = Y(find(abs(X - x_loc) <= mar)),
         temp_int = int_m(find(abs(X - x_loc) <= mar)),
         int_near = temp_int(find(abs(temp_y - y_loc) <= mar));
  uword value = (uword) all(int_near <= int_m((uword) i));
  return value;
}

// [[Rcpp::export]]
IntegerVector sq_search_cpp_f(NumericVector int_mr, NumericVector Xr, NumericVector Yr, double mar) {
  int n = Xr.size();
  ucolvec location_v(n);
  for(int i = 0; i < n; i++)
    location_v(i) = sq_search_f_cpp_i_f(i, int_mr, Xr, Yr, mar);
  return wrap(location_v);
}')

mar = 1
library(snowfall)
start.time = Sys.time()
loc1 = sq_search_cpp_f(data.m[,3], data.m[,1], data.m[,2], mar)
Sys.time() - start.time
# Time difference of 20.32596 secs

start.time = Sys.time()
loc2 = sapply(1:nrow(data.m), function(i) sq_search_f_cpp_i_f(i-1, data.m[,3], data.m[,1], data.m[,2], mar))
Sys.time() - start.time
# Time difference of 1.229692 mins

sfInit(parallel = TRUE, cpus = 8)
sfExport("data.m", "sq_search_f", "mar")
start.time = Sys.time()
loc3 = sfSapply(1:nrow(data.m), function(i) sq_search_f(i, data.m, c(1,2), 3, mar))
Sys.time() - start.time
sfStop()
# Time difference of 1.530629 mins

all.equal(as.vector(loc1), loc2)
# TRUE
all.equal(as.vector(loc1), as.numeric(loc3))
# TRUE

# circle search
cr_search_f = function(i, data.m, loc.v, obj, mar){
  x = data.m[i,loc.v[1]]; y = data.m[i,loc.v[2]]
  near_int = data.m[(data.m[,loc.v[1]] - x)^2+ (data.m[,loc.v[2]] - y)^2 <= mar,obj]
  if( length(near_int) >= 1){
     all(near_int <= data.m[i,obj])
  }
  else{
     FALSE
  }
}

library(Rcpp)
sourceCpp(code = '
// [[Rcpp::depends(RcppArmadillo)]]
#include <RcppArmadillo.h>
using namespace Rcpp;
using namespace arma;

// [[Rcpp::export]]
uword cr_search_f_cpp_i_f(int i, NumericVector int_mr, NumericVector Xr, NumericVector Yr, double mar) {
  int max_threads_mkl = mkl_get_max_threads();
  mkl_set_num_threads(max_threads_mkl);
  int n = Xr.size();
  colvec X(Xr.begin(), n, false),
         Y(Yr.begin(), n, false),
     int_m(int_mr.begin(), n, false);
  double x_loc = X( (uword) i), y_loc = Y( (uword) i);
  colvec int_near = int_m(find(square(X - x_loc) + square(Y - y_loc) <= mar));
  uword value = (uword) all(int_near <= int_m((uword) i));
  return value;
}

// [[Rcpp::export]]
IntegerVector cr_search_cpp_f(NumericVector int_mr, NumericVector Xr, NumericVector Yr, double mar) {
  int n = Xr.size();
  ucolvec location_v(n);
  for(int i = 0; i < n; i++)
    location_v(i) = cr_search_f_cpp_i_f(i, int_mr, Xr, Yr, mar);
  return wrap(location_v);
}')

mar = 1
library(snowfall)
start.time = Sys.time()
loc1 = cr_search_cpp_f(data.m[,3], data.m[,1], data.m[,2], mar)
Sys.time() - start.time
# Time difference of 19.21388 secs

start.time = Sys.time()
loc2 = sapply(1:nrow(data.m), function(i) cr_search_f_cpp_i_f(i-1, data.m[,3], data.m[,1], data.m[,2], mar))
Sys.time() - start.time
# Time difference of 47.72535 secs

sfInit(parallel = TRUE, cpus = 8)
sfExport("data.m", "cr_search_f", "mar")
start.time = Sys.time()
loc3 = sfSapply(1:nrow(data.m), function(i) cr_search_f(i, data.m, c(1,2), 3, mar))
Sys.time() - start.time
sfStop()
# Time difference of 1.055895 mins

all.equal(as.vector(loc1), loc2)
# TRUE
all.equal(as.vector(loc1), as.numeric(loc3))
# TRUE
```

The above result shows that sapply is not fast enough, write Rcpp function if you can! The speed of paralleled R function (8 threads) is close to Rcpp function with sapply (one threads), the for loop in R is so slow.

My environment is ubuntu 14.04, R 3.1.1 compiled by Intel c++, fortran compiler with MKL. My CPU is 3770K@4.3GHz.

