---
layout: post
title: Grid search in data.table
---

有人傳了一篇用`tidyverse`做grid search的blogger給我看 ([Grid search in the tidyverse](https://www.r-bloggers.com/grid-search-in-the-tidyverse/))

我想說那我來寫一篇for `data.table`的吧，code如下：

``` R
library(data.table)
library(pipeR)
library(rpart)

set.seed(245)
dataDT <- data.table(mtcars) %>>% `[`(j = am := factor(am, labels = c("Automatic", "Manual")))
dataDT[ , trainFlag := !is.na(match(.I, sample.int(nrow(mtcars), floor(.8 * nrow(mtcars)))))]

buildMdl <- function(s, d) {
  rpart(am ~ hp + mpg, dataDT[trainFlag == TRUE], control = rpart.control(minsplit = s, maxdepth = d))
}

gs <- CJ(minsplit = c(2, 5, 10), maxdepth = c(1, 3, 8))
gs[ , mod := list(mapply(buildMdl, minsplit, maxdepth, SIMPLIFY = FALSE))]
print(gs)
#    minsplit maxdepth     mod
# 1:        2        1 <rpart>
# 2:        2        3 <rpart>
# 3:        2        8 <rpart>
# 4:        5        1 <rpart>
# 5:        5        3 <rpart>
# 6:        5        8 <rpart>
# 7:       10        1 <rpart>
# 8:       10        3 <rpart>
# 9:       10        8 <rpart>

calAccu <- function(mod, testData, testLabel) {
  mean(predict(mod, testData, type = "class") == testLabel)
}

gs[ , `:=`(trainAccu = mapply(function(m) dataDT[trainFlag == TRUE] %>>% {calAccu(m, ., .$am)}, mod),
           testAccu = mapply(function(m) dataDT[trainFlag == FALSE] %>>% {calAccu(m, ., .$am)}, mod))]
setorder(gs, -testAccu, -trainAccu)
print(gs)
#    minsplit maxdepth     mod trainAccu  testAccu
# 1:        2        8 <rpart>      1.00 0.8571429
# 2:        2        3 <rpart>      0.92 0.8571429
# 3:        5        3 <rpart>      0.88 0.8571429
# 4:        5        8 <rpart>      0.88 0.8571429
# 5:        2        1 <rpart>      0.84 0.7142857
# 6:        5        1 <rpart>      0.84 0.7142857
# 7:       10        1 <rpart>      0.84 0.7142857
# 8:       10        3 <rpart>      0.84 0.7142857
# 9:       10        8 <rpart>      0.84 0.7142857
```

我是覺得寫起來有點麻煩XD，改成用`foreach` + `iterators`我覺得會好很多，code如下：

``` R
library(data.table)
library(pipeR)
library(rpart)
library(foreach)
library(iterators)

set.seed(245)
dataDT <- data.table(mtcars) %>>% `[`(j = am := factor(am, labels = c("Automatic", "Manual")))
dataDT[ , trainFlag := !is.na(match(.I, sample.int(nrow(mtcars), floor(.8 * nrow(mtcars)))))]

resDT <- CJ(minsplit = c(2, 5, 10), maxdepth = c(1, 3, 8)) %>>%
  {
    foreach(i = isplit(., as.list(.)), .final = rbindlist) %do% 
    {
      mod <- rpart(am ~ hp + mpg, dataDT[trainFlag == TRUE], 
                   control = do.call(rpart.control, i$value))
      trainAccu <- dataDT[trainFlag == TRUE] %>>% {mean(predict(mod, ., type = "class") == .$am)}
      testAccu <- dataDT[trainFlag == FALSE] %>>% {mean(predict(mod, ., type = "class") == .$am)}
      return(cbind(i$value, data.table(mod = list(mod), trainAccu = trainAccu, testAccu = testAccu)))
    }
  } %>>% setorder(-testAccu, -trainAccu)
print(resDT)
#    minsplit maxdepth     mod trainAccu  testAccu
# 1:        2        8 <rpart>      1.00 0.8571429
# 2:        2        3 <rpart>      0.92 0.8571429
# 3:        5        3 <rpart>      0.88 0.8571429
# 4:        5        8 <rpart>      0.88 0.8571429
# 5:        2        1 <rpart>      0.84 0.7142857
# 6:        5        1 <rpart>      0.84 0.7142857
# 7:       10        1 <rpart>      0.84 0.7142857
# 8:       10        3 <rpart>      0.84 0.7142857
# 9:       10        8 <rpart>      0.84 0.7142857
```

`isplit`那裏是shadow copy，所以應該不會造成什麼問題

這樣寫會比用`tidyverse`或是直接`data.table` + `mapply`來的直覺，而且程式會相對精簡很多

