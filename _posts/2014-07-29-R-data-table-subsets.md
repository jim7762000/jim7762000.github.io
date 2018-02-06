---
layout: post
title: R data.table - subsets
---

`data.table` is a powerful tool for exploring data. However, how is it fast? Here we provides a performance test for subsetting data.

Code:

``` R
library(data.table)
library(dplyr)
library(fastmatch)
library(Rcpp)
library(rbenchmark)
perf_test = function(N){
    tmp <- list()
    for(i in 1:N) tmp[[i]] <- iris
    m <- do.call(rbind, tmp)
    m2 = data.table(m)
    setkey(m2, "Sepal.Width")
    m3 = as.matrix(m[,1:4])
    benchmark(replications=100, order = "relative",
        data.frame = m[m$Sepal.Width == 3.5,],
        subset = subset(m, Sepal.Width == 3.5),
        dt1 = m2[J(3.5)],
        filter_dt = filter(m, Sepal.Width == 3.5),
        filter_df = filter(m2, Sepal.Width == 3.5),
        dt2 = m2[list(3.5)],
        fmatch = m2[fmatch(m2$Sepal.Width, 3.5, nomatch = 0L),],
        matrix = m3[m3[,2]==3.5,],
        columns = c("test", "replications", "elapsed", "relative")
  )
}
# iris的大小
object.size(iris)
# 7088 bytes
# 200倍的資料量
perf_test(200)
#         test replications elapsed relative
# 4  filter_dt          100   0.038    1.000
# 5  filter_df          100   0.088    2.316
# 6        dt2          100   0.119    3.132
# 3        dt1          100   0.131    3.447
# 8     matrix          100   0.134    3.526
# 7     fmatch          100   0.222    5.842
# 1 data.frame          100   0.407   10.711
# 2     subset          100   0.490   12.895

# 500倍的資料量
perf_test(500)
#         test replications elapsed relative
# 4  filter_dt          100   0.083    1.000
# 5  filter_df          100   0.119    1.434
# 6        dt2          100   0.126    1.518
# 3        dt1          100   0.127    1.530
# 8     matrix          100   0.371    4.470
# 7     fmatch          100   0.517    6.229
# 1 data.frame          100   1.056   12.723
# 2     subset          100   1.224   14.747

# 1000倍的資料量
perf_test(1000)
#         test replications elapsed relative
# 3        dt1          100   0.136    1.000
# 6        dt2          100   0.139    1.022
# 4  filter_dt          100   0.159    1.169
# 5  filter_df          100   0.194    1.426
# 8     matrix          100   0.809    5.949
# 7     fmatch          100   1.128    8.294
# 1 data.frame          100   2.157   15.860
# 2     subset          100   2.541   18.684

# 1500倍的資料量
perf_test(1500)
#         test replications elapsed relative
# 3        dt1          100   0.144    1.000
# 6        dt2          100   0.148    1.028
# 4  filter_dt          100   0.259    1.799
# 5  filter_df          100   0.287    1.993
# 8     matrix          100   1.204    8.361
# 7     fmatch          100   1.543   10.715
# 1 data.frame          100   3.242   22.514
# 2     subset          100   3.729   25.896

# 3000倍的資料量
perf_test(3000)
#         test replications elapsed relative
# 3        dt1          100   0.174    1.000
# 6        dt2          100   0.174    1.000
# 5  filter_df          100   0.405    2.328
# 4  filter_dt          100   0.509    2.925
# 8     matrix          100   2.441   14.029
# 7     fmatch          100   2.993   17.201
# 1 data.frame          100   6.428   36.943
# 2     subset          100   7.458   42.862

# 5000倍的資料量
perf_test(5000)
#         test replications elapsed relative
# 6        dt2          100   0.224    1.000
# 3        dt1          100   0.225    1.004
# 5  filter_df          100   0.632    2.821
# 4  filter_dt          100   0.869    3.879
# 8     matrix          100   4.027   17.978
# 7     fmatch          100   4.797   21.415
# 1 data.frame          100  10.578   47.223
# 2     subset          100  12.177   54.362
```

After above benchmarks, we can see that `filter` in `dplyr` is fast when data size is low (lower than 10 MB), but data.table searching by key is faster when data size is larger. Fastmatch is not fast. HAHA!! Even data.frame is slower than matrix. `data.table` is so worth to learn!

My environment is ubuntu 14.04, R 3.1.1 compiled by intel c++, fortran compiler with MKL. My CPU is 3770K@4.3GHz.


