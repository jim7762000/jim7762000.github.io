---
layout: post
title: R data.table - collating data
---

`data.table` is a powerful tool for collating data. Here we provides a example. Recently, I join a team and participate a competition of R data analysis.

It is an estate data which contains housing price and some variables. I put data in my [Google drive].(https://drive.google.com/open?id=0B1UBN4lCLHrVeWRzQ1FHQUoyem8).

Code:

``` R
library(data.table)
library(dplyr)
dat = lapply(paste0("List_", LETTERS[LETTERS!="L" & LETTERS!="R" & LETTERS!="S" & LETTERS!="Y"], ".csv"), read.csv)
dat2 = rbindlist(dat)
region_v = c("臺北市","臺中市","基隆市","臺南市","高雄市","新北市","宜蘭縣","桃園縣","嘉義市","新竹縣","苗栗縣","南投縣","彰化縣","新竹市","雲林縣","嘉義縣","屏東縣","花蓮縣","臺東縣","金門縣","澎湖縣","連江縣")
region = unlist(lapply(1:length(dat), function(i) rep(region_v[i], nrow(dat[[i]]))))
dat2 = data.table(dat2, region = region)
cols = c(1:3, 5:7, 9:15, 20:21, 24)
dat2[, (cols):=lapply(.SD, as.character),.SDcols=cols]
setnames(dat2, "交易標的", "trading_target")
dat2 = filter(dat2, grepl("建物", trading_target))

setnames(dat2, "交易年月", "trading_ym")
dat2 = mutate(dat2, year.trading = substr(trading_ym, 0, nchar(trading_ym)-2))
dat2 = mutate(dat2, month.trading = substr(trading_ym, nchar(as.character(trading_ym))-1, nchar(trading_ym)))
set(dat2, i = which(dat2$month.trading >= 13), j = "month.trading", value = NA)
set(dat2, i = which(dat2$month.trading == 0), j = "month.trading", value = NA)
set(dat2, i = which(is.na(dat2$month.trading)), j = "month.trading", value = "6")
set(dat2, j = "year.trading", value = as.integer(dat2$year.trading))
set(dat2, j = "month.trading", value = as.integer(dat2$month.trading))

setnames(dat2, "交易筆棟數", "trading_detail")
dat2 = mutate(dat2, n.land = as.integer(substr(trading_detail,3,3)))
dat2 = mutate(dat2, n.building = as.integer(substr(trading_detail,6,6)))
dat2 = mutate(dat2, n.parking_lot = as.integer(substr(trading_detail,9,9)))

setnames(dat2, "主要建材", "building_materials")
dat2 = mutate(dat2, building_materials = as.character(building_materials))
set(dat2, i = which(dat2$building_materials==""), j = "building_materials", value = NA)
dat2 = mutate(dat2, CRC = grepl("混凝土", building_materials))

setnames(dat2, "建築完成年月", "construction_ym")
dat2 = mutate(dat2, year.construction = as.numeric(rep(NA, nrow(dat2))))
dat2 = mutate(dat2, month.construction = as.numeric(rep(NA, nrow(dat2))))

dat2[nchar(construction_ym)==2 & !is.na(construction_ym), year.construction := as.integer(construction_ym)]

dat2[nchar(construction_ym)==4 & substr(construction_ym, 1, 2) != 10 & !is.na(construction_ym), year.construction := as.integer(substr(construction_ym, 1, 2))]
dat2[nchar(construction_ym)==4 & substr(construction_ym, 1, 2) != 10 & !is.na(construction_ym), month.construction := as.integer(substr(construction_ym, 3, 4))]
dat2[nchar(construction_ym)==4 & substr(construction_ym, 1, 2) == 10 & !is.na(construction_ym), year.construction := as.integer(substr(construction_ym, 1, 3))]
dat2[nchar(construction_ym)==4 & substr(construction_ym, 1, 2) == 10 & !is.na(construction_ym), month.construction := as.integer(substr(construction_ym, 4, 4))]

dat2[nchar(construction_ym)==5 & (substr(dat2$construction_ym, 1, 1) == 1 | substr(dat2$construction_ym, 1, 1) == 0) & !is.na(construction_ym), year.construction := as.integer(substr(construction_ym, 1, 3))]
dat2[nchar(construction_ym)==5 & (substr(dat2$construction_ym, 1, 1) == 1 | substr(dat2$construction_ym, 1, 1) == 0) & !is.na(construction_ym), month.construction := as.integer(substr(construction_ym, 4, 5))]
dat2[nchar(construction_ym)==5 & substr(dat2$construction_ym, 1, 1) != 1 & substr(dat2$construction_ym, 1, 1) != 0 & !is.na(construction_ym), year.construction := as.integer(substr(construction_ym, 1, 2))]
dat2[nchar(construction_ym)==5 & substr(dat2$construction_ym, 1, 1) != 1 & substr(dat2$construction_ym, 1, 1) != 0 & !is.na(construction_ym), month.construction := as.integer(substr(construction_ym, 3, 3))]

dat2[nchar(construction_ym)==6 & (substr(dat2$construction_ym, 1, 1) == 1 | substr(dat2$construction_ym, 1, 1) == 0) & !is.na(construction_ym), year.construction := as.integer(substr(construction_ym, 1, 3))]
dat2[nchar(construction_ym)==6 & (substr(dat2$construction_ym, 1, 1) == 1 | substr(dat2$construction_ym, 1, 1) == 0) & !is.na(construction_ym), month.construction := as.integer(substr(construction_ym, 4, 5))]
dat2[nchar(construction_ym)==6 & substr(dat2$construction_ym, 1, 1) != 1 & substr(dat2$construction_ym, 1, 1) != 0 & !is.na(construction_ym), year.construction := as.integer(substr(construction_ym, 1, 2))]
dat2[nchar(construction_ym)==6 & substr(dat2$construction_ym, 1, 1) != 1 & substr(dat2$construction_ym, 1, 1) != 0 & !is.na(construction_ym), month.construction := as.integer(substr(construction_ym, 3, 4))]

dat2[nchar(construction_ym)==7 & (substr(dat2$construction_ym, 1, 1) == 1 | substr(dat2$construction_ym, 1, 1) == 0) & !is.na(construction_ym), year.construction := as.integer(substr(construction_ym, 1, 3))]
dat2[nchar(construction_ym)==7 & (substr(dat2$construction_ym, 1, 1) == 1 | substr(dat2$construction_ym, 1, 1) == 0) & !is.na(construction_ym), month.construction := as.integer(substr(construction_ym, 4, 5))]
dat2[nchar(construction_ym)==7 & substr(dat2$construction_ym, 1, 1) != 1 & substr(dat2$construction_ym, 1, 1) != 0 & !is.na(construction_ym), year.construction := as.integer(substr(construction_ym, 1, 2))]
dat2[nchar(construction_ym)==7 & substr(dat2$construction_ym, 1, 1) != 1 & substr(dat2$construction_ym, 1, 1) != 0 & !is.na(construction_ym), month.construction := as.integer(substr(construction_ym, 4, 5))]

set(dat2, i = which(dat2$year.construction >= 104), j = "year.construction", value = NA)
set(dat2, i = which(dat2$month.construction >= 13), j = "month.construction", value = NA)
set(dat2, i = which(dat2$month.construction == 0), j = "month.construction", value = NA)
set(dat2, i = which(is.na(dat2$month.construction)), j = "month.construction", value = 6)

dat2 = mutate(dat2, trading = year.trading *12 + month.trading)
dat2 = mutate(dat2, construction = year.construction *12 + month.construction)
dat2 = mutate(dat2, age_house = trading - construction)

### output
dat3 = as.data.frame(select(dat2, c(23,29,4,5,39,11,12,40,16:21,25,32:35)))
dat4 = dat3[setdiff(1:nrow(dat3), which(is.na(dat3), arr.ind = TRUE)[,1]),]
names(dat4) = c("Y", paste0("V", 1:(ncol(dat4)-1)))
dat4$V1 = as.factor(dat4$V1)
dat4$V3 = as.factor(dat4$V3)
dat4$V5 = as.numeric(dat4$V5)
dat4$V6 = as.factor(dat4$V6)
dat4$V12 = as.factor(dat4$V12)
dat4$V13 = as.factor(dat4$V13)
dat4$V18 = as.factor(dat4$V18)
dat4 = dat4[dat4$Y!=0,]
dat4 = dat4[setdiff(1:nrow(dat4), which(is.na(dat4), arr.ind = TRUE)[,1]),]

### modeling
lm.fit <- lm(log(Y+0.1)~.,data=dat4)
summary(lm.fit)

dat4.sub<-dat4[-which(rownames(dat4)%in%c(323747, 80936, 222834)),]
lm.fit <- lm(log(Y+0.1)~.,data=dat4.sub)
summary(lm.fit)

library(MASS)
lm.AIC <- stepAIC(lm.fit)

fold = 10
cv_index_f = function(n, fold = 10){
  fold_n = floor(n / fold)
  rem = n - fold_n * fold
  size = rep(fold_n, fold)
  if(rem > 0)
    size[1:rem] = fold_n + 1
  cv_index = unlist(sapply(1:fold, function(i) rep(i, size[i])))
  cv_index = sample(cv_index, length(cv_index))
  return(cv_index)
}
index = cv_index_f(nrow(dat4.sub), fold)

library(snowfall)
n_rep = 10
sfInit(TRUE, 8)
sfExport("dat4.sub")
pred.error = sapply(1:n_rep , function(i){
  index = cv_index_f(nrow(dat4.sub), fold)
  sfExport("index")
  lm.CV <- sfSapply(1:fold, function(v){
    dat4.train = dat4.sub[index != v,]
    dat4.test = dat4.sub[index == v,]
    lm.fit.train = lm(log(Y+0.1)~.,data = dat4.train)
    sum((log(dat4.test$Y+0.1) - predict(lm.fit.train, dat4.test))^2)/nrow(dat4.test)
    })
  mean(lm.CV)
})
sfStop()
mean(pred.error)
# 0.1912725
pred.error
# [1] 0.1913776 0.1912849 0.1912692 0.1913275 0.1914682 0.1914266 0.1911751 0.1911378 0.1911330 0.1911254


library(grpreg)
X.m <- model.matrix(log(Y+0.1)~. , data=dat4.sub)[,-1]
group.v <- substring(colnames(X.m),1,2)
group.v[41:49] <- substring(colnames(X.m),1,3)[41:49]
group.v <- as.numeric(factor(group.v, levels=paste("V",c(1:18),sep="")))
out.glasso <- cv.grpreg(X.m, log(dat4.sub$Y+0.1), group=group.v)
plot(out.glasso)
out.glasso$lambda.min
# 0.0001378642
out.glasso$cve[out.glasso$min]
# 0.1911332
coef(out.glasso)
```

Thanks to this competition, I have an interesting data to practice the use of `data.table`. In addition, `data.table` and `dplyr` are so fast that I drop `data.frame`.

My environment is ubuntu 14.04, R 3.1.1 compiled by intel c++, fortran compiler with MKL. My CPU is 3770K@4.3GHz.





