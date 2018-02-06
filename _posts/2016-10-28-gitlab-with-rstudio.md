---
layout: post
title: "GitLab Server與RStudio結合管理程式碼"
---


因為不想要用公開的程式碼管理

想要用local server做管理

又想要用類似github的功能

所以找了一下，發現gitlab又提供類似功能

而且RStudio可以直接使用

安裝的話，直接參考：https://about.gitlab.com/downloads/#centos7

那安裝完之後，可以先連去 `http://<伺服器IP>`

一開始會先讓你改密碼，然後你就可以自己create一個新帳號了

進去會看到一片空白，上面有個狐狸頭

![](/images/gitlab3.png)


我們可以先新創一個專案，叫做`my-first-project`

這裡創專案可以用Group或是Individual的型式來建立，我們這用Individual來建立，如下圖：

![](/images/gitlab4.png)


先在本機端裝好Git([官網](https://git-scm.com/))，看是要下載Portable還是安裝版皆可

再來就可以打開RStudio了

先進到`Tools->Global Options...`裡面的'Git/SVN`的分頁，點選`Create RSA Key...`

建立之後，可以到使用者資料夾下的.ssh裡面看到自己的SSH key

在GitLab上設定SSH Key，之後Commit就不用輸入帳號密碼了

GitLab的設定在你登入後點右上方小圖案進去的`Profile Settings`

裡面有一個`SSH Keys`的分頁，把你的`id_ras.pub`裡面

那串ssh-rsa開頭的文字貼到Key，並給個名字，然後Add即可

![](/images/gitlab1.png)


最後，就是在本地端開一個資料夾

根據建立好的專案後面的指令做一次Clone，如下圖：

![](/images/gitlab2.png)

PS: 我這裡把Hostname都換成IP了，因為我沒有設定Hostname...


最後，就可以在Rstudio開新的專案來使用Git管理專案，流程如下：

![](/images/gitlab5.png)

![](/images/gitlab6.png)

創好專案之後，切換到`Git`分頁，點`Commit`，就會跳出下面視窗

你可以勾選左邊有變更的檔案，然後輸入你要Commit的訊息就可以留下一個record了

![](/images/gitlab7.png)

結束之後按下Push就會成功上傳到GitLab上了

![](/images/gitlab8.png)

去網頁就可以看到你上傳的檔案

![](/images/gitlab9.png)

PS: 建議不要用New Project裡面的Version Control會認不到，要去更改remote.url的設定
