---
layout: post
title: R data.table - sum by groups
---

`data.table` is a powerful tool for exploring data. However, how is it fast? Here we provides a performance test for summing by groups.

Code:

``` R
N = 1e4
x = data.frame(Freq=runif(N,0,1),Category=c("T","F"))

library(data.table)
library(plyr)
library(dplyr)
x_dt = data.table(x)
setkey(x_dt, Category)
Cate_group_dt = group_by(x_dt, Category)
Cate_group_df = group_by(x_dt, Category)
library(rbenchmark)
benchmark(data_table = x_dt[, sum(Freq),by = Category],
          tapply = tapply(x$Freq, x$Category, FUN=sum),
          plyr_dt = aggregate(Freq ~ Category, data = x_dt, FUN=sum),
          plyr_dt2 = ddply(x_dt, .(Category), colwise(sum)),
          plyr_df = aggregate(Freq ~ Category, data = x, FUN=sum),
          plyr_df2 = ddply(x, .(Category), colwise(sum)),
          dplyr_dt = summarise(Cate_group_dt, sum(Freq)),
          dplyr_df = summarise(Cate_group_df, sum(Freq)),
          replications = 20,
          columns=c('test', 'replications', 'elapsed','relative', 'user.self'),
          order='relative')

# Result for N = 1e4:
#         test replications elapsed relative user.self
# 2     tapply          100   0.082    1.000     0.082
# 1 data_table          100   0.143    1.744     0.142
# 8   dplyr_df          100   0.186    2.268     0.186
# 7   dplyr_dt          100   0.188    2.293     0.187
# 6   plyr_df2          100   0.312    3.805     0.312
# 4   plyr_dt2          100   0.320    3.902     0.320
# 3    plyr_dt          100   2.738   33.390     2.739
# 5    plyr_df          100   3.887   47.402     3.888

# Result for N = 1e6:
#         test replications elapsed relative user.self
# 1 data_table           20   0.613    1.000     0.608
# 7   dplyr_dt           20   0.631    1.029     0.626
# 8   dplyr_df           20   0.636    1.038     0.630
# 2     tapply           20   1.179    1.923     1.168
# 6   plyr_df2           20   1.445    2.357     1.419
# 4   plyr_dt2           20   1.475    2.406     1.451
# 3    plyr_dt           20  66.165  107.936    66.162
# 5    plyr_df           20  98.394  160.512    98.416

# Result for N = 1e7:
#         test replications elapsed relative user.self
# 6   dplyr_df           20   5.939    1.000     5.823
# 5   dplyr_dt           20   5.954    1.003     5.840
# 1 data_table           20   5.980    1.007     5.847
# 2     tapply           20  12.080    2.034    11.649
# 3   plyr_dt2           20  14.759    2.485    13.813
# 4   plyr_df2           20  14.857    2.502    13.809
```

In the case with small sample size, `tapply` is the most efficient tool for summing by groups. In the case with large sample size, `data.table` and `summarise` in `dplyr` are more efficient.

Next, we benchmark the performance of summing by two groups. Code:

``` R
N = 1e4
set.seed(100)
x <- data.frame(Freq=runif(N,0,1),Category=c("T","F"),Category2=sample(c("T","F"), N, replace = TRUE))

library(data.table)
library(plyr)
library(dplyr)
x_dt = data.table(x)
setkey(x_dt, Category, Category2)
Cate_group_dt = group_by(x_dt, Category, Category2)
Cate_group_df = group_by(x_dt, Category, Category2)
library(rbenchmark)
benchmark(data_table = x_dt[, sum(Freq),by = key(x_dt)],
          tapply = tapply(x$Freq, list(x$Category, x$Category2), FUN=sum),
          plyr_dt2 = ddply(x_dt, .(Category, Category2), colwise(sum)),
          plyr_df2 = ddply(x, .(Category, Category2), colwise(sum)),
          dplyr_dt = summarise(Cate_group_dt, sum(Freq)),
          dplyr_df = summarise(Cate_group_df, sum(Freq)),
          replications = 100,
          columns=c('test', 'replications', 'elapsed','relative', 'user.self'),
          order='relative')

# Result for N = 1e4:
#         test replications elapsed relative user.self
# 2     tapply          100   0.093    1.000     0.359
# 1 data_table          100   0.161    1.731     0.642
# 5   dplyr_dt          100   0.217    2.333     0.833
# 6   dplyr_df          100   0.219    2.355     0.219
# 4   plyr_df2          100   0.487    5.237     1.870
# 3   plyr_dt2          100   0.498    5.355     1.914

# Result for N = 1e6:
#         test replications elapsed relative user.self
# 1 data_table           20   0.476    1.000     0.960
# 6   dplyr_df           20   0.491    1.032     0.491
# 5   dplyr_dt           20   0.502    1.055     1.137
# 2     tapply           20   1.416    2.975     1.417
# 3   plyr_dt2           20   3.285    6.901    12.437
# 4   plyr_df2           20   3.292    6.916    12.673

# Result for N = 1e7:
#         test replications elapsed relative user.self
# 1 data_table           20   4.472    1.000     4.427
# 5   dplyr_dt           20   4.520    1.011     4.454
# 6   dplyr_df           20   4.526    1.012     4.460
# 2     tapply           20  14.732    3.294    14.245
# 4   plyr_df2           20  31.481    7.040    48.578
# 3   plyr_dt2           20  31.625    7.072    48.127
```

In the case of summing by two groups, `data.table` is much more efficient in large size. We also try a `Rcpp` in summing by groups. Code:


``` R
library(Rcpp)
sourceCpp(code = '
// [[Rcpp::depends(RcppArmadillo)]]
#include <RcppArmadillo.h>
using namespace Rcpp;
using namespace arma;

// [[Rcpp::export]]
NumericVector group_sum_f(NumericVector Xr, IntegerVector Groupr) {
  int n = Xr.size();
  colvec X(Xr.begin(), n, false);
  ivec Group(Groupr.begin(), n, false);
  ivec Group_unique = unique(Group);
  int k = Group_unique.size();
  colvec result(k);
  for(int i = 0; i < k; i++)
    result(i) = sum(X.elem(find(Group == Group_unique(i))));
  return wrap(result);
}')

N = 1e4; k = 2
x = data.frame(Freq=runif(N,0,1),Category=sample(LETTERS[1:k], N, replace = TRUE))

library(data.table)
library(dplyr)
x_dt = data.table(x)
setkey(x_dt, Category)
Cate_group_dt = group_by(x_dt, Category)
Cate_group_df = group_by(x_dt, Category)
library(rbenchmark)
benchmark(data_table = x_dt[, sum(Freq),by = Category],
          tapply = tapply(x$Freq, x$Category, FUN=sum),
          Rcpp = group_sum_f(x$Freq, x$Category),
          dplyr_dt = summarise(Cate_group_dt, sum(Freq)),
          dplyr_df = summarise(Cate_group_df, sum(Freq)),
          replications = 100,
          columns=c('test', 'replications', 'elapsed','relative', 'user.self'),
          order='relative')

# Result for N = 1e4:
#         test replications elapsed relative user.self
# 3       Rcpp           20   0.005      1.0     0.005
# 2     tapply           20   0.016      3.2     0.016
# 1 data_table           20   0.027      5.4     0.027
# 5   dplyr_df           20   0.035      7.0     0.035
# 4   dplyr_dt           20   0.037      7.4     0.037


# Result for N = 1e6:
#         test replications elapsed relative user.self
# 3       Rcpp           20   0.502    1.000     0.502
# 1 data_table           20   0.616    1.227     0.616
# 5   dplyr_df           20   0.620    1.235     0.620
# 4   dplyr_dt           20   0.628    1.251     0.628
# 2     tapply           20   1.141    2.273     1.142

# Result for N = 1e7:
# #         test replications elapsed relative user.self
# 3       Rcpp           20   5.449    1.000     5.365
# 4   dplyr_dt           20   5.883    1.080     5.789
# 5   dplyr_df           20   5.916    1.086     5.821
# 1 data_table           20   5.967    1.095     5.841
# 2     tapply           20  12.030    2.208    11.603
```

We can see that `data.table` is compatible with `Rcpp` and more convenient than `Rcpp`.

My environment is ubuntu 14.04, R 3.1.1 compiled by intel c++, fortran compiler with MKL. My CPU is 3770K@4.3GHz.





