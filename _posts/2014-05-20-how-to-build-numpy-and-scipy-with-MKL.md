---
layout: post
title: 如何compile numpy and scipy with Intel C++ compiler and　Intel MKL
---

這篇要來敘述怎麼在linux中，利用Intel C++ compiler以及Intel MKL編譯numpy以及scipy這兩個python的套件，以下是參考連結：

1. [https://software.intel.com/en-us/articles/numpyscipy-with-intel-mkl](https://software.intel.com/en-us/articles/numpyscipy-with-intel-mkl)
2. [http://songuke.blogspot.tw/2012/02/compile-numpy-and-scipy-with-intel-math.html](http://songuke.blogspot.tw/2012/02/compile-numpy-and-scipy-with-intel-math.html)
3. [Numpy使用MKL库提升计算性能](http://unifius.wordpress.com/2013/01/18/numpy%E4%BD%BF%E7%94%A8mkl%E5%BA%93/)
4. [Numpy fails with python-dbg](http://stackoverflow.com/questions/13587136/numpy-fails-with-python-dbg-undefined-symbol-py-initmodule4-64)

首先，先取得編譯環境以及root權限以方便進行編譯的工作，另外還有一些需要的套件要安裝，命令如下：

``` bash
sudo apt-get install python-setuptools python-pip python-dev cython
# python 3
sudo apt-get install python3-setuptools python3-pip python3-dev cython3
```

接著切換到Downloads目錄(這你可以自己調整)並下載numpy以及scipy的原始碼，命令如下：

``` bash
cd; cd Downloads
git clone https://github.com/numpy/numpy.git
git clone https://github.com/scipy/scipy.git
```

接著在numpy資料夾中新增一個site.cfg的檔案(此處以sublime text做編輯器)，命令如下：

``` bash
sudo -s
source /opt/intel/composer_xe_2013_sp1.3.174/mkl/bin/mklvars.sh intel64
source /opt/intel/composer_xe_2013_sp1.3.174/bin/compilervars.sh intel64
cd numpy
# rm -rf build # if there is a build folder
subl site.cfg
```

並且添加內容：

``` bash
[mkl]
library_dirs = /opt/intel/composer_xe_2013_sp1.3.174/compiler/lib/intel64:/opt/intel/composer_xe_2013_sp1.3.174/mkl/lib/intel64
include_dirs = /opt/intel/composer_xe_2013_sp1.3.174/compiler/include:/opt/intel/composer_xe_2013_sp1.3.174/mkl/include
mkl_libs = mkl_def, mkl_intel_lp64, mkl_intel_thread, mkl_core
lapack_libs = mkl_lapack95_lp64
libraries = iomp5
```

接著修改編譯的參數，

``` bash
subl numpy/distutils/intelccompiler.py
```

以下方文字分別取代取代文件中`self.cc_exe='icc -fPIC'`以及`self.cc_exe='icc -m64 -fPIC'`：

``` bash
self.cc_exe = 'icc -O3 -g -fPIC -fp-model strict -fomit-frame-pointer -openmp -xhost'
self.cc_exe = 'icc -m64 -O3 -g -fPIC -fp-model strict -fomit-frame-pointer -openmp -xhost'
```

最後運行這個指令就可以進行安裝了。

``` bash
python setup.py config --compiler=intelem build_clib --compiler=intelem build_ext --compiler=intelem install
# python 3
python3 setup.py config --compiler=intelem build_clib --compiler=intelem build_ext --compiler=intelem install
```

如果出現`No module named msvc9compiler`，就把`numpy/distutil/intelccompiler.py`裡面有關msvc9compiler的code都註解掉就好了。

請先測試numpy是否正常，先安裝nose這個套件：

``` bash
easy_install nose
# python3
easy_install3 nose
```

開啟python並運行(注意環境還是要source上方兩個檔案)：

``` python
import numpy
numpy.test()
```

接著編譯scipy，把site.cfg從numpy複製到scipy的資料夾中：

``` bash
cp site.cfg ../scipy/site.cfg
cd ../scipy
python setup.py config --compiler=intelem --fcompiler=intelem build_clib --compiler=intelem --fcompiler=intelem build_ext --compiler=intelem --fcompiler=intelem install
# python 3
python3 setup.py config --compiler=intelem --fcompiler=intelem build_clib --compiler=intelem --fcompiler=intelem build_ext --compiler=intelem --fcompiler=intelem install
```

開啟python測試scipy(注意環境還是要source上方兩個檔案)：

``` python
import scipy
scipy.test()
```

我跑scipy的測試會失敗三個，看了一下[別人的問答](http://stackoverflow.com/questions/9239989/error-when-testing-scipy)，他們認為應該不是太嚴重的錯，我也沒有再裡他了。最後如果中間有出錯，請記得移除掉你安裝套件的位置，假設你使用的python是2.7版就是執行下方指令，在加上tab補全剩下的檔名：

``` bash
sudo rm -r /usr/local/lib/python2.7/dist-packages/numpy
sudo rm -r /usr/local/lib/python2.7/dist-packages/scipy
# python 3.4
sudo rm -r /usr/local/lib/python3.4/dist-packages/numpy
sudo rm -r /usr/local/lib/python3.4/dist-packages/scipy
```

Add $LD_LIBRARY_PATH in environment by using `subl ~/.bashrc`.

``` bash
export LD_LIBRARY_PATH=/opt/intel/composer_xe_2013_sp1.3.174/compiler/lib/intel64:/opt/intel/composer_xe_2013_sp1.3.174/mkl/lib/intel64:$LD_LIBRARY_PATH
```
