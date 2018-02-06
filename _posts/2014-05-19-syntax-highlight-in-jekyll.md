---
layout: post
title: Syntax highlight in Jekyll
---

從上週就一直在嘗試如何把我部落格的程式碼都上色，弄了好多天才發現主要的癥結。中間參考太多網站，列幾個重要的。參考網站如下：

1. [Do I need to generate a css file ...](http://stackoverflow.com/questions/9652490/do-i-need-to-generate-a-css-file-from-pygments-for-my-jekyll-blog-to-enable-col)

2. [Add code highlight with Pygments](https://github.com/pudgecon/blog-repository/blob/master/_posts/2012-09-03-add-code-highlight-with-pygments.md)

你要有syntax highlight的功能，要先安裝幾個重要的工具，第一個是要有Python，並且安裝其套件Pygments，Ruby要安裝Pygments.rb，版本0.5.4可能會出錯，我裝的是0.5.0，這個版本大多人都可以成功。接著需要調整_config.yml中的選項：

``` html
highlighter: pygments
```

另外，還有要生成highlight所需的css檔案，於cmd中鍵入

``` bash
pygmentize -S default -f html > pygments.css
```

然後在themes中的default.html中加上下面這一行：

``` html
<link rel="stylesheet" type="text/css" href="/path/to/pygments.css">
```

這樣就成功了。
