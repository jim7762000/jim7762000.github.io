---
layout: post
title: R with GPU performance of HiPLARM
---

This post is to benchmark the performance of HiPLARM and to introduce how to install HiPLARM.

The performance and test code are showed in following:

``` R
library(Matrix)
p = 6000
X = Matrix(rnorm(p**2), p)
Y = Matrix(rnorm(p**2), p)
s = proc.time(); Z = X %*% Y; proc.time() - s
#   user  system elapsed
# 20.646   0.137   5.193
s = proc.time(); Z = solve(X, Y); proc.time() - s
#   user  system elapsed
# 40.418   0.342  10.741
s = proc.time(); Z = crossprod(X, Y); proc.time() - s
#   user  system elapsed
# 22.706   0.128   5.986
s = proc.time(); Z = chol(X %*% t(X)); proc.time() - s
#   user  system elapsed
# 25.774   0.242   6.934
s = proc.time(); Z = solve(t(X) %*% X, X %*% Y); proc.time() - s
#   user  system elapsed
# 86.842   0.755  23.521

library(HiPLARM)
p = 6000
X = Matrix(rnorm(p**2), p)
Y = Matrix(rnorm(p**2), p)
s = proc.time(); Z = X %*% Y; proc.time() - s
#  user  system elapsed
# 2.826   1.031   3.858
s = proc.time(); Z = solve(X, Y); proc.time() - s
#  user  system elapsed
# 4.295   1.343   5.642
s = proc.time(); Z = crossprod(X, Y); proc.time() - s
#  user  system elapsed
# 2.899   1.029   3.931
s = proc.time(); Z = chol(X %*% t(X)); proc.time() - s
#  user  system elapsed
# 4.165   1.333   5.504
s = proc.time(); Z = solve(t(X) %*% X, X %*% Y); proc.time() - s
#   user  system elapsed
# 10.440   3.377  13.825
```

HiPLARM is 1.6 times faster than the R without HiPLARM. I think that it will be much faster if the size of matrix is bigger. Since the memory of my gpu is 2GB, I can't test bigger size matrix. The tests for smaller size matrix are almost the same because the data movement between host (CPU) and device (GPU). PS: I have ran the `benchmark script`(found in [Simon Urbanek’s](http://r.research.att.com/benchmarks/)), but it is so slow that I do not put it on this post.

I simply introduce how to install `HiPLARM`. Since my R is compiled by icc and MKL, so I have some trouble in installing it. You can download the auto-installer in [here](http://www.hiplar.org/download.html). I ran the auto-installer with ALTAS (I can't compile OpenBLAS and I don't know why.) and it stopped at installing the R package caused by the environment variable. I solve this problem by add following line in the file `.bashrc`:

``` bash
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/clarence/Downloads/LALibs/lib
```

where `/home/clarence/Downloads/LALibs/lib` is the installation directory of magma, plasma and hwloc. After installation, you should add `export R_PLASMA_NUM_THREADS=4` into `.bashrc`. Then you can use it!!

I have tried to compile plasma and magma with icc and MKL, the install script is present in following:

```
#!/bin/bash

mkdir LALibs
export BLDDIR=/home/clarence/Downloads/LALibs
cd $BLDDIR
mkdir lib
mkdir include
LIBDIR=$BLDDIR/lib
INCDIR=$BLDDIR/include
NUM_PHYS_CPU=`cat /proc/cpuinfo | grep 'physical id' | sort | uniq | wc -l`
NUM_CORES=`cat /proc/cpuinfo | grep 'cpu cores' | uniq | sed 's/[a-z]//g' | sed 's/://g'`
let TOTAL_CORES=$[NUM_PHYS_CPU * NUM_CORES]

## start install hwloc
wget http://www.open-mpi.org/software/hwloc/v1.9/downloads/hwloc-1.9.tar.gz
tar -xf hwloc-1.9.tar.gz
cd hwloc-1.9
./configure --prefix="$BLDDIR"
make -j $NUM_CORES
make install
export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:$LIBDIR/pkgconfig
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$LIBDIR
cd $BLDDIR

## start install PLASMA
export OMP_NUM_THREADS=1
export MKL_NUM_THREADS=1
wget http://icl.cs.utk.edu/projectsfiles/plasma/pubs/plasma-installer_2.6.0.tar.gz
tar -xf plasma-installer_2.6.0.tar.gz
cd plasma-installer_2.6.0
./setup.py --prefix="$BLDDIR" --cc=icc --fc=ifort      \
           --blaslib="-L/opt/intel/mkl/lib/lib64 -lmkl_intel_lp64 -lmkl_sequential -lmkl_core" \
           --cflags="-O3 -fPIC -I$BLDDIR/include" \
           --fflags="-O3 -fPIC" --noopt="-fPIC" \
           --ldflags_c="-I$BLDDIR/include"\
           --notesting --downlapc

# compile shared libraries
cd $LIBDIR
icc -shared -o libplasma.so -Wl,-whole-archive libplasma.a -Wl,-no-whole-archive -L. -lhwloc -llapacke
icc -shared -o libcoreblas.so -Wl,-whole-archive libcoreblas.a -Wl,-no-whole-archive -L. -llapacke
icc -shared -o libquark.so -Wl,-whole-archive libquark.a -Wl,-no-whole-archive
icc -shared -o libcoreblasqw.so -Wl,-whole-archive libcoreblasqw.a -Wl,-no-whole-archive -L. -llapacke
export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:$LIBDIR/pkgconfig
export PLASMA_NUM_THREADS=$TOTAL_CORES
export R_PLASMA_NUM_THREADS=$TOTAL_CORES
cd $BLDDIR

## start install MGMGA
export CUDADIR=/usr/local/cuda-6.0
export CUDALIB=$CUDADIR/lib64
wget http://icl.cs.utk.edu/projectsfiles/magma/downloads/magma-1.4.1.tar.gz
tar -xf magma-1.4.1.tar.gz
cd magma-1.4.1

echo "#include<cuda.h>
#include<cuda_runtime_api.h>
#include<stdio.h>

int main() {

  int deviceCount = 0;
  cudaError_t error_id = cudaGetDeviceCount(&deviceCount);

  int dev, driverVersion = 0, runtimeVersion = 0;
  struct cudaDeviceProp deviceProp;
  int prev = 0;

  for (dev = 0; dev < deviceCount; dev++) {
    cudaGetDeviceProperties(&deviceProp, dev);
    if(deviceProp.major > prev)
      prev = deviceProp.major;
    }
    if(prev >= 2 && prev < 3)
      printf(\"GPU_TARGET = Fermi\");
    else if(prev >= 3)
      printf(\"GPU_TARGET = Kepler\");
    else
      printf(\"GPU_TARGET = Tesla\");

      return 0;
}
" > getDevice.c
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$CUDALIB
gcc getDevice.c -I$CUDADIR/include -o getDevice -L$CUDALIB -lcudart
./getDevice > make.inc

echo "
CC        = gcc
NVCC      = nvcc
FORT      = gfortran

ARCH      = ar
ARCHFLAGS = cr
RANLIB    = ranlib

OPTS      = -fPIC -O3 -DADD_ -Wall -fno-strict-aliasing -fopenmp -DMAGMA_WITH_MKL -DMAGMA_SETAFFINITY
F77OPTS   = -fPIC -O3 -DADD_ -Wall
FOPTS     = -fPIC -O3 -DADD_ -Wall -x f95-cpp-input
NVOPTS    =       -O3 -DADD_ -Xcompiler -fno-strict-aliasing,-fPIC
LDOPTS    = -fPIC -fopenmp

LIB       = -lmkl_intel_lp64 -lmkl_intel_thread -lmkl_core -lpthread -lcublas -lcudart -lstdc++ -lm -liomp5 -lgfortran

-include make.check-mkl
-include make.check-cuda

LIBDIR    = -L$LIBDIR -L$MKLROOT/lib/intel64 -L$CUDADIR/lib64
INC       = -I$CUDADIR/include -I$MKLROOT/include
" >> make.inc
make -j $NUM_CORES shared

cd lib
cp *.so $LIBDIR
cd ../
cp include/*.h $BLDDIR/include/
cd $BLDDIR

## install HiPLARM
wget http://www.hiplar.org/downloads/HiPLARM_0.1.1.tar.gz
R CMD INSTALL --configure-args="--with-lapack=-L$MKLROOT/lib/intel64 --with-plasma-lib=/home/clarence/Downloads/LALibs --with-cuda-home=/usr/local/cuda-6.0 --with-magma-lib=/home/clarence/Downloads/LALibs" HiPLARM_0.1.1.tar.gz
```

I have two troubles in compiling magma. First one is failure on compiling magma with icc and success with switching the compiler to gcc. Another trouble is the function `lapack_const` which is defined in both plasma and magma, so I can't compile and I use the suggestion on [MAGMA forum](http://icl.cs.utk.edu/magma/forum/viewtopic.php?f=2&t=961) to disable the magma-with-plasma routines. To disable the magma-with-plasma routines, you need to comment the line `PLASMA = ...` in the file `Makefile.internal` like this:

``` bash
# Use Plasma to compile zgetfl and ztstrf
# PLASMA = $(shell pkg-config --libs plasma 2> /dev/null )
```

My environment is ubuntu 14.04. My CPU is 3770K@4.3GHz and GPU is GTX 670. If you have some questions, you can reference following urls:



