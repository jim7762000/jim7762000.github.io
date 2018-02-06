---
layout: post
title: "wrapr範例"
---

原文是我在PTT回的一篇文章[連結](https://www.ptt.cc/bbs/R_Language/M.1489058786.A.745.html)

其實是有人來問怎麼將character verctor放入函數中執行對應的動作

好讓函數變得簡單使用跟操作

那我提供了下面幾個解法：

``` R
library(dplyr)
library(pipeR)
library(ggplot2)
library(data.table)

data("diamonds", package = "ggplot2")

# 一般寫法 (dplyr)
df_group_fn <- function(df, meanCol, col_1, col_2){
  df %>>% group_by_(.dots = c(col_1, col_2)) %>>%
    summarise_(.dots = c(n = "n()",
                         mean = paste0("mean(", meanCol, ")"))) %>>%
    {ggplot(., aes(mean,n)) + geom_point()}
}
df_group_fn(diamonds, "price", "cut", "color")

# 一般寫法 (data.table)
dt_group_fn <- function(dt, meanCol, col_1, col_2){
  dt[ , .(n = .N, mean = eval(parse(text = paste0("mean(", meanCol, ")")))),
     by = c(col_1, col_2)] %>>%
  {ggplot(., aes(mean,n)) + geom_point()}
}
dt_group_fn(data.table(diamonds), "price", "cut", "color")

# wrapr + dplyr
library(wrapr)
df_group_fn2 <- function(df, meanCol, col_1, col_2){
  let(list(y = meanCol, c1 = col_1, c2 = col_2), {
    df %>>% group_by(c1, c2) %>>% summarise(n = n(), mean = mean(y))
  }) %>>% {ggplot(., aes(mean,n)) + geom_point()}
}
df_group_fn2(diamonds, "price", "cut", "color")

# wrapr + data.table
dt_group_fn2 <- function(dt, meanCol, col_1, col_2){
  let(list(y = meanCol, c1 = col_1, c2 = col_2), {
    dt[ , .(n = .N, mean = mean(y)), by = .(c1, c2)]
  }) %>>% {ggplot(., aes(mean,n)) + geom_point()}
}
dt_group_fn2(data.table(diamonds), "price", "cut", "color")

# 進階，不把欄位給死的方法：
# dplyr
df_group_fn3 <- function(df, meanCol, groupByCols){
  let(list(y = meanCol), {
    df %>>% group_by_(.dots = groupByCols) %>>%
      summarise(n = n(), mean = mean(y))
  }) %>>%
  {ggplot(., aes(mean,n)) + geom_point()}
}
df_group_fn3(diamonds, "price", c("cut", "color"))

# data.table
dt_group_fn3 <- function(dt, meanCol, groupByCols){
  let(list(y = meanCol), {
    dt[ , .(n = .N, mean = mean(y)), by = groupByCols]
  }) %>>% {ggplot(., aes(mean,n)) + geom_point()}
}
dt_group_fn3(data.table(diamonds), "price", c("cut", "color"))
```

但是最漂亮的寫法應該是下面這樣：

``` R
# data.table + ... + substitute
dt_group_fn3 <- function(dt, meanCol, ...){
  groupByCols <- as.character(as.list(substitute(list(...)))[-1L])
  y <- substitute(meanCol)
  dt[ , .(n = .N, mean = mean(y)), by = groupByCols] %>>%
    {ggplot(., aes(mean,n)) + geom_point()}
}
dt_group_fn3(data.table(diamonds), price, cut, color)
```

