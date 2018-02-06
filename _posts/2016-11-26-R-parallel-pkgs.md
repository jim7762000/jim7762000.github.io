---
layout: post
title: "A comparison for R parallel packages"
---

有些程式不能全部靠RcppParallel加速

所以想說只能靠R的一些平行套件來解決

但是平行套件其實不少，那哪一個又有比較好的performance?

下面是Benchmark的R script:

``` R
require(doParallel)
library(foreach)
require(snowfall)
library(microbenchmark)

sfInit(parallel = TRUE, cpus = 6L)
cl <- sfGetCluster()
registerDoParallel(cl)

a <- rnorm(1e3)
b <- rnorm(1e4)
d <- 0.5
e <- rnorm(1e5) # unused variables
# f is not a good method to get that result, it is just for benchmark
f <- function(x) {
   sum <- 0
   for (i in seq(1, x)) sum <- sum + (mean(a) - mean(b))*d*i
   return(sum)
}
sfExport("a", "b", "d")
clusterExport(cl, c("a", "b", "d"))

g1 <- function(x) {
  out1 <- vector("numeric", length = 100)
  for (i in 1:1000) out1[[i]] <- f(i)
  return(out1)
}

g2 <- function(x) {
  out2 <- sapply(1:1000, f)
  return(out2)
}

g3 <- function(x) {
  out3 <- sfSapply(1:1000, f)
  return(out3)
}

g4 <- function(x) {
  out4 <- parSapply(cl, 1:1000, f)
  return(out4)
}

g5 <- function(x) {
  out5 <- foreach(i = 1:1000, .combine = c, .export = "f") %dopar% f(i)
  return(out5)
}

microbenchmark(g1(), g2(), g3(), g4(), g5(), times = 20L)
# Unit: seconds
# expr       min        lq      mean    median        uq       max neval
# g1() 12.197837 12.289874 12.364563 12.351551 12.460248 12.549038    20
# g2() 12.149057 12.213601 12.274867 12.262235 12.318187 12.548602    20
# g3()  3.846017  3.983526  4.065733  4.047937  4.075122  4.530810    20
# g4()  3.933617  3.973013  4.030316  4.016203  4.074077  4.224589    20
# g5()  2.764633  2.800048  2.861919  2.845468  2.914351  3.008172    20

# find the main difference
library(profvis)
profvis(g3()) # using clusterApply
profvis(g5()) # using clusterApplyLB
# clusterApplyLB is a load balancing version of clusterApply. If the length p of seq is not greater 
# than the number of nodes n, then a job is sent to p nodes. Otherwise the first n jobs are placed 
# in order on the n nodes. When the first job completes, the next job is placed on the node that 
# has become free; this continues until all jobs are complete. Using clusterApplyLB can result in 
# better cluster utilization than using clusterApply, but increased communication can reduce performance. 
# Furthermore, the node that executes a particular job is non-deterministic.

g6 <- function(x) {
  out6 <- clusterApplyLB(cl, 1:1000, f)
  return(out6)
}
g7 <- function(x) {
  out7 <- parSapplyLB(cl, 1:1000, f)
  return(out7)
}
library(plyr)
g8 <- function(x) {
  out8 <- laply(1:1000, f, .parallel = TRUE)
  return(out8)
}
microbenchmark(g5(), g6(), g7(), g8(), times = 20L)
# Unit: seconds
#  expr      min       lq     mean   median       uq      max neval
#  g5() 2.811597 2.840651 2.899999 2.860215 2.925780 3.193207    20
#  g6() 2.627146 2.640597 2.692733 2.680902 2.734413 2.816154    20
#  g7() 3.949628 3.978429 4.045335 4.029595 4.068357 4.384041    20
#  g8() 2.866590 2.883808 2.927684 2.909651 2.957875 3.070483    20

sfStop()
rm(cl)
```

結論，如果要用比較底層的parallel function

考慮使用有loading balancing的函數可能會gain到比較多平行的效果

但是也要考慮communication帶來的

因此，在某些場合下，寫起來可能就不是那麼方便了

不過也非一定要學`foreach`這個套件

基本上，Hadley的`plyr`已經把`foreach`直接包在裡面

只要register一個parallel cluster (or `multicore`) (`multicore`這個套件只能用在linux上)

並在`*ply`的函數後面加上`.parallel = TRUE`就可以直接享受`foreach`幫你自動調整的速度了

不過我手上沒有loading balancing帶來比較差效能的例子，未來遇上再補上


額外補充，`foreach`有可能遇到的雷：

如果input的長度是不一定的，有可能是1的話，會帶來一些麻煩

當output是向量的時候，`foreach`的`.combine`使用`cbind`

會導致長度1的時候輸出的不是`matrix`，而是`vector`

在使用`foreach`時，這一點要特別注意

``` R
# without parallel
f2 <- function(x) rnorm(5)

o1 <- foreach(i = 1:2, .combine = cbind) %do% f2(i)
o2 <- sapply(cl, 1:2, f2)

o3 <- foreach(i = 1, .combine = cbind) %do% f2(i)
o4 <- sapply(cl, 1, f2)

class(o1)  # "matrix"
class(o2)  # "matrix"
class(o3)  # "matrix"
class(o4)  # "numeric"
```

2016/11/28補充：

後來發現一個整合還不錯的套件 - `parallelMap`

``` R
require2(parallelMap)
parallelStart("socket", 6L)

a <- rnorm(1e3)
b <- rnorm(1e4)
d <- 0.5
e <- rnorm(1e5) # unused variables
# f is not a good method to get that result, it is just for benchmark
f <- function(x) {
  sum <- 0
  for (i in seq(1, x)) sum <- sum + (mean(a) - mean(b))*d*i
  return(sum)
}
parallelExport("a", "b", "d")

g9 <- function(x) {
  out9 <- parallelSapply(1:1000, f)
  return(out9)
}

library(microbenchmark)
microbenchmark(g9(), times = 20L)
# Unit: seconds
#  expr      min       lq     mean   median    uq      max neval
#  g9() 2.921699 2.931741 3.002052 2.941502 3.018 3.317669    20
parallelStop()
```

表現也跟前面用loading balancing的函數差不多


