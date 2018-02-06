---
layout: post
title: Transform Variables of R data.frame
---

有人在PTT問說，怎樣轉換R data.frame的多行column，連結：[PTT文章](https://www.ptt.cc/bbs/R_Language/M.1492142292.A.C75.html)

提醒：下面程式沒有判斷factor，如有factor請自行加入判斷式

``` R
# data.table做法：
library(data.table)
DT[ , lapply(.SD, function(x)iconv(x,"UTF8", "BIG5"))]
 
# 如果有numeric或是integer column的話：
DT[ , lapply(.SD, function(x){
  if (is.character(x))
    iconv(x,"UTF8", "BIG5")
} else return(x)})]
 
 
# dplyr做法：
library(dplyr)
DF %>% mutate_each(funs(iconv(., "UTF8", "BIG5")))
 
# 如果有numeric或是integer column的話：
DF %>% mutate_if(is.character, funs(iconv(., "UTF8", "BIG5")))    
 
# base函數解法：
evalExpr <- lapply(names(DF), function(x){
  bquote(iconv(.(as.symbol(x)), "UTF8",  "BIG5"))
})
do.call(function(...) transform(DF, ...), evalExpr)
 
# 如果有numeric或是integer column的話：
evalExpr <- lapply(names(DF)[sapply(DF, is.character)],
                   function(x) bquote(iconv(.(as.symbol(x)), "UTF8",  "BIG5")))
do.call(function(...) transform(DF, ...), evalExpr)
```

當然也可以直接用column name帶入去做轉換：

``` R
lapply(names(DF), function(x){
   iconv(DF[[x]], "UTF8",  "BIG5")
})

# 或是

lapply(names(DF)[sapply(DF, is.character)], function(x){
   iconv(DF[[x]], "UTF8",  "BIG5")
})
```

只是除了要多轉一次data.frame之外，第二個還要把numeric, integer column併回去  
