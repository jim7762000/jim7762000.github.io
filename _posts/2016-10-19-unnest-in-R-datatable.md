---
layout: post
title: "unnest in R data.table"
---

我們今天可能會遇到json parse出來的資料長下面這樣

``` R
library(data.table)
DT <- data.table(a = list(c(1:5), c(2:4), c(1:5)), b = 1:3, 
                 c = list(c(0:4), c(6:8), c(7:11)),  d = 2:4)
#            a b              c d
# 1: 1,2,3,4,5 1      0,1,2,3,4 2
# 2:     2,3,4 2          6,7,8 3
# 3: 1,2,3,4,5 3  7, 8, 9,10,11 4
```

那在這種情況下，可以直接選擇用`tidyr`的`unnest`去做，如下面所示

``` R
library(tidyr)
unnest(DT, a, c)
#     b d a  c
#  1: 1 2 1  0
#  2: 1 2 2  1
#  3: 1 2 3  2
#  4: 1 2 4  3
#  5: 1 2 5  4
#  6: 2 3 2  6
#  7: 2 3 3  7
#  8: 2 3 4  8
#  9: 3 4 1  7
# 10: 3 4 2  8
# 11: 3 4 3  9
# 12: 3 4 4 10
# 13: 3 4 5 11
```

但是這時候我們很難的去自動解析這種表格，必須讓使用者自行處理

(PS: 其實這裡可以直接用`unnest_` + 下面的`autoFind`裡面的一部分就可以很輕易做到，下面會實做)

所以如果我們能用簡單的方式去自動辨別需要轉換就更好了

基於此，我就用data.table去開發了這樣想的程式，如下：

``` R
extendTbl <- function(DT, unnestCols = NULL){
  # check the columns to unnest
  if (is.null(unnestCols)) {
    unnestCols <- names(DT)[sapply(DT, function(x) any(class(x) %in% "list"))]
    message("Automatically recognize the nested columns: ", paste0(unnestCols, collapse = ", "))
  }
  # check unnestCols is in the DT
  if (any(!unnestCols %in% names(DT)))
    stop(sprintf("The columns, %s, does not in the DT.",
                 paste0(unnestCols[!unnestCols %in% names(DT)], collapse = ", ")))
  # get the group by variable
  groupbyVar <- setdiff(names(DT), unnestCols)
  # generate the expression to remove group by variable
  chkExpr <- paste0(groupbyVar, "=NULL", collapse = ",") %>>% (paste0("`:=`(", ., ")"))
  # check the lengths of each cell in list-column are all the same
  chkLenAllEqual <- DT[ , lapply(.SD, function(x) sapply(x, length)), by = groupbyVar] %>>%
    `[`(j = eval(parse(text = chkExpr))) %>>% as.matrix %>>% apply(1, diff) %>>% `==`(0) %>>% all
  if (!chkLenAllEqual)
    stop("The length in each cell is not equal.")

  # generate unnest expression
  expr <- unnestCols %>>% (paste0(., "=unlist(",  ., ")")) %>>%
    paste0(collapse = ",") %>>% (paste0(".(", ., ")"))
  # return unnested data.table
  return(DT[ , eval(parse(text = expr)), by = groupbyVar])
}
extendTbl(DT)
#     b d a  c
#  1: 1 2 1  0
#  2: 1 2 2  1
#  3: 1 2 3  2
#  4: 1 2 4  3
#  5: 1 2 5  4
#  6: 2 3 2  6
#  7: 2 3 3  7
#  8: 2 3 4  8
#  9: 3 4 1  7
# 10: 3 4 2  8
# 11: 3 4 3  9
# 12: 3 4 4 10
# 13: 3 4 5 11
extendTbl(DT, c("b", "d"))
#     b d a  c
#  1: 1 2 1  0
#  2: 1 2 2  1
#  3: 1 2 3  2
#  4: 1 2 4  3
#  5: 1 2 5  4
#  6: 2 3 2  6
#  7: 2 3 3  7
#  8: 2 3 4  8
#  9: 3 4 1  7
# 10: 3 4 2  8
# 11: 3 4 3  9
# 12: 3 4 4 10
# 13: 3 4 5 11
```

我們可以來比較一下`tidyr`跟`data.table`的速度

會要求速度最主要的原因是因為我遇到的資料都非常大

不用快一點的方法，真的會等很久

``` R
library(microbenchmark)
N <- 1e5
dbColsCnt <- 300
arrColCnt <- 50

sizePerRow <- sapply(1:N, function(x) sample(2:25, 1))
arrCols <- replicate(arrColCnt, lapply(sizePerRow, rnorm)) %>>% 
  data.table %>>% setnames(paste0("V", (dbColsCnt+1):(dbColsCnt+arrColCnt)))
DT <- data.table(matrix(rnorm(dbColsCnt*N), N), arrCols)

autoFind_unnest <- function(DT){
  names(DT)[sapply(DT, function(x) any(class(x) %in% "list"))]
}
microbenchmark(unnest = unnest_(DT, autoFind_unnest(DT)),
               datatable = extendTbl(DT), times = 20L)
autoFind_unnest <- function(DT){
  names(DT)[sapply(DT, function(x) any(class(x) %in% "list"))]
}
microbenchmark(unnest = unnest_(DT, autoFind_unnest(DT)),
               datatable = extendTbl(DT), times = 20L)
# Unit: seconds
#      expr       min       lq     mean   median       uq     max neval
#    unnest 15.806110 16.29989 16.62312 16.75588 16.88089 17.4564    20
# datatable  9.362995 10.25902 10.82319 10.45128 10.94098 14.0597    20
```
