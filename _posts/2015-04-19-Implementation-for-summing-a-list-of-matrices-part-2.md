---
layout: post
title: Implementation for summing a list of matrices -- part 2
---

I have showed several implementations for summing a list of matrices in the previous post. I introduce a function to be applied in `do.call` and test performance.

``` R
N = 10
input = lapply(1:N**2, function(x) matrix(rnorm(N**2), N))
a = do.call(function(...){
  arglist = list(...)
  #browser()
  out = arglist[[1]]
  for (i in 2:length(arglist ))
    out = out + arglist[[i]]
  return(out)
}, input)
b = Reduce('+', input)
for_sum = function(input){
  out = input[[1]]
  for (i in 2:length(input))
    out = out + input[[i]]
  return(out)
}
d = for_sum(input)
all.equal(a, b)
all.equal(a, d)

N = 200
K = 100
library(microbenchmark)
microbenchmark(a = do.call(function(...){
  arglist = list(...)
  out = arglist[[1]]
  for (i in 2:length(arglist ))
    out = out + arglist[[i]]
  return(out)
  }, lapply(1:K, function(x) matrix(rnorm(N**2), N))),
  b = Reduce('+', lapply(1:K, function(x) matrix(rnorm(N**2), N))),
  d = for_sum(lapply(1:K, function(x) matrix(rnorm(N**2), N))),
  times = 20L
)
# Unit: milliseconds
#  expr      min       lq     mean   median       uq      max neval
#     a 392.7532 409.6669 431.2166 424.3133 443.7561 498.5549    20
#     b 399.3968 414.4893 446.8753 432.2366 444.0107 685.5244    20
#     d 379.0724 400.0660 411.3291 408.3251 427.2988 443.7041    20
```
