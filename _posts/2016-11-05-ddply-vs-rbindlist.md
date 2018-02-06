---
layout: post
title: plyr::ddply vs data.table::rbindlist
---

最近遇到在計算functinoal data的cross-covariance surface的時候

發現`plyr::ddply`裡面的`list_to_dataframe`有點慢

反而利用`plyr::dlply`加上`data.table`的`rbindlist`可以快上不少

而且`plyr::ddply`消耗的記憶體相對起`rbindlist`高上不少

會發現這些都要感謝rstudio新出的套件`profvis`提供了良好的performance視覺化

其中`profvis`可以在[github](https://github.com/rstudio/profvis)找到

下面是Benchmark的R script:

``` R
library(plyr)
library(data.table)
library(pipeR)

N <- 2000
nt <- 100
p <- 4
dataDT <- data.table(subId = rep(1:(N/nt), p, each = nt), variable = rep(1:p, each = N), 
                     timePnt = rep(seq(0, 10, length.out = nt), p*N/nt), value = rnorm(N*p))

getRawCrCov1 <- function(demeanDataDT){
  # geneerate the all combinations of t1,t2 and varaibles
  baseDT <- demeanDataDT[ , .(t1 = rep(timePnt, length(timePnt)), t2 = rep(timePnt, each=length(timePnt)),
                              value.var1 = rep(value, length(timePnt))), by = .(variable, subId)]
  # calculation of raw cross-covariance
  rawCrCovDT <- do.call("dlply", list(demeanDataDT, "variable", function(df){
    merge(baseDT[variable >= df$variable[1]], df, suffixes = c("1", "2"),
          by.x = c("subId", "t2"), by.y = c("subId", "timePnt"))
  })) %>>% rbindlist %>>% setnames("value", "value.var2") %>>%
    `[`(j = .(sse = sum(value.var1 * value.var2), cnt = .N), by = .(variable1, variable2, t1, t2)) %>>%
    setorder(variable1, variable2, t1, t2) %>>% `[`(j = weight := 1)
  return(rawCrCovDT)
}

getRawCrCov2 <- function(demeanDataDT){
  # geneerate the all combinations of t1,t2 and varaibles
  baseDT <- demeanDataDT[ , .(t1 = rep(timePnt, length(timePnt)), t2 = rep(timePnt, each=length(timePnt)),
                              value.var1 = rep(value, length(timePnt))), by = .(variable, subId)]
  # calculation of raw cross-covariance
  rawCrCovDT <- do.call("ddply", list(demeanDataDT, "variable", function(df){
    merge(baseDT[variable >= df$variable[1]], df, suffixes = c("1", "2"),
          by.x = c("subId", "t2"), by.y = c("subId", "timePnt"))
  })) %>>% setDT %>>% `[`(j = variable := NULL) %>>% setnames("value", "value.var2") %>>%
    `[`(j = .(sse = sum(value.var1 * value.var2), cnt = .N), by = .(variable1, variable2, t1, t2)) %>>%
    setorder(variable1, variable2, t1, t2) %>>% `[`(j = weight := 1)
  return(rawCrCovDT)
}

getRawCrCov3 <- function(demeanDataDT){
  # geneerate the all combinations of t1,t2 and varaibles
  baseDT <- demeanDataDT[ , .(t1 = rep(timePnt, length(timePnt)), t2 = rep(timePnt, each=length(timePnt)),
                              value.var1 = rep(value, length(timePnt))), by = .(variable, subId)]
  # calculation of raw cross-covariance
  rawCrCovDT <- do.call("llply", list(split(demeanDataDT, by = "variable"), function(df){
    merge(baseDT[variable >= df$variable[1]], df, suffixes = c("1", "2"),
          by.x = c("subId", "t2"), by.y = c("subId", "timePnt"))
  })) %>>% rbindlist %>>% setnames("value", "value.var2") %>>%
    `[`(j = .(sse = sum(value.var1 * value.var2), cnt = .N), by = .(variable1, variable2, t1, t2)) %>>%
    setorder(variable1, variable2, t1, t2) %>>% `[`(j = weight := 1)
  return(rawCrCovDT)
}

getRawCrCov4 <- function(demeanDataDT){
  # geneerate the all combinations of t1,t2 and varaibles
  baseDT <- demeanDataDT[ , .(t1 = rep(timePnt, length(timePnt)), t2 = rep(timePnt, each=length(timePnt)),
                              value.var1 = rep(value, length(timePnt))), by = .(variable, subId)]
  # set the keys of data.table
  setkey(baseDT, "subId", "t2")
  setkey(demeanDataDT, "subId", "timePnt")
  # calculation of raw cross-covariance
  rawCrCovDT <- do.call("llply", list(split(demeanDataDT, by = "variable"), function(df){
    merge(baseDT[variable >= df$variable[1]], df, suffixes = c("1", "2"),
          by.x = c("subId", "t2"), by.y = c("subId", "timePnt"))
  })) %>>% rbindlist %>>% setnames("value", "value.var2") %>>%
    `[`(j = .(sse = sum(value.var1 * value.var2), cnt = .N), by = .(variable1, variable2, t1, t2)) %>>%
    setorder(variable1, variable2, t1, t2) %>>% `[`(j = weight := 1)
  return(rawCrCovDT)
}

x1 <- getRawCrCov1(dataDT)
x2 <- getRawCrCov2(dataDT)
x3 <- getRawCrCov3(dataDT)
x4 <- getRawCrCov4(dataDT)
all.equal(x1, x2) # TRUE
all.equal(x1, x3) # TRUE
all.equal(x1, x4) # TRUE

library(microbenchmark)
microbenchmark(rbindlist = getRawCrCov1(dataDT), ddply = getRawCrCov2(dataDT), 
               split.DT = getRawCrCov3(dataDT), setkey.first = getRawCrCov4(dataDT), times = 50L)
# Unit: milliseconds
#          expr       min        lq      mean    median        uq       max neval
#     rbindlist  657.1571  700.8676  726.3146  716.2397  741.4616  911.0223    50
#         ddply 2951.0253 3196.2281 3292.1848 3283.4185 3415.7735 3638.6447    50
#      split.DT  653.3183  699.5276  732.4667  720.5936  747.7091 1016.1446    50
#  setkey.first  496.7661  542.6954  562.4172  554.0295  584.7101  701.3526    50
```

速度整整差了近5倍(3347 / 709 ~= 4.72)

因此，建議以後plyr系列，盡量避開`*dply`系列的函數

用到`plyr:::list_to_dataframe`這個函數的效能都不好

盡量去使用`data.table::rbindlist`


用`dlply` + `rbindlist`跟用`split.data.table` + `llply` + `rbindlist`

其實最後兩者速度差不多，時間並沒有太大的區別 (726ms vs 732ms)

其實我覺得可以直接都統一用`split.data.table` + `llply` + `rbindlist`

統一減少使用`*dply`或是`d*ply`


而在迴圈中做`merge`前，建議全部都先`setkey`

這樣拆分之後的`data.table`，還是有key

`merge`時就不用再建key了

而這一點也是透過`profvis`這個套件發現

多看幾次`profvis`套件出來的結果，可以對每個程式怎麼運作更加了解

就能因時因地制宜了，程式自然會更有效率
