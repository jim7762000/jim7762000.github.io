---
layout: post
title: "Locally quantile regression in R"
---

最近稍微找了一下locally quantile regression的資訊

結果發現[Quantile LOESS](https://www.r-statistics.com/2010/04/quantile-loess-combining-a-moving-quantile-window-with-loess-r-function/)這篇可以得到比quantreg更好的效果

不過這方法沒有直覺的方式去選擇bandwidth，所以我就自己加了一點東西進去

結合我開發套件中的`rfda`(`rfda`可以在[這裡](https://github.com/ChingChuan-Chen/rfda)找到)裡面的`locPoly1d`

就可以很自然使用bandwidth去調整線的平滑程度了，程式如下：

``` R
data(airquality)
library(quantreg)
library(ggplot2)
library(data.table)
# source Quantile LOESS
source("https://www.r-statistics.com/wp-content/uploads/2010/04/Quantile.loess_.r.txt")

airquality2 <- na.omit(airquality[ , c(1, 4)])

# quantreg::rq
rq_fit <- rq(Ozone ~ Temp, 0.95, airquality2)
rq_fit_df <- data.table(t(coef(rq_fit)))
names(rq_fit_df) <- c("intercept", "slope")
# quantreg::lprq
lprq_fit <- lapply(1:3, function(bw){
  fit <- lprq(airquality2$Temp, airquality2$Ozone, h = bw, tau = 0.95)
  return(data.table(x = fit$xx, y = fit$fv, bw = paste0("bw=", bw), fit = "quantreg::lprq"))
})
# Quantile LOESS
ql_fit <- Quantile.loess(airquality2$Ozone, jitter(airquality2$Temp), window.size = 10,
                         the.quant = .95,  window.alignment = c("center"))
ql_fit_df <- data.table(x = ql_fit$x, y = ql_fit$y.loess, bw = "bw=1", fit = "Quantile LOESS")
# rfda::locQuantPoly1d
xout <- seq(min(airquality2$Temp), max(airquality2$Temp), length.out = 30)
locQuantPoly1d_fit <- lapply(1:3, function(bw){
  fit <- rfda::locQuantPoly1d(bw, 0.95, airquality2$Temp, airquality2$Ozone, 
                              rep(1, length(airquality2$Temp)), xout, "gauss", 0, 1)
  return(data.table(x = xout, y = fit, bw = paste0("bw=", bw), fit = "rfda::locQuantPoly1d"))
})

graphDF <- rbindlist(c(lprq_fit, list(ql_fit_df), locQuantPoly1d_fit))

ggplot(airquality2, aes(Temp, Ozone)) + geom_point() +
  labs(title = "Predicting the 95% Ozone level according to Temperature", 
       colour = "Methods", linetype = "Bandwidth") + 
  geom_abline(aes(intercept = intercept, slope = slope, colour ="quantreg::rq", 
                  linetype = "bw=1"), rq_fit_df, show.legend = TRUE) +
  geom_line(aes(x, y, colour = fit, linetype = bw), graphDF, show.legend = TRUE)
```

結果圖：![](/images/localQuantileReg.png)
