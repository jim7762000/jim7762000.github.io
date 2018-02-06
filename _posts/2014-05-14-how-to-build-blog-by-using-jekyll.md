---
layout: post
title: 如何利用jekyll建立你的blogger
---

以下詳細介紹如何在windows環境下使用sublime text在github上建立屬於你自己的部落格

以下教學來自下列兩個網站

* [yizeng的blogger](http://yizeng.me/2013/05/10/setup-jekyll-on-windows/)
* [madhur的blogger](http://www.madhur.co.in/blog/2011/09/01/runningjekyllwindows.html

## 安裝步驟如下:

A. 下載我壓縮的工具包：[Google drive](https://drive.google.com/open?id=0B1UBN4lCLHrVU3JOR0JDQ1J4Zmc)
解壓縮之後，裡面包含九個檔案：

* rubyinstaller-2.3.0-x64.exe
* DevKit-mingw64-64-4.7.2-20130224-1432-sfx.exe
* Anaconda2-4.1.0-Windows-x86_64.exe
* install.bat
* Git-1.9.5-preview20150319.exe
* RedmondPathzip.rar

這六個檔案分別為ruby安裝檔、ruby開發環境的檔案、python安裝檔案、python安裝bat檔、Git安裝檔案以及path修改的軟體，請依下面指示安裝。

* ruby預設安裝到C:\Ruby23-x64。
* 點擊兩下DevKit-mingw64-64-4.7.2-20130224-1432-sfx.exe，進行解壓縮，為了方便說明，以及環境設定，請解壓縮到C:\rubydevkit
* 解壓縮RedmondPathzip.rar，打開資料夾中的Redmond Path.exe，在任意視窗中下方加入; C:\Ruby23-x64，(你安裝路徑有更動，請跟著更改)，如下圖下示：
![](/images/path_setup.png)
* 點擊Anaconda2-4.1.0-Windows-x86_64.exe，安裝python，預設安裝到C:\Anaconda2，然後點擊兩下install.bat，會詢問你是否安裝，按下y，便完成python安裝。
* 點擊Git-1.9.5-preview20150319.exe安裝Git，中間要注意，勾選Use Git from the Windows Command Prompt
![](/images/git_install.PNG)

B. 為了工作方便，請先按下windows鍵(在Ctr跟Alt之間)+R，開啟執行視窗，鍵入cmd，打開`Windows Command Prompt`，為了解釋方便，以後稱這個視窗為cmd。
![](/images/cmd_1.png)
![](/images/cmd_2.png)

打開cmd，他的預設目錄是在你的使用者下，請先輸入`cd ../..`，退到C:\>，如圖：
![](/images/cmd_3.png)

然後在cmd中輸入下列指令：

``` bash
cd C:/rubydevkit
ruby dk.rb init
notepad config.yml
```

輸入完以上三行指令後，將會用記事本打開一個名為`_config.yml`的檔案，最後一行改成 ` - C:\Ruby23-x64`。
![](/images/dk_rb_edit.png)

回到cmd，鍵入`ruby dk.rb install`，如果成功會出現下面的訊息：
![](/images/dk_rb_edit_2.png)

然後回到cmd，鍵入`gem install jekyll`，然後等待一下之後，他會安裝數個gems(不一定是27)，如圖：
![](/images/ruby_install_jekyll_1.png)
![](/images/ruby_install_jekyll_2.png)

另外，還需要安裝pygments，請鍵入`gem install pygments.rb`。

C. 申請git，並且clone我的庫當作基底。請到 [Github](https://github.com/)申請一個帳號，假設你的使用者名稱(username)為USERNAME，在你的github中建立一個新的repository，repository的名稱請設定為USERNAME.github.com，這樣就完成github初步的設定。接下來，請先建立好你的工作目錄，例如我設定在E:\website中，那我可以利用這個指令`cd /d E:\website`到該目錄下，你可以自行更改工作目錄，假設clone我的庫做為基底，輸入下方指令：

``` bash
mkdir USERNAME.github.com
git clone https://github.com/ChingChuan-Chen/chingchuan-chen.github.com USERNAME.github.com
```

記得當中的USERNAME要改成你在github的username。例如我的username叫做imstupid，預期output如下圖：
![](/images/cmd_3.png)

再來就是init github的本地倉庫，以及設定你的github遠端帳號，指令如下：

``` bash
cd USERNAME.github.com
git init
git remote set-url origin https://github.com/USERNAME/USERNAME.github.com.git
git push origin master
```

過程中會要求輸入你的github的帳號(username)以及其密碼(password)，之後你就可以在你的github上看到你上傳的檔案了！最後就是一些簡單的修改，例如記事本去修改`_config.yml` (簡單的指令是`notepad _config.yml`，或是用記事本把它打開)：
![](/images/config.png)
![](/images/config2.png)

檔案中，#是註解，程式不會去閱讀的部分可以寫在#後面，其他前面沒有#的部分就是你可以更改的部分，當然更進階的話，你還可以添加一些選項進去，像是更動`title :`後面的文字就是在更改你主題頁的名稱。修改之後存檔，在cmd中輸入`git commit -am "message"`這個目的是儲存你所有的修改，以及添加修改的相關訊息message (這個可以自己改)，例如我想記錄這次的修改是增加新的文章，我可以打`git commit -am "new post"`。

還有git的使用如你想在你的目錄下新增東西，你要讓它能夠出現在網站上就要先加入名單中，輸入`git add .`的指令加入你所新增的檔案，舉例來說，我增加了幾張圖，我就先打入`git add .`，接著commit，然後push(上傳)到我的github，操作示範如下：
![](/images/cmd_5.png)

如果想移除檔案，就輸入`git rm filename`，filename是你的檔案名稱，只是注意的是這個操作不只把從github抹除，同時也會把你硬碟的檔案刪除。其他的指令利用可能就要慢慢再去學習。

D. 其他部分，最重要的是如何預覽，在cmd中輸入`jekyll serve`會出現下面：
![](/images/cmd_6.png)

然後在你的瀏覽器(例如IE, chrome or firefox)輸入`localhost:4000`，就可以出現你blogger的預覽畫面：
![](/images/browser.png)

還有PO文部分，可以先更改_posts下我的文章，它的檔案格式是yyyy-mm-dd-ANameOfPost.md，可以直接利用記事本做編輯，最前面是一些基本設定：

``` html
---
layout: post
cTitle: 如何利用jekyll建立你的blogger
title: how to build blog by using jekyll
description: ""
---
{}
一些文字...
123

456
```

兩個---中是關於你post的設定，layout是設定我現在的格式是什麼，在_posts裡面就理所當然是設定post，title是設定你文章的標題(這是顯示的標題)，title是標題(提供給程式控制)，decription是關於你這篇文章的敘述，category是你文章的分類，cssdemo是檔案的格式，這部分我還不熟，請先跟我設定相同，或是你自行摸索，tags是標籤，方便你自己以及其他人找尋相關文章，最後，published是設定是否要公開於網站上，你如果還沒寫好的文章就可以先改成false，那你確定要公開就改成true，include部分是必須要引入的設定，最好不要省略，more那列是在首頁顯示部分到此，例如上面的例子，就是首頁只會顯示123，而456要等你點開文章才會看到。剩下還要更改的部分是index.html以及一些小地方，如果需要幫助，再到左下角點選我的名字就可以連到我的facebook與我聯絡。最後，溫馨提醒：文章的編寫可以對照我的post跟我的blogger顯示文章去推敲寫法，總之，從模仿開始，我也才剛學會架設blogger一周而已。

Note: github不會即時更新，需要等待幾分鐘才會更新你新的上傳。

2016-01-18增補：

沒空寫新的文章，根據這個部落格風格的原作者表示出了一個github-page的gem，用法參考這篇，[這篇](http://wcc723.github.io/jekyll/2014/09/05/github-page)。

``` bash
gem install github-pages
gem install rails
cd USERNAME.github.com
bundle install
bundle exec jekyll serve --watch
```
