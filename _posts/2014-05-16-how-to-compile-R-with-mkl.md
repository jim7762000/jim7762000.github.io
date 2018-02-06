---
layout: post
title: 如何compile R with Intel C++ compiler and Intel MKL
---

以下文章參考下列四個網址：

1. [Using Intel MKL with R](https://software.intel.com/en-us/articles/using-intel-mkl-with-r)
2. [Build R-3.0.1 with Intel C++ Compiler and Intel MKL on Linux](https://software.intel.com/en-us/articles/build-r-301-with-intel-c-compiler-and-intel-mkl-on-linux)
3. [Compiling R 3.0.1 with MKL support](http://www.r-bloggers.com/compiling-r-3-0-1-with-mkl-support/)
4. [R Installation and Administraction](http://cran.r-project.org/doc/manuals/r-devel/R-admin.html)

開始之前，先用Default R and R with Openblas來測試看看，I use testing script found in [Simon Urbanek’s](http://r.research.att.com/benchmarks/)，Openblas部份參考這個網站[For faster R use OpenBLAS instead: better than ATLAS, trivial to switch to on Ubuntu](http://www.r-bloggers.com/for-faster-r-use-openblas-instead-better-than-atlas-trivial-to-switch-to-on-ubuntu/)。

``` bash
# to install package in /usr/lib/R/library
sudo chmod -R 774 /usr/lib/R
sudo chown -R celest.celest /usr/lib/R
sudo chmod -R 774 /usr/local/lib/R
sudo chown -R celest.celest /usr/local/lib/R
# install required package
R -e "install.packages('SuppDists', repos = 'http://cran.rstudio.com/')"
# run benchmark
R -e "source('http://r.research.att.com/benchmarks/R-benchmark-25.R')"
# install OpenBLAS
sudo apt-get install libopenblas-base libatlas3gf-base
# check the BLAS is replaced with OpenBLAS
sudo update-alternatives --config libblas.so.3
sudo update-alternatives --config liblapack.so.3
# run benchmark again
R -e "source('http://r.research.att.com/benchmarks/R-benchmark-25.R')"
```

測試結果如下：
Default R：

``` R
   R Benchmark 2.5
   ===============
Number of times each test is run__________________________:  3

   I. Matrix calculation
   ---------------------
Creation, transp., deformation of a 2500x2500 matrix (sec):  0.812333333333334
2400x2400 normal distributed random matrix ^1000____ (sec):  0.474666666666667
Sorting of 7,000,000 random values__________________ (sec):  0.563333333333333
2800x2800 cross-product matrix (b = a' * a)_________ (sec):  8.99466666666667
Linear regr. over a 3000x3000 matrix (c = a \ b')___ (sec):  4.42166666666667
                      --------------------------------------------
                 Trimmed geom. mean (2 extremes eliminated):  1.26481956425649

   II. Matrix functions
   --------------------
FFT over 2,400,000 random values____________________ (sec):  0.396000000000001
Eigenvalues of a 640x640 random matrix______________ (sec):  0.718000000000001
Determinant of a 2500x2500 random matrix____________ (sec):  3.03633333333334
Cholesky decomposition of a 3000x3000 matrix________ (sec):  3.42433333333333
Inverse of a 1600x1600 random matrix________________ (sec):  2.56266666666666
                      --------------------------------------------
                Trimmed geom. mean (2 extremes eliminated):  1.77441555997029

   III. Programmation
   ------------------
3,500,000 Fibonacci numbers calculation (vector calc)(sec):  0.543999999999992
Creation of a 3000x3000 Hilbert matrix (matrix calc) (sec):  0.294333333333337
Grand common divisors of 400,000 pairs (recursion)__ (sec):  0.686666666666658
Creation of a 500x500 Toeplitz matrix (loops)_______ (sec):  0.49266666666666
Escoufier's method on a 45x45 matrix (mixed)________ (sec):  0.378999999999991
                      --------------------------------------------
                Trimmed geom. mean (2 extremes eliminated):  0.466584631384852


Total time for all 15 tests_________________________ (sec):  27.8006666666666
Overall mean (sum of I, II and III trimmed means/3)_ (sec):  1.01548017027814
                      --- End of test ---
```

R with Openblas:

``` R
   R Benchmark 2.5
   ===============
Number of times each test is run__________________________:  3

   I. Matrix calculation
   ---------------------
Creation, transp., deformation of a 2500x2500 matrix (sec):  0.755666666666667
2400x2400 normal distributed random matrix ^1000____ (sec):  0.473
Sorting of 7,000,000 random values__________________ (sec):  0.572
2800x2800 cross-product matrix (b = a' * a)_________ (sec):  0.411
Linear regr. over a 3000x3000 matrix (c = a \ b')___ (sec):  0.213
                      --------------------------------------------
                 Trimmed geom. mean (2 extremes eliminated):  0.480875883392325

   II. Matrix functions
   --------------------
FFT over 2,400,000 random values____________________ (sec):  0.372666666666666
Eigenvalues of a 640x640 random matrix______________ (sec):  0.895
Determinant of a 2500x2500 random matrix____________ (sec):  0.271333333333335
Cholesky decomposition of a 3000x3000 matrix________ (sec):  0.243333333333333
Inverse of a 1600x1600 random matrix________________ (sec):  0.328000000000001
                      --------------------------------------------
                Trimmed geom. mean (2 extremes eliminated):  0.321291459145567

   III. Programmation
   ------------------
3,500,000 Fibonacci numbers calculation (vector calc)(sec):  0.522333333333335
Creation of a 3000x3000 Hilbert matrix (matrix calc) (sec):  0.259666666666668
Grand common divisors of 400,000 pairs (recursion)__ (sec):  0.674333333333337
Creation of a 500x500 Toeplitz matrix (loops)_______ (sec):  0.499333333333335
Escoufier's method on a 45x45 matrix (mixed)________ (sec):  0.352999999999994
                      --------------------------------------------
                Trimmed geom. mean (2 extremes eliminated):  0.451548428599678


Total time for all 15 tests_________________________ (sec):  6.84366666666667
Overall mean (sum of I, II and III trimmed means/3)_ (sec):  0.411666478563312
                      --- End of test ---
```

可以看到total time已經從27.8秒到6.8秒左右，改善幅度已經不少，接著來compile R:

1. 取得R與其開發包，並安裝需要的套件，在terminal use following commands:

``` bash
sudo add-apt-repository ppa:webupd8team/java && sudo apt-get update && sudo apt-get install oracle-java8-installer && sudo apt-get install oracle-java8-set-default
apt-cache search readline xorg-dev && sudo apt-get install libreadline6 libreadline6-dev texinfo texlive texlive-binaries texlive-latex-base xorg-dev tcl8.6-dev tk8.6-dev libtiff5 libtiff5-dev libjpeg-dev libpng12-dev libcairo2-dev libglu1-mesa-dev libgsl0-dev libicu-dev R-base R-base-dev libnlopt-dev libstdc++6 build-essential libcurl4-openssl-dev texlive-fonts-extra libxml2-dev aptitude
# sudo apt-get install texlive-latex-extra
```

有一個工具要另外安裝，方式如下：

``` bash
wget http://ftp.gnu.org/pub/gnu/libiconv/libiconv-1.14.tar.gz
tar -xvzf libiconv-1.14.tar.gz
cd libiconv-1.14 && ./configure --prefix=/usr/local/libiconv
make && sudo make install
```

但是我在make過程中有出錯，我google之後找到的解法是修改`srclib/stdio.in.h`的698列:
原本的script:

``` c
_GL_WARN_ON_USE (gets, "gets is a security hole - use fgets instead");
```

修改後的scipt:

``` c
#if defined(__GLIBC__) && !defined(__UCLIBC__) && !__GLIBC_PREREQ(2, 16)
 _GL_WARN_ON_USE (gets, "gets is a security hole - use fgets instead");
#endif
```

之後再重新make就成功了。

2. 取得R source code:

``` bash
wget http://cran.csie.ntu.edu.tw/src/base/R-3/R-3.2.3.tar.gz
tar -xvzf R-3.2.3.tar.gz
```

3. 取得Intel C++ compiler and Intel MKL，你可以取得non-commercial license for this two software in intel website. 另外，64bit linux system不支援32 bits的compiler，安裝時記得取消掉IA32的安裝。

4. compilitation:

``` bash
sudo -s
source /opt/intel/composer_xe_2015/mkl/bin intel64
source /opt/intel/composer_xe_2015/bin/compilervars.sh intel64
MKL_path=/opt/intel/composer_xe_2015/mkl
ICC_path=/opt/intel/composer_xe_2015/compiler
export LD="xild"
export AR="xiar"
export CC="icc"
export CXX="icpc"
export CFLAGS="-wd188 -ip -std=gnu99 -g -O3 -openmp -parallel -xHost -ipo -fp-model precise -fp-model source"
export CXXFLAGS="-g -O3 -openmp -parallel -xHost -ipo -fp-model precise -fp-model source"
export F77=ifort
export FFLAGS="-g -O3 -openmp -parallel -xHost -ipo -fp-model source"
export FC=ifort
export FCFLAGS="-g -O3 -openmp -parallel -xHost -ipo -fp-model source"
export ICC_LIBS=$ICC_path/lib/intel64
export IFC_LIBS=$ICC_path/lib/intel64
export LDFLAGS="-L$ICC_LIBS -L$IFC_LIBS -L$MKL_path/lib/intel64 -L/usr/lib -L/usr/local/lib -openmp"
export SHLIB_CXXLD=icpc
export SHLIB_LDFLAGS="-shared -fPIC"
export SHLIB_CXXLDFLAGS="-shared -fPIC"
MKL="-L$MKL_path/lib/intel64 -lmkl_intel_lp64 -lmkl_intel_thread -lmkl_core -liomp5 -lpthread -ldl -lm"
./configure --with-blas="$MKL" --with-lapack --with-x --enable-memory-profiling --with-tcl-config=/usr/lib/tcl8.6/tclConfig.sh --with-tk-config=/usr/lib/tk8.6/tkConfig.sh --enable-R-shlib --enable-BLAS-shlib --enable-prebuilt-html
```

如果順利會出現下方的畫面：

``` bash
R is now configured for x86_64-pc-linux-gnu

  Source directory:          .
  Installation directory:    /usr/local

  C compiler:                icc  -wd188 -ip -std=gnu99 -g -O3 -openmp -parallel -xHost -ipo -fp-model precise -fp-model source
  Fortran 77 compiler:       ifort  -g -O3 -openmp -parallel -xHost -ipo -fp-model source

  C++ compiler:              icpc  -g -O3 -openmp -parallel -xHost -ipo -fp-model precise -fp-model source
  C++ 11 compiler:           icpc  -std=c++11 -g -O3 -openmp -parallel -xHost -ipo -fp-model precise -fp-model source
  Fortran 90/95 compiler:    ifort -g -O3 -openmp -parallel -xHost -ipo -fp-model source
  Obj-C compiler:

  Interfaces supported:      X11, tcltk
  External libraries:        readline, BLAS(MKL), zlib, bzlib, lzma, PCRE, curl
  Additional capabilities:   PNG, JPEG, TIFF, NLS, cairo, ICU
  Options enabled:           shared R library, shared BLAS, R profiling, memory profiling, static HTML

  Capabilities skipped:
  Options not enabled:

  Recommended packages:      yes
```

出現上方畫面就可以開始make跟install了：

``` bash
make && make check
# removing R before installation
rm /usr/lib/libR.so
rm -r /usr/lib/R
rm -r /usr/bin/Rscript
rm -r /usr/local/lib/R
make docs
make install
chown -R celest.celest /usr/local/lib/R
chmod -R 775 /usr/local/lib/R
# to add mkl and intel c compiler into path
echo 'source /opt/intel/composer_xe_2015/mkl/bin/mklvars.sh intel64' >> /etc/bash.bashrc
echo 'source /opt/intel/composer_xe_2015/bin/compilervars.sh intel64' >> /etc/bash.bashrc
exit
# install required package
R -e "install.packages('SuppDists', repos = 'http://cran.rstudio.com/')"
# run benchmark
R -e "source('http://r.research.att.com/benchmarks/R-benchmark-25.R')"
# to run rstudio-server, you have two options, first:
# echo 'rsession-which-r=/usr/local/bin/R' >> /etc/rstudio/rserver.conf
# second , please use option configure with --prefix=/usr
```

然後他就會幫你把R安裝於`usr/local/lib/R`中。

5. 測試結果

``` R
   R Benchmark 2.5
   ===============
Number of times each test is run__________________________:  3

   I. Matrix calculation
   ---------------------
Creation, transp., deformation of a 2500x2500 matrix (sec):  0.683
2400x2400 normal distributed random matrix ^1000____ (sec):  0.259666666666667
Sorting of 7,000,000 random values__________________ (sec):  0.560333333333333
2800x2800 cross-product matrix (b = a' * a)_________ (sec):  0.44
Linear regr. over a 3000x3000 matrix (c = a \ b')___ (sec):  0.193666666666666
                      --------------------------------------------
                 Trimmed geom. mean (2 extremes eliminated):  0.400041560496478

   II. Matrix functions
   --------------------
FFT over 2,400,000 random values____________________ (sec):  0.393666666666667
Eigenvalues of a 640x640 random matrix______________ (sec):  0.326999999999999
Determinant of a 2500x2500 random matrix____________ (sec):  0.215666666666666
Cholesky decomposition of a 3000x3000 matrix________ (sec):  0.183999999999999
Inverse of a 1600x1600 random matrix________________ (sec):  0.182333333333332
                      --------------------------------------------
                Trimmed geom. mean (2 extremes eliminated):  0.234990082575285

   III. Programmation
   ------------------
3,500,000 Fibonacci numbers calculation (vector calc)(sec):  0.274333333333333
Creation of a 3000x3000 Hilbert matrix (matrix calc) (sec):  0.221333333333333
Grand common divisors of 400,000 pairs (recursion)__ (sec):  0.275666666666667
Creation of a 500x500 Toeplitz matrix (loops)_______ (sec):  0.255
Escoufier's method on a 45x45 matrix (mixed)________ (sec):  0.338999999999999
                      --------------------------------------------
                Trimmed geom. mean (2 extremes eliminated):  0.268164327390206


Total time for all 15 tests_________________________ (sec):  4.80466666666666
Overall mean (sum of I, II and III trimmed means/3)_ (sec):  0.293214347493761
                      --- End of test ---
```

最後只需要用到4.8秒就可以完成了，可是complitation過程是滿麻煩的，雖然參考了多個網站，可是參數的設定都不太一樣，linux又有權限的限制，而且就算編譯成功，Rcpp這個套件不見得能夠成功，因此花了很久才終於編譯成功，並且能夠直接開啟，只是要利用到c, cpp or fortran時還是需要source compilervars.sh才能夠運行，而且我安裝了三四十個套件都沒有問題了。最後，如果沒有特別要求速度下，其實直接用OpenBLAS就可以省下很多麻煩。另外，我做了一個小小的測試於Rcpp上，速度有不少的提昇(因為用intel C++ compiler，大概增加5~10倍)，測試結果就不放上來了。以上資訊供大家參考，轉載請註明來源，謝謝。

最後附上測試環境: My environment is mint 17.3, R 3.2.3 compiled by Intel c++, fortran compiler with Intel MKL. My CPU is 3770K@4.4GHz.

To use the html help page and change the default language of R to english, you can do that:

``` bash
echo 'options("help_type"="html")' > ~/.Rprofile
echo 'LANGUAGE="en"' > ~/.Renviron
```

如果要讓Rstudio Server裡面成功啟動並且可以使用`icpc`，請在`/usr/lib/rstudio-server/R/ServerOptions.R`裡面加入下方：

``` R
Sys.setenv(PATH = "/opt/intel/composer_xe_2015.1.133/bin/intel64:/opt/intel/composer_xe_2015.1.133/debugger/gdb/intel64_mic/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/usr/lib/jvm/java-8-oracle/bin:/usr/lib/jvm/java-8-oracle/db/bin:/usr/lib/jvm/java-8-oracle/jre/bin:$PATH")
```

java config and install some useful packages:

``` R
R CMD javareconf
install.packages(c('devtools', 'testthat'))
devtools::install_github(c('klutometis/roxygen', 'hadley/assertthat', 'RcppCore/Rcpp', 'hadley/devtools', 'hadley/testthat', 'hadley/lazyeval'))
devtools::install_github(c('smbache/magrittr', 'Rdatatable/data.table', 'hadley/reshape', 'hadley/plyr', 'hadley/dplyr'))
devtools::install_github(c('RcppCore/RcppArmadillo', 'RcppCore/RcppEigen', 'RcppCore/RcppParallel'))
devtools::install_github(c('hadley/tidyr', 'hadley/purrr', 'yihui/knitr'))
```

