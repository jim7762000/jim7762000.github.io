---
layout: post
title: "inconsolata.sty not found in CentOS"
---

因為R編譯manual需要用到`inconsolata`

但是CentOS 7下的texlive沒有`inconsolata.sty`這個檔案，所以要自己安裝

先用`yum install levien-inconsolata-fonts`安裝字型

再來是安裝tex檔案：

``` sh
wget http://mirrors.ctan.org/install/fonts/inconsolata.tds.zip
mkdir inconsolata
unzip inconsolata.tds.zip -d inconsolata
sudo cp -r inconsolata/* /usr/share/texmf
sudo mktexlsr

tee -a /usr/share/texlive/texmf-dist/web2c/updmap.cfg << EOF
Map zi4.map
EOF

sudo yum install perl-Digest-MD5
sudo updmap-sys --force
```

這樣就可以快樂的編譯R manual了！

