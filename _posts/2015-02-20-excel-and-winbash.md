---
layout: post
title: 利用winbash快速複製excel檔案(逐月)，並在excel依檔名變更日期
---

寫了一個bat 去複製檔案~~~
緣由：
這個單位每天都要用excel做日報表
一般作法是複製改裡面的日期，一次做大概半年用
這種事情相當耗時...

改進方式：
1. EXCEL可以根據呼叫該檔檔名，根據該檔檔名去變動日期
EX: 1040216日報表.xls 裡面日期會顯示 104年02月16日
EXCEL指令：

``` bash
=CONCATENATE( LEFT(MID(CELL("filename"), SEARCH("[",CELL("filename"))+1,
   SEARCH("]",CELL("filename")) - SEARCH("[",CELL("filename"))-1), 3), "年",
  RIGHT(LEFT(MID(CELL("filename"), SEARCH("[",CELL("filename")) + 1,
  SEARCH("]",CELL("filename"))-SEARCH("[",CELL("filename"))-1), 5),2), "月",
  RIGHT(LEFT(MID(CELL("filename"),SEARCH("[",CELL("filename")) + 1,
    SEARCH("]",CELL("filename"))-SEARCH("[",CELL("filename"))-1), 7),2), "日")
```
2. BAT檔案做COPY，自動根據月份做判斷複製相對應的份數，並且改成適當的檔名
母檔：YYYMMDD***.xls
複製成 1040201***.xls 到 1040228***.xls

``` bash
@Echo Off
set "year=1990"
set "month=01"
if %month% == 01 goto d31
if %month% == 02 goto d28
if %month% == 03 goto d31
if %month% == 04 goto d30
if %month% == 05 goto d31
if %month% == 06 goto d30
if %month% == 07 goto d31
if %month% == 08 goto d31
if %month% == 09 goto d30
if %month% == 10 goto d31
if %month% == 11 goto d30
if %month% == 12 goto d31

:d31
for /d %%i IN (0 1 2) DO (
        for %%j IN (0 1 2 3 4 5 6 7 8 9) DO (
        copy "YYYYMMDD測試檔案.xls" "%year%%month%%%i%%j測試檔案.xls"
        )
)
del "%year%%month%00測試檔案.xls"
copy "YYYYMMDD測試檔案.xls" "%year%%month%30測試檔案.xls"
copy "YYYYMMDD測試檔案.xls" "%year%%month%31測試檔案.xls"

:d30
for /d %%i IN (0 1 2) DO (
        for %%j IN (0 1 2 3 4 5 6 7 8 9) DO (
        copy "YYYYMMDD測試檔案.xls" "%year%%month%%%i%%j測試檔案.xls"
        )
)
del "%year%%month%00測試檔案.xls"
copy "YYYYMMDD測試檔案.xls" "%year%%month%30測試檔案.xls"

:d28
for /d %%i IN (0 1 2) DO (
        for %%j IN (0 1 2 3 4 5 6 7 8 9) DO (
        copy "YYYYMMDD測試檔案.xls" "%year%%month%%%i%%j測試檔案.xls"
        )
)
del "%year%%month%00測試檔案.xls"
del "%year%%month%29測試檔案.xls"
```
