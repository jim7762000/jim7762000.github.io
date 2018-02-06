---
layout: post
title: "R web service with opencpu"
---

昨天後來再google發現opencpu這個更強大的套件

昨天太累就沒試了XD

我把前幾天弄得`unnest`函數在配上opencpu

就可以輕鬆做到unnested json的轉換了


先列參考來源：

1. [Taiwan R User Group - Opencpu sharing](https://docs.google.com/presentation/d/1lz8W61Vihwm-BfbcGO60MkXUVOw-bTSdYi4YyystIp8)

其實投影片講的非常清楚

照著做就可以出來只是路徑要看清楚XDD (因為路徑少打R/試了超久)

首先，安裝`opencpu`，然後安裝`devtools`

``` R
# 安裝我的套件
devtools::install_github("ChingChuan-Chen/test.opencpu")
# 啟動opencpu
library(opencpu) # library之後會出現下面的訊息，http://localhost:8850/ocpu 就是你可以連過去做測試的網址
# Initiating OpenCPU server...
# Using config: C:/Users/Celest/Documents/.opencpu.conf
# OpenCPU started.
# [httpuv] http://localhost:8850/ocpu
# OpenCPU single-user server ready.
```

我的測試檔案：[GitHub](https://github.com/ChingChuan-Chen/rfda/blob/master/inst/extdata/funcData.json)

原始資料長相：![](/images/opencpu1.PNG)

opencpu web UI操作POST：![](/images/opencpu3.PNG)

回傳的結果：![](/images/opencpu2.PNG)


但是這樣還不夠完美，所以可以透過回傳的內容有連結資訊

(由上角小框框裡面是POST回傳的結果)

來用R來取得，其程式碼就會像下面這樣：

``` R
library(pipeR)
library(httr)
library(stringr)
library(jsonlite)
POST("http://localhost:8850/ocpu/library/test.opencpu/R/convertJSON", 
     body = list(input = upload_file("funcdata.json")) %>>%
  content("text") %>>% str_split("\r\n") %>>% `[[`(1) %>>% `[`(1) %>>% 
  str_c("/json") %>>% sprintf(fmt="http://localhost:8850%s") %>>% GET %>>% 
  content("text") %>>% fromJSON
#    sampleID    t       y
# 1         1 9.37 -0.0515
# 2         1 4.06 -0.3743
# 3         1 8.92  1.2116
# 4         1 8.83  0.4139
# 5         1 1.27  1.7179
# 6         1 5.86 -1.3720
# 7         1 4.69 -2.4096
# 8         1 9.55 -0.7041
# 9         1 2.62  1.9196
# 10        1 7.93  1.5362
# 11        1 4.42 -0.1172
# 12        1 8.38  0.1368
# 13        1 9.19 -0.9449
# 14        1 4.33 -1.5905
# 15        1 9.73 -0.3867
```

最後放一張完成圖：![](/images/opencpu2.PNG)
