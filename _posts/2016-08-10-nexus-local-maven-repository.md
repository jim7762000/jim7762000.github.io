---
layout: post
title: "用Nexus建立本地maven倉庫"
---

sbt每次撈maven跟sbt相關的套件時

都會花很多時間，如果能夠透過本地proxy去降低時間就好了

或是在公司內部網路無法access到外部網路時

就能夠透過proxy去處理這類問題

此時，簡單易用的Nexus就提供很好的協助

安裝很簡單，只需要到[官網](http://www.sonatype.com/download-oss-sonatype?__hssc=31049440.4.1471001289156&__hstc=31049440.ab44c999338fa3bd70ea922bfa73b935.1470829351372.1470829351372.1471001289156.2&__hsfp=203408250&hsCtaTracking=43e2beb1-4cb4-42e4-b2d8-9bb4edf0493b%7C7c6988c8-ca35-42d0-b585-ae5147f27d5b)下載

然後電腦裝有Java就可以跑了

我自己是下載2.13的版本，下載之後，解壓縮放到`C:\`下面

此處路徑假設是`C:\nexus-2.13.0-01`，按開始搜尋cmd，對cmd.exe點右鍵

以系統管理員身分開啟，鍵入`cd C:\nexus-2.13.0-01\bin`

接著，打`nexus.bat install`進行安裝，再用`nexus.bat start`運行Nexus

在瀏覽器上打`http:\\localhost:8081\nexus`就可以成功進入到configuration的網頁了

右上角log in，帳號密碼為admin/admin123


至於設置方式，這部分也很簡單

在左邊Views/Repositories下面的Repositories裡面

按下`Add...`新增`Proxy Repository`

Repository ID/Repository Name都隨便你打

Remote Storage Location就輸入你要的倉庫

這裡列舉一些我用到的：

1. http://repo2.maven.org/maven2/
1. http://repository.jboss.com/maven2/
1. http://dl.bintray.com/sbt/sbt-plugin-releases/
1. http://repo.typesafe.com/typesafe/ivy-releases/

建好之後，要為ivy另外建一個Repository Group

一樣是Add..然後Repository Group，Key上ID跟Name

把typesafe跟sbt那兩個放進去

至於repo2跟maven2放到public跟maven的central repo一起即可

最後，要在本地端使用，就要到`~/.sbt`下，去新增/修改repositories

內容是：

``` bash
[repositories]

  local

  my-ivy-proxy-releases: http://localhost:8081/nexus/content/groups/ivy/, [organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext]

  my-maven-proxy-releases: http://localhost:8081/nexus/content/groups/public
```

Reference:
  1. http://www.coderblog.cn/server/building-maven-and-sbt-repository-using-nexus-3.0.0/
  1. http://lab.howie.tw/2012/06/prepare-for-build-automation-install.html
  1. http://shzhangji.com/blog/2014/11/07/sbt-offline/
