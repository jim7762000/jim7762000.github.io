---
layout: post
title: Faster R in windows
---

R的BLAS庫效率不彰，在linux上可以透過更換成openBLAS來加速，或是compiled with intel MKL，在windows上compile R是一個痛苦的過程。

因此，有人提供這方面的資源，最有名的就是Revolution，Revolution是compiled with intel MKL，但是要錢。但是天無絕人之路，總是有其他辦法的。

1. 用Revolution R Open，你可以在[官方網站](http://www.revolutionanalytics.com/revolution-r-open)下載到。

1-2.

如果討厭RRO的猴子圖案，可以把RRO/bin/x64中的`libiomp5md.dll`, `RBlas.dll`, `Rlapack.dll`這三個檔案複製到R/bin/x64取代原本的檔案。

2. 更換BLAS庫，網路上有人提供GotoBLAS2編譯的RBlas.dll，[連結在此](http://prs.ism.ac.jp/~nakama/SurviveGotoBLAS2/binary/windows/x64/)，win32的部分，可以參考[CRAN](http://cran.r-project.org/bin/windows/contrib/ATLAS/)，下載相對應CPU的`RBlas.dll`然後替換掉R/bin/x64 (or i386)的`RBlas.dll`即可享受快速的BLAS帶來的效能。

3. 至於OpenBLAS的部分則參考[此連結](http://www.douban.com/note/296114898/?start=0&post=ok#last)，這個方法比較複雜一點。

總結：個人測試這三個BLAS都差不多快，不會差太多，自己選擇喜歡的使用即可。但是有人回報用第三個方案常常會導致R崩潰，可能是因為沒有經過R的編譯之緣故。
