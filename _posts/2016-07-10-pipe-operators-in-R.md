---
layout: post
title: "Pipe Operators in R"
---

我在ptt分享過magrittr的文章([連結](https://www.ptt.cc/bbs/R_Language/M.1437452331.A.CD1.html))，做為資料處理系列文章的第一篇

後來有一些額外的心得([連結](https://www.ptt.cc/bbs/Statistics/M.1450534933.A.8A4.html))，所以又有一篇補充了一些觀念

一個大陸人Kun Ren(任堃)後來在2015年上傳了一個pipeR套件

宣稱比magrittr的pipe operator更好用，效能更佳，而且沒有模糊的界線(他的文章連結：[Blog Post](https://renkun.me/blog/2014/08/08/difference-between-magrittr-and-pipeR.html))

效能差異大概是快了近8倍，後面會在提及這方面

## magrittr

### 背景介紹

先介紹一下magrittr的來由，根據這篇[blog文章](http://www.r-statistics.com/2014/08/simpler-r-coding-with-pipes-the-present-and-future-of-the-magrittr-package/)所提到的：

> The `|>` operator in `F#` is indeed very simple: it is defined as let `(|>) x f = f x`. However, the usefulness of this simplicity relies heavily on a concept that is not available in `R`: partial application. Furthermore, functions in `F#` almost always adhere to certain design principles which make the simple definition sufficient. Suppose that `f` is a function of two arguments, then in `F#` you may apply `f` to only the first argument and obtain a new function as the result — a function of the second argument alone.

`F#`的pipe operator, `|>`, 讓他覺得非常的方便以及實用，因此，決定把這個東西實作到R裡面

(注：其實shell script也有pipe operator, `|`, 而且更廣為人知。)

這就是`R`為什麼會有`magrittr`這個套件的背景原由了，所以才會有了`%>%`, `%<>%`跟`%T>%`這幾個operator。

(注：我這裡補充一點，其實此時開發`ggplot2`, `plyr`, `dplyr`等套件的Hadley已經在`dplyr`上

推出了`%.%`這個operator，而且也把`dplyr`的函數全部都設計第一個放`data.frame`做為input，

只是`magrittr`的推出，把整個功能用到更加完善，並且還涵蓋了alias function，

因此，Hadley後來加入了`magrittr`的發展計畫，就未繼續開發`%.%`，

在Hadley後面發展的套件都可以看到他都有直接把`%>%` import在他的套件中，

例如：很好用的爬蟲套件`rvest`、開發用來處理list的`purrr`等都可以看到`magrittr`的身影。)



### pipe operator的基本概念

假設現在有一個函數`f`，其input為`a`，一般在寫程式的時候，

我們都可以透過`f(a)`獲得`f`函數在`a`這一點的值

而`%>%`可以把LHS(注：left hand side of operator)當作RHS(注：right hand side of operator)的函數的第一個input

因此，透過 a %>% f 或是 a %>% f() 都等同於 f(a)

同理，如果現在還有一個函數`g`，有兩個input，`x`跟`y`，則`g(x,y)`可以寫做`x %>% g(y)`

這樣的好處是當我們有層層推疊的`(`時，就可以輕易地拆解開來並執行

例如：

``` r
a_list <- list(1:6, 3:5, 4:7)
sort_uni_a <- sort(unique(unlist(a_list)))
```

第二行這樣連在一起非常的難讀，但是改成下面這樣，又有效能問題(暫存變數)：


``` r
unlist_list <- unlist(a_list)
sort_uni_a <- sort(unique(unlist_list))
```

或是改成這樣，卻沒覺得可讀性上升多少：


``` r
sort_uni_a <- sort(unique(unlist(a_list)))
```

但是用`%>%`改寫就可以兼顧可讀性跟效能問題：


``` r
sort_uni_a <- a_list %>% unlist %>% unique %>% sort
```

這樣就可以很容易知道他把`a_list`先拉成一個向量(`unlist`)，然後取唯一(`unique`)

接著排列所有元素(`sort`)，照著`%>%`的順序去讀就可以順利解讀

此時，`magrittr`的特性就可以被看到，他省去了暫存變數，也增加了易讀性。



### %>%介紹

舉一個簡單的例子，來說明`%>%`的用法


``` r
a <- 1
f <- function(a) a + 1
f(a)
a %>% f
a %>% f()
a %>% f(.)
```

跑上面的程式可以發現，最後四個output都一樣

其實`%>%`做的就是把左邊變數放進右邊函數裡做執行

也就是說`f(a)`等同於`a %>% f`(或是上面其他三種)

但是怎麼最後一個`a %>% f(.)`看起來怪怪的，為啥有個`.`在那

其實在`magrittr`中，`.`是用來代表`%>%`前面的變數

所以`a %>% f(.)`程式會把`.`的位置換成`a`，變成`f(a)`

`.` 在`magrittr`的應用中，會佔很大的比例，也在Hadley往後的套件扮演重大的角色

像是`do.call`, `Reduce`第一個input是`function`，第二個是`list`

我們通常傳入`list`，所以此時必須用`.`做輸入位置的控制

再者，`c`, `cbind`, `rbind`會根據位置不同來決定是合併於何處

也是一個很重要的問題，因此，用 `.`做傳入位置的控制是必須的

針對這個，我給一段簡單的程式碼來示範：


``` r
a_list <- list(1:5, 3:7, 6:10)
a_list %>% do.call(rbind, .)
a_list %>% Reduce(cbind, .)

1:5 %>% rbind(3:7, .)
1:5 %>% rbind(., 3:7)

f <- function(x, a, b) a * x^2 + b
1:5 %>% f(., 2, 5)  # 同 1:5 %>% f(2, 5)
1:5 %>% f(2, ., 5)
1:5 %>% f(2, 5, .)
```

但是，這點被Kun Ren攻擊，這裡有一些模稜兩可的問題，後面在提。

先介紹一下`{}`在`%>%`的用法，用`{}`括住之後，

裡面的只要不是其他`%>%`後面的 `.`都代表你前面傳入的值

這樣說有點難懂，舉個例子


``` r
1:2 %>% {
    list(cbind(9:10, .), 3:4 %>% cbind(9:10, .))
}
```

```
## [[1]]
##         .
## [1,]  9 1
## [2,] 10 2
## 
## [[2]]
##         .
## [1,]  9 3
## [2,] 10 4
```

可以看到第一個可以很直覺的解讀，`9:10`是跟傳入的`1:2`做行合併

而第二個`.`，因為前面有了一個新的`%>%`

所以這一個`.`就被前面的`3:4`取代

所以第二個output變成`9:10`跟`3:4`做行合併

回到Kun Ren攻擊的點



``` r
f <- function(x, y, z = "nothing") {
    cat("x =", x, "\n")
    cat("y =", y, "\n")
    cat("z =", z, "\n")
}
```

`1:10 %>% f(mean(.), median(.))`到底是

`f(1:10, mean(1:10), median(1:10))`還是`f(mean(1:10),median(1:10))`？

你可能會根據上面的rule很快回答出來是`f(1:10, mean(1:10), median(1:10))`

但是`a_list %>% do.call(rbind, .)`這個呢？

剛剛`1:10 %>% f(mean(.), median(.))`不是`f(1:10, mean(1:10), median(1:10))`這個嗎？

怎麼此時`a_list %>% do.call(rbind, .)`又是`do.call(rbind, a_list)`？

還有剛剛看到的`1:5 %>% rbind(3:7, .)`這個呢？

剛剛`1:5 %>% rbind(3:7, .)`是`rbind(1:5, 3:7, 1:5)`還是`rbind(3:7, 1:5)`這個嗎？

這是`magrittr`為了一些函數做了特例，讓整個rule出現了一些模糊的情況

但是，通常也不會去寫成`a_list %>% {do.call(rbind, .)}`這樣去避免模糊情況(`{}`後面說明)

不過這個方式不能說不好，只是`pipeR`提供了更明確的方式去避免更複雜的方式使用`do.call`跟`Reduce`


### %<>%, %T>%跟%$%介紹

如果懂了`%>%`， 這個就不難了

先看簡單的例子 (`add`是`magrittr`提供用在`%>%`上的`+` (這部分請看最後面的補充))


``` r
a <- 1
a %>% add(1)  # 同 a %>% '+'(1) or a %>% '+'(., 1)
a  # 1
a %<>% add(1)
a  # 2
```

這個例子可以看的出來`%<>%`除了傳入變數之外，也會改變傳入變數的值

也就是可以把`a %<>% add(1)`看成`a = a + 1`

你如果有一串要做最後賦值給你傳入的變數

只需要在第一個傳導變數的operator做改變即可，舉例來說：


``` r
dat <- data.frame(a = 1:3, b = 8:10)
dat <- dat %>% rbind(dat)
dat2 <- data.frame(a = 1:3, b = 8:10)
dat2 %<>% rbind(dat2)
all.eqaul(dat, dat2)  # TRUE
```

至於`%T>%`，他只傳遞變數，不回傳值，通常用來傳遞到不回傳值的`function`上

像是`plot`, `library`, `install.packages`, `plyr`的`*_ply`等

這個operator可以幫你把前面做好的值賦予一個變數

並且同時做後面function的動作，舉例來說：


``` r
dat <- data.frame(a = rep(1:3, 2), b = rnorm(6))
dat2 <- dat %>% {
    tapply(.$b, .$a, sum)
} %>% {
    data.frame(a = names(.) %>% as.integer, b = .)
} %T>% plot(.$a, .$b)
```

這裡`dat2`就是一個新的`data.frame`，同時，我們也把`a, b`的scatter plot畫出來

這部分可以用`dplyr`的`group_by`以及`summarise`完成

還沒提到`dplyr`，所以我們先用替代方法做

再來是第四個operator `%$%`

舉個例來說，雖然有了`%>%`，但是`dat %>% {tapply(.$b, .$a, sum)}`還是會覺得冗長

而且也容易忘記要放`.$`，此時`%$%`就提供了直接把前面變數的元素直接以名字做操作

再也不需要`.$name`這麼麻煩，直接用`name`做你想要的操作就好

所以，就可以簡單寫成`dat %$% tapply(b, a, sum)`

是不是就變得簡單的很多？

再給一個例子說明`%$%`就好


``` r
a <- 3
b <- -2
x <- rnorm(100)
y <- a + b * x + rnorm(100)
fit <- lm(y ~ x)
sigma_hat <- fit %$% {
    crossprod(residuals)/df.residual
}
```

### magrittr一些補充

`magrittr`提供很多其他`function`的別名

像是`+`, `*`, `[`, `[[`, `<- rownames()`等等

有興趣請去`magrittr`的manual查看`extract`的部分

這個可以讓你寫pipe chain的時候更加順手，像是


``` r
vals <- 1:3 %>% data.frame(a = ., b = .^2) %>% set_rownames(LETTERS[1:3]) %>% lm(b ~ a, data = .) %>% predict
```

不然你可能會這樣寫


``` r
dat <- 1:3 %>% data.frame(a = ., b = .^2)
rownames(dat) <- LETTERS[1:3]
vals = dat %>% lm(b ~ a, data = .) %>% predict
```

你可能只是要`vals`這個變數，你卻還要多創一個`dat`這個暫存變數，而中斷chain

## pipeR

這部分是未在ptt上分享過的部分，我主要根據[pipeR Tutorial](http://renkun.me/pipeR/)的內容作介紹

我先說明差異，`pipeR`提供了另一個pipe operator, `%>>%`, 其中最重要的改進就是上面說的模糊問題

強制LHS一定是RHS的第一個input

例如：

`1:5 %>>% rbind(3:7)`是`rbind(1:5, 3:7)`

`1:5 %>>% rbind(3:7, .)`是`rbind(1:5, 3:7, 1:5)`

`1:5 %>>% rbind(., 3:7)`是`rbind(1:5, 1:5, 3:7)`

那`a_list %>% do.call(rbind, .)`怎麼辦

`pipeR`設定只要寫上第一個的parameter name之後，他就會自動傳到第二個

舉例來說，`a_list %>>% do.call(what = rbind)`，他就會看成`do.call(rbind, a_list)`

因為第一個parameter的名字已經被用了，所以會自動傳到第二個參數去

或像是tutorial裡面提到的`lm`函數，打`mtcars %>>% lm(formula = mpg ~ cyl + wt)`

就會等於`lm(formula = mpg ~ cyl + wt, mtcars)`，其中的`mtcars`就會自動放在第二個`data`的input

這樣就規則就會沒有模糊問題，而且不需要再用`{}`去界定位置了



### () in pipeR

再來是一些新功能，`pipeR`提供了`()`去使用單行命令，做指定位置的動作

不需要再開block (`{}`)去產生新的`environment`，可以增進效率

例如上面的`a_list %>>% do.call(what = rbind)`也可以寫成

`a_list %>>% (do.call(rbind, .))`或像是`lm`的例子也可以寫成

`mtcars %>>% (lm(mpg ~ cyl + wt, .))`，至於要用上面的方式還是這裡的模式去寫

就看個人的coding習慣了，基本上，我個人是偏好前面那一種，第二種會多出太多`()`

而且`()`在`pipeR`還會有其他功能，在讀程式時，有時候會有點搞混，不建議用`()`

`()`在`pipeR`的另一個用途就是side effect，下面就來介紹他


### side effect in pipeR

那什麼是side effect?

如果想要在pipe的過程中，想要作指派、輸出一些資訊

而輸出一些資訊就跟`%T>%`的功能一樣，但指派就是新東西了

所有side effect的指令都需要用`(~`跟`)`包起來

例如：


``` r
a_list <- list(1:5, 3:7, 6:10)
sort_uni_a <- a_list %>>% unlist %>>% (~cat("what is it? show me:\n", .)) %>>% unique %>>% sort
```

這樣一來就可以在pipe的過程中把途中的變數show出來

這樣有兩個好處，一個是不需要在額外print東西出來，可以直接pipe，另一個則是方便debug

接著是指派，舉個例子，你需要把a取unique排序後，去掉全資料的平均

以往的寫法會像是：


``` r
a_list <- list(1:5, 3:7, 6:10)
a_list_to_vec <- a_list %>>% unlist
mean_list_a <- mean(a_list_to_vec)
sort_uni_demean_a <- a_list_to_vec - mean_list_a
```

這樣寫就需要兩個暫存變數，`a_list_to_vec`跟`mean_list_a`，而且不能一路pipe

那改用side effect之後，就可以寫成這樣：


``` r
a_list <- list(1:5, 3:7, 6:10)
sort_uni_demean_a <- a_list %>>% unlist %>>% (~mean_list_a <- mean(.)) %>>% unique %>>% sort %>>% -mean_list_a
```

至於`%>%`跟`%>>%`的效能比較可以看下面這段程式(例子取自前面提及的Kun Ren部落格文章)：


``` r
library(magrittr)
library(pipeR)
library(microbenchmark)
eq_f <- `==`
microbenchmark(magrittr = {
    lapply(1:10000, function(i) {
        sample(letters, 6, replace = T) %>% paste(collapse = "") %>% eq_f("rstats")
    })
}, pipeR = {
    lapply(1:10000, function(i) {
        sample(letters, 6, replace = T) %>>% paste(collapse = "") %>>% eq_f("rstats")
    })
}, times = 20L)
```

```
## Unit: milliseconds
##      expr       min       lq      mean   median        uq       max neval
##  magrittr 1215.1390 1253.883 1282.4720 1279.271 1304.1553 1400.9079    20
##     pipeR  377.7212  386.284  395.8731  393.892  406.5949  418.6119    20
```

可以看出效能改進相當顯著，大概快了3倍，隨著loop次數增加

效能改進的倍率還會再增加，這裡就留給有興趣的人自己測試了

而`pipeR`還有兩個函數`Pipe`跟`pipeline`，不過我就沒研究了，有興趣的人在自己看吧

以上是我這篇文章的分享
