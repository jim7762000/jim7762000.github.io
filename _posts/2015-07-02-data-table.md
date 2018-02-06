---
layout: post
title: introduction to data.table
---

這是我自己在PTT PO的文，詳細介紹data.table，以下是正文~~

data.table包含的東西很多

但是很多東西都可以被plyr, dplyr的function取代

所以data.table很多function，我都不太熟

這裡簡單介紹一下data.table

如果你想要了解更多，請自行去看manual

要了解data.table，我們可以先從package的description來看

  "Fast aggregation of large data (e.g. 100GB in RAM), fast ordered joins,
fast add/modify/delete of columns by group using no copies at all, list
columns and a fast file reader (fread). Offers a natural and flexible syntax,
for faster development."

簡單翻譯一下，大資料(例如，記憶體中大小為100GB的資料)的在不創建複本下，根據

類別(group)變數進行快速整合、排列、合併、增加/修改/刪除行資料等動作。...

重點就在不創建複本，因為R修改data.frame時，會先複製一次再修改，

然後傳回複本，因此，會浪費不少記憶體，而且很容易拖累速度，因此，

data.table提供這方面更有效率的操作。

(這方面的速度比較可以參考#1LeXNCKV (R_Language) [分享] 資料數據處理修改)


1. data.table

這個函數基本上data.frame使用差不多，而且data.frame的參數都可以放進

像是很常用到的stringsAsFactors，只是data.table預設是FALSE，

這點跟data.frame不同，使用上需要注意，範例如下：

``` R
  t = data.table(a = LETTERS[1:3])
  str(t)
# Classes ‘data.table’ and 'data.frame':  3 obs. of  1 variable:
#  $ a: chr  "A" "B" "C"
#  - attr(*, ".internal.selfref")=<externalptr>
  t2 = data.frame(a = LETTERS[1:3])
  str(t2)
# 'data.frame':   3 obs. of  1 variable:
#  $ a: Factor w/ 3 levels "A","B","C": 1 2 3
```

第二個差異是data.table不包含rownames，

在轉換data.frame到data.table時，要注意這點

下一章會提到把rowname轉成column的函數

附註一條：data.table都包含data.frame的class

          可以用在data.frame的方法都可以在data.table上實現


但是data.table還多了一個引數 "key"，我對它的解讀是一種索引的概念

而透過索引的動作都會被加速。

key可以是一個變數，也可以是多個變數，這點看個人使用。


再來，就是data.table的'['，這部分跟data.frame不太一樣

所以需要特別說明，但是這部分，我自己也不是很熟悉，我只能大概講過

a. 我們很常在data.frame做取多行的動作，在data.table是不可行的，舉例：

``` R
  vars = data.frame(X = rnorm(3), Y = rnorm(3), Z = rnorm(3))
  vars[,1:2]
#            X         Y
# 1 -0.5677575 2.1831285
# 2 -0.7161529 0.3714633
# 3  1.2665120 0.7837508

  vars_dt = data.table(vars)
  vars_dt[,1:2]
# [1] 1 2
```

但是你想這麼做，怎麼辦？ 加上with=FALSE就好了，或是用list包住column name

``` R
  vars_dt[,1:2,with=FALSE]
#             X         Y
# 1: -0.5677575 2.1831285
# 2: -0.7161529 0.3714633
# 3:  1.2665120 0.7837508

  vars_dt[j=list(X, Y)]
#             X         Y
# 1: -0.5677575 2.1831285
# 2: -0.7161529 0.3714633
# 3:  1.2665120 0.7837508
```

剩下像是by, .SD, .SDcols等自行?data.table查看吧

data.table的部分就先說明到這，接下來，講一些相關的function

b. setkey: 改變key的值, setnames: 改變column name，但是一樣不製造複本

c. copy: 製造data.table的複本

d. setDF: 在不製作複本下，把data.table的class改為data.frame

舉例：

``` R
  DT = data.table(X = rnorm(3), Y = rnorm(3))
  str(DT)
# Classes ‘data.table’ and 'data.frame':  3 obs. of  2 variables:
#  $ X: num  -1.3738 0.167 -0.0578
#  $ Y: num  0.487 1.728 0.646
#  - attr(*, ".internal.selfref")=<externalptr>

  setDF(DT)
  str(DT)
# 'data.frame':   3 obs. of  2 variables:
#  $ X: num  -1.3738 0.167 -0.0578
#  $ Y: num  0.487 1.728 0.646

  DT = data.table(X = rnorm(3), Y = rnorm(3))
  tracemem(DT)
# [1] "<0000000006A1BE28>"
  setDF(DT)  #  沒有複製的動作

  DF = data.frame(DT)
  retracemem(DF, retracemem(DT))
# tracemem[<0000000006A1BE28> -> 0x00000000061ec928]:
  ## 記憶體位置就發生改變了，就複製了DT一次
```

這部分可能不太懂，不過沒關係，記住一點，要轉成data.frame用setDF就好

e. setDT: setDF的反向

f. duplicated, unique

duplicated提供一個跟data.table列數相等長度的邏輯值向量，

TRUE代表前面有一樣的列，FALSE代表沒有

unique則是留下沒有重複的列，舉例來說：

``` R
  set.seed(100)
  DT = data.table(A = rbinom(5, 1, 0.5), B = rbinom(5, 1, 0.5))
#    A B
# 1: 0 0
# 2: 0 1
# 3: 1 0
# 4: 0 1
# 5: 0 0

  duplicated(DT)
# [1] FALSE FALSE FALSE  TRUE  TRUE

  unique(DT)
#    A B
# 1: 0 0
# 2: 0 1
# 3: 1 0

  DT[!duplicated(DT)]
#    A B
# 1: 0 0
# 2: 0 1
# 3: 1 0
```

不過unique還有更多功能，它可以選擇變數做unique，舉例來說：

``` R
  unique(DT, by = "A")
#    A B
# 1: 0 0
# 2: 1 0

  unique(DT, by = "B")
#    A B
# 1: 0 0
# 2: 0 1
```
順便一提，dplyr的distinct，如果你input的class是data.table

它就是用unique做的

``` R
  library(dplyr)
  distinct(DT)
#    A B
# 1: 0 0
# 2: 0 1
# 3: 1 0
```
你如果想看distinct怎麼做，可以在R上面打dplyr:::distinct_.data.table

> dplyr:::distinct_.data.table
function (.data, ..., .dots)
{
    dist <- distinct_vars(.data, ..., .dots = .dots)
    if (length(dist$vars) == 0) {
        unique(dist$data)
    }
    else {
        unique(dist$data, by = dist$vars)
    }
}

之後提到distinct，我們再來講distinct

其他相關function像是subset, setcolorder, setorder (setorderv)

對這三個function有興趣，再去看manual，不贅述

這三個對應到dplyr的filter, select, arrange，之後我們會再提到這些


g. transform: 改變column的屬性、值等，舉例來說：


``` R
  DT = data.table(a = 1:3, b = 2:4, c = LETTERS[1:3])
  DT2 = copy(DT)
  DT[, b := b**2]
  DT2 %<>% transform(b = b**2)
  all.equal(DT, DT2) # TRUE
  DT %<>% transform(c = as.factor(c))
  str(DT)
# Classes ‘data.table’ and 'data.frame':  3 obs. of  3 variables:
#  $ a: int  1 2 3
#  $ b: num  4 9 16
#  $ c: Factor w/ 3 levels "A","B","C": 1 2 3
#   - attr(*, ".internal.selfref")=<externalptr>
```

h. set: 用來變更特定column，某些列的值，舉個簡單的例子

``` R
  DT = data.table(a = 1:3, b = 2:4)
  DT2 = copy(DT)
  DT[, b := 1]
  set(DT2,, "b", value = 1)
  all.equal(DT, DT2) # TRUE
```
一般來說都用'['來做，但是你如果需要用到for再來完成，再用set


還有一個function是 J，這裡就不提了，一樣請洽manual

最後，還有一個operator，':='，它是用來擴增data.table的column，

同樣，也不創造複本，這樣可以更快的增加column

那如果刪除怎麼辦？還記得前面學過 DT[, list('X', 'Y')]，就用這個


再來，我們講一些data.table中其他function

2. fread

功能可以用來取代read.table, read.csv

它可以用多種separate去分割columns，然後讀入R

而且讀入速度比read.table, read.csv快很多

但是注意，不規則的檔案會讀入失敗


這裡提幾個參數：

a. sep: column跟column之間的分隔，如果是csv就是','，

   如果是tab separated values就是'\t'

b. na.strings: 視作NA的字串，它可以是一個vector

c. stringsAsFactors：是否要把字串轉成factor，預設是否

d. colClasses：各行的classes，可以自行設定

我愛用fread還有一個原因，第一個input可以直接放我要讀的字串，

但是read.table需要經過其他的方式，有點麻煩(我懶得記，其實沒記過)

舉例來說

``` R
text = "a b
1 2
3 4"
DT = fread(text)
setDF(DT) # 轉成data.frame，前面學過，還記得嗎？
DF = read.table(header = TRUE, text = text)           # text format
DF2 = read.table(textConnection(text), header = TRUE) # file format

all.equal(DT, DF)  # TRUE
all.equal(DT, DF2) # TRUE
```

fread很適合拿來讀大資料，所以有必要把table輸出成text

用文字方式處理時，讀入就變得很方便，可見 #1LegOjwB (R_Language)

還剩下 dcast.data.table, melt 跟 merge

它們會留到之後跟tidyr一起介紹

下一章重點會放在dplyr


補充：

key，我也不是很熟悉，也很少用，因此，我這裡介紹的很少

如果對key有興趣，可能需要自行研究