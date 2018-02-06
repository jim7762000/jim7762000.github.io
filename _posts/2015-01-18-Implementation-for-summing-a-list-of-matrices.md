---
layout: post
title: Implementation for summing a list of matrices
---

Usually, we have to sum a list of matrices, I introduce several ways to do that.

In my test, `Reduce` is the most easy and fast to do this, but it need a lot of memory. A function written in `Rcpp` is the fastest way to do. But there is a requirement that you have learned C++. Besides above two methods, the naive method by using `for` is not so slow, it is also a good option.

``` R
# Aware that this setting require your memory to be large than 6GB. Otherwise, your computer will be out of memory and maybe shut down. If this happens or not enough memory, please change the `mat_size` or `length_list`.

length_list = 30
mat_size = c(3000, 3000)

mat = matrix(rnorm(prod(mat_size)), mat_size[1], mat_size[2])
mat.list = lapply(1:length_list, function(i) mat)

res1 = function(){
  mat_sum = matrix(rep(0, prod(mat_size)), mat_size[1], mat_size[2])
  for (i in 1:length_list){
    mat_sum = mat_sum + mat.list[[i]]
  }
  mat_sum
}
res2 = function() Reduce('+', mat.list)
res3 = function() matrix(rowSums(matrix(unlist(mat.list), nrow=prod(mat_size))), nrow=mat_size[1])
res4 = function() matrix(colSums(do.call(rbind, lapply(mat.list, as.vector))), nrow=mat_size[1])
res5 = function() matrix(rowSums(do.call(cbind, lapply(mat.list, as.vector))), nrow=mat_size[1])
library(Rcpp)
library(RcppArmadillo)
library(inline)
sourceCpp(code = '
// [[Rcpp::depends(RcppArmadillo)]]
#include <RcppArmadillo.h>
#include <stdio.h>
using namespace Rcpp;
using namespace arma;
// [[Rcpp::export]]
mat list_mat_sum_f(List data_list){
  int n = data_list.size();
  SEXP mat1 = data_list[0];
  NumericMatrix mat1_rmat(mat1);
  mat result_mat(mat1_rmat.begin(), mat1_rmat.nrow(), mat1_rmat.ncol());
  for(int i = 1; i < n; i++)
  {
    SEXP tmp_m = data_list[i];
    NumericMatrix data_m(tmp_m);
  mat working_m(data_m.begin(), data_m.nrow(), data_m.ncol(), false);
    result_mat += working_m;
  }
  return result_mat;
}')
res6 = function() list_mat_sum_f(mat.list)

# cmpfun
library(compiler)
res1_cmp = cmpfun(res1)
res2_cmp = cmpfun(res2)
res3_cmp = cmpfun(res3)
res4_cmp = cmpfun(res4)
res5_cmp = cmpfun(res5)

all.equal(tmp <- res1(), res2())
# TRUE
all.equal(tmp, res3())
# TRUE
all.equal(tmp, res4())
# TRUE
all.equal(tmp, res5())
# TRUE
all.equal(tmp, res6())
# TRUE

library(rbenchmark)
benchmark(res1(), res2(), res3(), res4(), res5(), res6(), res1_cmp(), res2_cmp(),
  res3_cmp(), res4_cmp(), res5_cmp(), replications = 20,  order='relative',
  columns=c('test', 'replications', 'elapsed','relative', 'user.self'))

#          test replications elapsed relative user.self
# 6      res6()           20    6.49    1.000      6.13
# 2      res2()           20   15.58    2.401      8.45
# 8  res2_cmp()           20   16.19    2.495      8.61
# 1      res1()           20   18.44    2.841     10.61
# 7  res1_cmp()           20   19.33    2.978     11.15
# 5      res5()           20   56.60    8.721     40.24
# 11 res5_cmp()           20   56.79    8.750     40.36
# 4      res4()           20   70.48   10.860     54.63
# 10 res4_cmp()           20   70.53   10.867     55.17
# 3      res3()           20   92.87   14.310     76.21
# 9  res3_cmp()           20   93.74   14.444     77.19


# length_list = 60
# mat_size = c(3000, 3000)
# benchmark(res2(), res6(), replications = 20, columns=c('test', 'replications', 'elapsed','relative', 'user.self'),  order='relative')
#
#     test replications elapsed relative user.self
# 2 res6()           20   12.02    1.000     11.73
# 1 res2()           20   31.66    2.634     16.44
```
