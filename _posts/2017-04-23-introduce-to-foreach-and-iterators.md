---
layout: post
title: "Introduce to foreach and iterators"
---

## Introduction `*apply` functions, `Reduce` and `do.call`

我一直沒時間寫`*apply`系列函數的介紹

不過最近用了`foreach`跟`iterators`

發現其實這兩個套件就可以完全取代`*apply`系列函數

而且對新手來說寫起來也不容易搞錯


這裡先簡介一下`*apply`系列函數

1. `apply`


```r
m <- matrix(1:6, 3)
# input matrix, output vector/matrix/list
apply(m, 1, sum)  # output vector
```

```
## [1] 5 7 9
```

```r
apply(m, 1, `*`, 2)  # output matrix
```

```
##      [,1] [,2] [,3]
## [1,]    2    4    6
## [2,]    8   10   12
```

```r
apply(m, 1, function(v) rep(v[1], v[2]))  # output list
```

```
## [[1]]
## [1] 1 1 1 1
## 
## [[2]]
## [1] 2 2 2 2 2
## 
## [[3]]
## [1] 3 3 3 3 3 3
```

2. `lapply`


```r
# input vector/list, output list
lapply(1:3, `*`, 2)  # input vector
```

```
## [[1]]
## [1] 2
## 
## [[2]]
## [1] 4
## 
## [[3]]
## [1] 6
```

```r
lapply(as.list(1:3), `*`, 2)  # input list
```

```
## [[1]]
## [1] 2
## 
## [[2]]
## [1] 4
## 
## [[3]]
## [1] 6
```


3. `sapply`


```r
# input vector/list, output vector/matrix/list
sapply(1:3, `*`, 2)  # input vector, output vector
```

```
## [1] 2 4 6
```

```r
sapply(1:3, rep, each = 2)  # input vector, output matrix
```

```
##      [,1] [,2] [,3]
## [1,]    1    2    3
## [2,]    1    2    3
```

```r
sapply(1:3, rep, x = 2)  # input vector, output list
```

```
## [[1]]
## [1] 2
## 
## [[2]]
## [1] 2 2
## 
## [[3]]
## [1] 2 2 2
```

```r
sapply(as.list(1:3), `*`, 2)  # input list, output vector
```

```
## [1] 2 4 6
```

```r
sapply(as.list(1:3), rep, each = 2)  # input list, output matrix
```

```
##      [,1] [,2] [,3]
## [1,]    1    2    3
## [2,]    1    2    3
```

```r
sapply(as.list(1:3), rep, x = 2)  # input list, output list
```

```
## [[1]]
## [1] 2
## 
## [[2]]
## [1] 2 2
## 
## [[3]]
## [1] 2 2 2
```

4. `mapply`


```r
# multiple inputs, output vector/matrix/list stick list output by using `SIMPLIFY = FALSE`
mapply(function(x, y) x + y, 1:3, as.list(2:4))  # input vector and list, output vector
```

```
## [1] 3 5 7
```

```r
mapply(function(x, y) rep(x * y, 2), 1:3, as.list(2:4))  # input vector and list, output matrix
```

```
##      [,1] [,2] [,3]
## [1,]    2    6   12
## [2,]    2    6   12
```

```r
mapply(function(x, y) x + y, 1:3, as.list(2:4), SIMPLIFY = FALSE)  # input vector and list, output list
```

```
## [[1]]
## [1] 3
## 
## [[2]]
## [1] 5
## 
## [[3]]
## [1] 7
```

```r
mapply(function(x, y) rep(x, y), 1:3, as.list(2:4))  # input vector and list, output list
```

```
## [[1]]
## [1] 1 1
## 
## [[2]]
## [1] 2 2 2
## 
## [[3]]
## [1] 3 3 3 3
```

5. `tapply`


```r
# input two vectors, one is value and the other one is group.  output vector/matrix
tapply(1:6, rep(1:2, 3), sum)
```

```
##  1  2 
##  9 12
```

```r
tapply(1:6, rep(1:2, 3), `+`, 1)
```

```
## $`1`
## [1] 2 4 6
## 
## $`2`
## [1] 3 5 7
```

簡單掃過`apply`系列之後，有沒有覺得光要弄懂輸出什麼格式就很折磨人了...

這裡額外提一下`do.call`這個指令

`do.call`是`do a function call`的意思

所以如果我們要把`list`的資料合併再一起

那我們就可以執行一個`c`/`cbind`/`rbind`的function call

把`list`裡面的element當成`input`丟入這個function call，例子如下：


```r
do.call(c, list(1, 2, 3))
```

```
## [1] 1 2 3
```

```r
do.call(cbind, list(1:2, 2:3, 3:4))
```

```
##      [,1] [,2] [,3]
## [1,]    1    2    3
## [2,]    2    3    4
```

```r
do.call(rbind, list(1:2, 2:3, 3:4))
```

```
##      [,1] [,2]
## [1,]    1    2
## [2,]    2    3
## [3,]    3    4
```

這裡可能有人學過`Reduce`這個函數

雖然寫法也差不多，像是：`Reduce(c, list(1, 2, 3))`這樣

但是其實差異還是滿大的，我這邊簡單呈現一下`do.call`跟`Reduce`的差異

假設一個`list`有四個elements，那`Reduce`跟`do.call`分別的做法如下：


```r
c(c(c(1, 2), 3), 4)  # <= Reduce的做法
```

```
## [1] 1 2 3 4
```

```r
c(1, 2, 3, 4)  # <= do.call的做法
```

```
## [1] 1 2 3 4
```

可以看出`Reduce`是兩兩做，所以他在內部處理的時候需要不斷擴充output vector/matrix的長度

但是`do.call`是一次做完，所以`do.call`就不需要擴充長度

效率方面可以自己動手測測看，會比較有感

那`Reduce`要幹嘛用？其實像是operator都只能輸入兩個參數，這情境就只能用`Reduce`

或是有一個`list`，element都是`data.frame`，你要根據每個element的`id`欄位做合併

那這時候你能用的函數也只有`merge`而已，簡單範例如下：


```r
listDF <- list(data.frame(id = 1:5, V1 = rnorm(5)), data.frame(id = 1:5, V2 = rnorm(5)), data.frame(id = 1:5, V3 = rnorm(5)))
Reduce(function(x, y) merge(x, y, by = "id"), listDF)
```

```
##   id         V1         V2         V3
## 1  1  0.5019189  1.9036192 -1.6574498
## 2  2 -1.2667358  0.3022502  0.1295988
## 3  3 -0.7827332 -3.4698064  0.4290014
## 4  4 -1.6435900  1.0705252  2.0689225
## 5  5 -0.4572341 -0.1567469 -0.1389343
```

講那麼久，還沒到正題Orz


## `foreach`

接下來，我們來進入正題吧！

`foreach`的用法相當直覺，我們從最簡單的case開始：


```r
foreach(x = 1:3) %do% {
    x + 1
}
```

```
## [[1]]
## [1] 2
## 
## [[2]]
## [1] 3
## 
## [[3]]
## [1] 4
```

輸入一個vector: 1:3，對每一個element做加一的動作，然後輸出是`list`

除非調整`foreach`的參數，不然輸出絕對是`list`，這是`foreach`第一個特性

`foreach`這個函數會得到一個物件，而透過`%do%`這個operator就可以執行後面的命令

`{}`是可以省略的，但是只限於一行，而且那一行不能是用operator的運算，不然會出現像是下面的錯誤


```r
tryCatch({
    foreach(x = 1:3) %do% x + 1
}, error = function(e) e)
```

```
## <simpleError in foreach(x = 1:3) %do% x + 1: 二元運算子中有非數值引數>
```

如果還是不要`{}`，可以用下面這樣，把operator當成函數來用即可：


```r
foreach(x = 1:3) %do% (x + 1)
```

```
## [[1]]
## [1] 2
## 
## [[2]]
## [1] 3
## 
## [[3]]
## [1] 4
```


再來，我們介紹怎麼改成vector/matrix輸出

我們只要在`foreach`這個命令裡面加上`.combine`這個屬性

那他裡面放的是我們要把結果合併的`function`，舉例如下：


```r
foreach(x = 1:3, .combine = c) %do% {
    x + 1
}  # vector
```

```
## [1] 2 3 4
```

```r
foreach(x = 1:3, .combine = cbind) %do% {
    c(x, x + 1)
}  # matrix
```

```
##      result.1 result.2 result.3
## [1,]        1        2        3
## [2,]        2        3        4
```

```r
foreach(x = 1:3, .combine = rbind) %do% {
    c(x, x + 1)
}  # matrix
```

```
##          [,1] [,2]
## result.1    1    2
## result.2    2    3
## result.3    3    4
```

那這裡的`.combine`用的是`Reduce`，如果要用`do.call`的話，要在加上一個參數，如：


```r
foreach(x = 1:3, .combine = c, .multicombine = TRUE) %do% {
    x + 1
}  # vector
```

```
## [1] 2 3 4
```

```r
foreach(x = 1:3, .combine = cbind, .multicombine = TRUE) %do% {
    c(x, x + 1)
}  # matrix
```

```
##      result.1 result.2 result.3
## [1,]        1        2        3
## [2,]        2        3        4
```

```r
foreach(x = 1:3, .combine = rbind, .multicombine = TRUE) %do% {
    c(x, x + 1)
}  # matrix
```

```
##          [,1] [,2]
## result.1    1    2
## result.2    2    3
## result.3    3    4
```

`foreach`還有幾個參數，稍微帶過，有些參數之後會再細講

`.inorder`是輸出是否要照順序，`.maxcombine`是最大結合數量

`.errorhandling`則是對錯誤的處理方式，`verbose`則是顯示更多訊息已提供user debug用

至於`.package`, `.export`跟`.noexport`，我們留到後面再解釋


剛剛上面看到`foreach`可以input vector, output list/vector/matrix了

但是我們還沒試過是不是`list`也行，舉個小範例：


```r
foreach(x = as.list(1:3)) %do% rep(x, x)  # list
```

```
## [[1]]
## [1] 1
## 
## [[2]]
## [1] 2 2
## 
## [[3]]
## [1] 3 3 3
```

```r
foreach(x = as.list(1:3), .combine = c, .multicombine = TRUE) %do% rep(x, x)  # vector
```

```
## [1] 1 2 2 3 3 3
```

```r
foreach(x = as.list(1:3), .combine = cbind, .multicombine = TRUE) %do% rep(x, 2)  # matrix
```

```
##      result.1 result.2 result.3
## [1,]        1        2        3
## [2,]        1        2        3
```

那這樣其實就可以完全取代`sapply`跟`lapply`的功能了


再來是`foreach`也可以支援多個input，把`mapply`的範例重新用`foreach`寫一次：


```r
foreach(x = 1:3, y = as.list(2:4)) %do% {
    x + y
}
```

```
## [[1]]
## [1] 3
## 
## [[2]]
## [1] 5
## 
## [[3]]
## [1] 7
```

```r
foreach(x = 1:3, y = as.list(2:4)) %do% rep(x * y, 2)
```

```
## [[1]]
## [1] 2 2
## 
## [[2]]
## [1] 6 6
## 
## [[3]]
## [1] 12 12
```

```r
foreach(x = 1:3, y = as.list(2:4)) %do% {
    x + y
}
```

```
## [[1]]
## [1] 3
## 
## [[2]]
## [1] 5
## 
## [[3]]
## [1] 7
```

```r
foreach(x = 1:3, y = as.list(2:4)) %do% rep(x, y)
```

```
## [[1]]
## [1] 1 1
## 
## [[2]]
## [1] 2 2 2
## 
## [[3]]
## [1] 3 3 3 3
```


但是`foreach`除了取代`sapply`跟`lapply`之外

其實`foreach`還有很棒的debug功能：




```r
print("i" %in% ls())
```

```
## [1] FALSE
```

```r
result <- foreach(i = 1:3) %do% {
    i + 1
}
if ("i" %in% ls()) print(i)
```

```
## [1] 3
```

可以看出來`foreach`會自動assign `i`，讓我們能夠直接把`i`帶入`{}`裡面直接執行裡面的程式，看最後一次的執行結果

另外，`foreach`還會提示是在哪一個task中失敗的：


```r
tryCatch({
    foreach(x = 1:3) %do% {
        if (x == 2) 
            stop("")
    }
}, error = function(e) e)
```

```
## <simpleError in {    if (x == 2)         stop("")}: task 2 failed - "">
```

上面的輸出可以看的出來就是`task 2 failed`，所以我們只要用`x=2`帶進`{}`裡面就可以找到問題了！

介紹到這，`foreach`還只是用到30%而已，這裡我要再介紹搭配同為Revolution R Analysis出的`iterators`來使用


## `iterators`

一開頭介紹的`*apply`系列函數還有兩個沒有用`foreach`取代

這節就是要來用`iteratos`的功能來取代`apply`跟`tapply`

要取代`apply`就是我們要讓`foreach`能一次存取一個列或是一個行

那麼就是用`iter`這個函數，然後藉由`by`這個參數來控制要算列還是行


```r
m <- matrix(1:6, 3)
# input matrix, output vector/matrix/list
foreach(v = iter(m, by = "row"), .combine = c, .multicombine = TRUE) %do% sum(v)
```

```
## [1] 5 7 9
```

```r
foreach(v = iter(m, by = "row"), .combine = cbind, .multicombine = TRUE) %do% (v * 2)
```

```
##      [,1] [,2] [,3] [,4] [,5] [,6]
## [1,]    2    8    4   10    6   12
```

```r
foreach(v = iter(m, by = "row")) %do% rep(v[1], v[2])
```

```
## [[1]]
## [1] 1 1 1 1
## 
## [[2]]
## [1] 2 2 2 2 2
## 
## [[3]]
## [1] 3 3 3 3 3 3
```

```r
foreach(v = iter(m, by = "column"), .combine = c, .multicombine = TRUE) %do% sum(v)
```

```
## [1]  6 15
```

```r
foreach(v = iter(m, by = "column"), .combine = cbind, .multicombine = TRUE) %do% (v * 2)
```

```
##      [,1] [,2]
## [1,]    2    8
## [2,]    4   10
## [3,]    6   12
```

```r
foreach(v = iter(m, by = "column")) %do% rep(v[1], v[2])
```

```
## [[1]]
## [1] 1 1
## 
## [[2]]
## [1] 4 4 4 4 4
```

如果是`array`的話就用`iapply`這個函數來取代`iter`，舉例如下：


```r
arr <- array(1:8, rep(2, 3))
foreach(v = iapply(arr, 1)) %do% {
    v
}  # iterate over rows
```

```
## [[1]]
##      [,1] [,2]
## [1,]    1    5
## [2,]    3    7
## 
## [[2]]
##      [,1] [,2]
## [1,]    2    6
## [2,]    4    8
```

```r
foreach(v = iapply(arr, 2)) %do% {
    v
}  # iterate over columns
```

```
## [[1]]
##      [,1] [,2]
## [1,]    1    5
## [2,]    2    6
## 
## [[2]]
##      [,1] [,2]
## [1,]    3    7
## [2,]    4    8
```

```r
foreach(v = iapply(arr, 3)) %do% {
    v
}  # iterate over slice
```

```
## [[1]]
##      [,1] [,2]
## [1,]    1    3
## [2,]    2    4
## 
## [[2]]
##      [,1] [,2]
## [1,]    5    7
## [2,]    6    8
```

```r
foreach(v = iapply(arr, 2:3)) %do% {
    v
}  # # iterate over all the columns of all the matrices
```

```
## [[1]]
## [1] 1 2
## 
## [[2]]
## [1] 3 4
## 
## [[3]]
## [1] 5 6
## 
## [[4]]
## [1] 7 8
```

`tapply`的部分則是要引進`isplit`這個函數

只是要注意`isplit`的`iterator`跟前面使用方式不同

要用值的話，要用`iterator`的`value`，不能直接拿來用

它另外一個info是包含`key`，也就是用來分割的group value是什麼


```r
str(isplit(1:6, rep(1:2, 3))$nextElem())
```

```
## List of 2
##  $ value: int [1:3] 1 3 5
##  $ key  :List of 1
##   ..$ : int 1
```

```r
foreach(it = isplit(1:6, rep(1:2, 3)), .combine = c, .multicombine = TRUE) %do% {
    sum(it$value)
}
```

```
## [1]  9 12
```

```r
foreach(it = isplit(1:6, rep(1:2, 3))) %do% {
    it$value + 1
}
```

```
## [[1]]
## [1] 2 4 6
## 
## [[2]]
## [1] 3 5 7
```

多組group vector的話，可以這樣做：


```r
splitList <- list(rep(1:2, 3), c(1, 1, 1, 3, 3, 3))
str(isplit(1:6, splitList)$nextElem())
```

```
## List of 2
##  $ value: int [1:2] 1 3
##  $ key  :List of 2
##   ..$ : int 1
##   ..$ : num 1
```

```r
foreach(it = isplit(1:6, splitList), .combine = c, .multicombine = TRUE) %do% {
    sum(it$value)
}
```

```
## [1]  4  2  5 10
```

```r
foreach(it = isplit(1:6, splitList)) %do% {
    it$value + 1
}
```

```
## [[1]]
## [1] 2 4
## 
## [[2]]
## [1] 3
## 
## [[3]]
## [1] 6
## 
## [[4]]
## [1] 5 7
```

但是`isplit`除了搭配`vector`之外，也可以搭配`data.frame`來使用：


```r
DF <- data.frame(x = rep(1:3, 3), y = c(rep(1:2, 4), 3))
foreach(it = isplit(DF, with(DF, list(x = x, y = y)))) %do% {
    print(it$key)
    nrow(it$value)
}
```

```
## $x
## [1] 1
## 
## $y
## [1] 1
## 
## $x
## [1] 2
## 
## $y
## [1] 1
## 
## $x
## [1] 3
## 
## $y
## [1] 1
## 
## $x
## [1] 1
## 
## $y
## [1] 2
## 
## $x
## [1] 2
## 
## $y
## [1] 2
## 
## $x
## [1] 3
## 
## $y
## [1] 2
## 
## $x
## [1] 1
## 
## $y
## [1] 3
## 
## $x
## [1] 2
## 
## $y
## [1] 3
## 
## $x
## [1] 3
## 
## $y
## [1] 3
```

```
## [[1]]
## [1] 2
## 
## [[2]]
## [1] 1
## 
## [[3]]
## [1] 1
## 
## [[4]]
## [1] 1
## 
## [[5]]
## [1] 2
## 
## [[6]]
## [1] 1
## 
## [[7]]
## [1] 0
## 
## [[8]]
## [1] 0
## 
## [[9]]
## [1] 1
```

這裡需要注意的是`isplit`不是現有組合有存在才會取值

而是將你輸入的group vector分別取unique後做`expand.grid`的動作展開

所以會有組合是沒有資料的，要記得在script裡面寫如果零列要怎麼處理：


```r
DF <- data.frame(x = rep(1:3, 3), y = c(rep(1:2, 4), 3))
foreach(it = isplit(DF, with(DF, list(x = x, y = y))), .combine = rbind, .multicombine = TRUE) %do% {
    if (nrow(it$value) == 0) 
        return(NULL)
    data.frame(it$key, sum(it$value))
}
```

```
##   x y sum.it.value.
## 1 1 1             4
## 2 2 1             3
## 3 3 1             4
## 4 1 2             3
## 5 2 2             8
## 6 3 2             5
## 7 3 3             6
```




