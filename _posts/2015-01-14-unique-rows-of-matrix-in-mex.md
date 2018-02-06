---
layout: post
title: Finding unique rows of a matrix in mex
---

This code is to find the unique rows of a matrix. It is based on the quick sort algorithm. Thanks to this, I can use MatLab to implement faster sorting of rows.

I compiled this code with Microsoft Visual Studio 2015 and Intel Parallel Studio XE 2017 and linking to armadillo library.

I put files in my github ([Link](https://github.com/ChingChuan-Chen/fast-unqiue-rows)).

My version of MatLab is 2016a. The test script:

``` matlab
A = [9 2 9 5; 9 2 9 0; 9 2 9 5; 9 2 9 0; 9 2 9 5];
unique_rows(A)
%      9     2     9     0
%      9     2     9     5

unique(A, 'rows')
%      9     2     9     0
%      9     2     9     5
```

The compile script:

``` matlab
MKLROOT = 'C:\Program Files (x86)\IntelSWTools\compilers_and_libraries_2017.1.143\windows\mkl';
ICLLIB = 'C:\Program Files (x86)\IntelSWTools\compilers_and_libraries_2017.1.143\windows\compiler\lib\intel64_win';
OPTIMFLAGS = 'OPTIMFLAGS= /O3 /DNDEBUG /QxHost /Qparallel /Qopenmp /Qprec-div- /Qipo /Qinline-calloc';
LINKER = 'LINKER=xilink';
mex('-f', 'intel_cpp_17_vs2015.xml', '-v', '-largeArrayDims', '-IC:\armadillo-7.800.2\include', ...
    ['-I', MKLROOT, '\include'], ['-L', MKLROOT, '\lib\intel64'], ['-L', ICLLIB], ...
    'test_mkl_blas.cpp', OPTIMFLAGS, LINKER, '-lmkl_intel_lp64', '-lmkl_intel_thread', ...
    '-lmkl_core', '-llibiomp5md')
```

My mex setup file (intel_cpp_17_vs2015.xml):

``` xml
<?xml version="1.0" encoding="UTF-8" ?>
<config
    Name="Intel C++ Composer XE 2017 with Microsoft Visual Studio 2015"
    ShortName="IntelCPP15MSVCPP140"
    Manufacturer="Intel"
    Version="17.0"
    Language="C++"
    Priority="A"
    Location="$CPPROOT" >
    <Details
        CompilerExecutable="$COMPILER"
        CompilerDefines="$COMPDEFINES"
        CompilerFlags="$COMPFLAGS"
        OptimizationFlags="$OPTIMFLAGS"
        DebugFlags="$DEBUGFLAGS"
        IncludeFlags="$INCLUDE"
        LinkerExecutable="$LINKER"
        LinkerFlags="$LINKFLAGS"
        LinkerLibraries="$LINKLIBS"
        LinkerDebugFlags="$LINKDEBUGFLAGS"
        LinkerOptimizationFlags="$LINKOPTIMFLAGS"
        CommandLineShell="$CPPROOT\bin\iclvars.bat "
        CommandLineShellArg="intel64 vs2015"
        CompilerDefineFormatter="/D%s"
        LinkerLibrarySwitchFormatter="lib%s.lib;%s.lib"
        LinkerPathFormatter="/LIBPATH:%s"
        LibrarySearchPath="$$LIB;$$LIBPATH;$$PATH;$$INCLUDE;$MATLABROOT\extern\lib\$ARCH\microsoft"
    />
    <!-- Switch guide: http://msdn.microsoft.com/en-us/library/fwkeyyhe(v=vs.71).aspx -->
    <vars
          CMDLINE100="$COMPILER /c $COMPFLAGS $OPTIM $COMPDEFINES $INCLUDE $SRC /Fo$OBJ"
          CMDLINE200="$LINKER $LINKFLAGS $LINKTYPE $LINKOPTIM $LINKEXPORT $OBJS $LINKLIBS /out:$EXE"
          CMDLINE250="mt -outputresource:$EXE;2 -manifest $MANIFEST"
          CMDLINE300="del $EXP $LIB $MANIFEST $ILK"
 
          COMPILER="icl"
          COMPFLAGS="/Zp8 /GR /W3 /EHs /nologo /MD"
          COMPDEFINES="/D_CRT_SECURE_NO_DEPRECATE /D_SCL_SECURE_NO_DEPRECATE /D_SECURE_SCL=0 $MATLABMEX"
          MATLABMEX=" /DMATLAB_MEX_FILE"
          OPTIMFLAGS="/O2 /DNDEBUG"
          INCLUDE="-I&quot;$MATLABROOT\extern\include&quot; -I&quot;$MATLABROOT\simulink\include&quot;"
          DEBUGFLAGS="/Z7"
 
          LINKER="link"
          LINKFLAGS="/nologo /manifest "
          LINKTYPE="/DLL"
          LINKEXPORT="/EXPORT:mexFunction"
          LINKLIBS="/LIBPATH:&quot;$MATLABROOT\extern\lib\$ARCH\microsoft&quot; libmx.lib libmex.lib libmat.lib kernel32.lib user32.lib gdi32.lib winspool.lib comdlg32.lib advapi32.lib shell32.lib ole32.lib oleaut32.lib uuid.lib odbc32.lib odbccp32.lib"
          LINKDEBUGFLAGS="/debug /PDB:&quot;$TEMPNAME$LDEXT.pdb&quot;"
          LINKOPTIMFLAGS=""
 
          OBJEXT=".obj"
          LDEXT=".mexw64"
          SETENV="set COMPILER=$COMPILER
                set COMPFLAGS=/c $COMPFLAGS $COMPDEFINES $MATLABMEX
                set OPTIMFLAGS=$OPTIMFLAGS
                set DEBUGFLAGS=$DEBUGFLAGS
                set LINKER=$LINKER
                set LINKFLAGS=$LINKFLAGS /export:%ENTRYPOINT% $LINKTYPE $LINKLIBS $LINKEXPORT
                set LINKDEBUGFLAGS=/debug /PDB:&quot;%OUTDIR%%MEX_NAME%$LDEXT.pdb&quot;
                set NAME_OUTPUT=/out:&quot;%OUTDIR%%MEX_NAME%%MEX_EXT%&quot;"
    />
    <client>
        <engine
          LINKLIBS="$LINKLIBS libeng.lib"
          LINKEXPORT=""
          LDEXT=".exe"
          LINKTYPE=""
          MATLABMEX=""
        />
    </client>
    <locationFinder>
        <!-- CPPROOT=C:\Program Files (x86)\IntelSWTools\compilers_and_libraries\windows\ -->
        <CPPROOT>
            <and>
                <envVarExists name="ICPP_COMPILER17" />
                <fileExists name="$$\Bin\intel64\icl.exe" />
                <dirExists name="$$\..\.." />
            </and>
        </CPPROOT>
        <!-- VCROOT=C:\Program Files (x86)\Microsoft Visual Studio 14.0\ -->
        <VCROOT>
            <and>
                <or>
                    <hklmExists path="SOFTWARE\Microsoft\VisualStudio\SxS\VC7" name="14.0" />
                    <hkcuExists path="SOFTWARE\Microsoft\VisualStudio\SxS\VC7" name="14.0" />
                    <hklmExists path="SOFTWARE\Wow6432Node\Microsoft\VisualStudio\SxS\VC7" name="14.0" />
                    <hkcuExists path="SOFTWARE\Wow6432Node\Microsoft\VisualStudio\SxS\VC7" name="14.0" />
                </or>
                <fileExists name="$$\bin\amd64\cl.exe" />
                <dirExists name="$$\..\.." />
            </and>
        </VCROOT>
        <!-- SDKROOT=C:\Program Files (x86)\Windows Kits\8.1\ -->
        <SDKROOT>
            <or>
                <hklmExists path="SOFTWARE\Microsoft\Microsoft SDKs\Windows\v8.1" name="InstallationFolder" />
                <hkcuExists path="SOFTWARE\Microsoft\Microsoft SDKs\Windows\v8.1" name="InstallationFolder" />
                <hklmExists path="SOFTWARE\Wow6432Node\Microsoft\Microsoft SDKs\Windows\v8.1" name="InstallationFolder" />
                <hkcuExists path="SOFTWARE\Wow6432Node\Microsoft\Microsoft SDKs\Windows\v8.1" name="InstallationFolder" />
            </or>
        </SDKROOT>
		<!-- KITSROOT=C:\Program Files (x86)\Windows Kits\10 -->
        <KITSROOT>
			<or>
                <hklmExists path="SOFTWARE\Microsoft\Windows Kits\Installed Roots" name="KitsRoot10" />
                <hkcuExists path="SOFTWARE\Microsoft\Windows Kits\Installed Roots" name="KitsRoot10" />
                <hklmExists path="SOFTWARE\Wow6432Node\Microsoft\Windows Kits\Installed Roots" name="KitsRoot10" />
                <hkcuExists path="SOFTWARE\Wow6432Node\Microsoft\Windows Kits\Installed Roots" name="KitsRoot10" />
			</or>
		</KITSROOT>
    </locationFinder>
    <env
        PATH="$CPPROOT\bin\intel64;$VCROOT\bin\amd64;$VCROOT\vcpackages;$VCROOT\..\Common7\IDE;$VCROOT\..\Common7\Tools;$SDKROOT\bin\x64;"
        INCLUDE="$CPPROOT\compiler\include;$VCROOT\include;$VCROOT\ATLMFC\INCLUDE;$KITSROOT\include\10.0.10240.0\ucrt;$SDKROOT\Include\shared;$SDKROOT\Include\um;$SDKROOT\Include\winrt;$MATLABROOT\extern\include;"
        LIB="$CPPROOT\compiler\lib\intel64;$VCROOT\lib\amd64;$VCROOT\ATLMFC\Lib\amd64;$KITSROOT\Lib\10.0.10240.0\ucrt\x64;$SDKROOT\Lib\winv6.3\um\x64;$MATLABROOT\lib\win64;"
        LIBPATH="$CPPROOT\compiler\lib\intel64;$VCROOT\lib\amd64;$VCROOT\atlmfc\lib\amd64;$SDKROOT\Lib\winv6.3\um\x64;$MATLABROOT\extern\lib\win64;"
    />
</config>
```

``` cpp
// import header files
#include <omp.h>
// use armadillo
#define ARMA_USE_MKL_ALLOC
#include <armadillo>
// include matlab mex header files
#include "mex.h"
#include "matrix.h"

using namespace arma;

void armaSetPr(mxArray *matlabMatrix, const Mat<double>& armaMatrix)
{
    double *dst_pointer = mxGetPr(matlabMatrix);
    const double *src_pointer = armaMatrix.memptr();
    std::memcpy(dst_pointer, src_pointer, sizeof(double)*armaMatrix.n_elem);
}

int compare_vec(const rowvec& mat_row, const rowvec& pivot_row)
{
  int v = 0;
  for (uword i = 0; i < mat_row.n_elem; i++)
  {
    if (mat_row(i) < pivot_row(i))
      v = 1;
    else if (mat_row(i) > pivot_row(i))
      v = -1;
    if (v != 0)
      break;
  }
  return v;
}

void sortrows_f(mat& M, const int left, const int right)
{
  if (left < right)
  {
    int i = left, j = right;
    uword mid_loc = (uword) (left+right)/2, pivot_loc = mid_loc;
    if (right - left > 5)
    {
      uvec sortIndex = stable_sort_index(M.col(0).subvec(mid_loc-2, mid_loc+2));
      pivot_loc = as_scalar(find(sortIndex == 2)) + mid_loc - 1;
    }
    rowvec pivot_row = M.row(pivot_loc);
    while (i <= j)
    {
      while (compare_vec(M.row( (uword) i), pivot_row) == 1)
        i++;
      while (compare_vec(M.row( (uword) j), pivot_row) == -1)
        j--;
      if (i <= j)
      {
        M.swap_rows((uword) i, (uword) j);
        i++;
        j--;
      }
    }
    if (j > 0)
      sortrows_f(M, left, j);
    if (i < (int) M.n_rows - 1)
      sortrows_f(M, i, right);
  }
}

void mexFunction(int nlhs, mxArray *plhs[], int nrhs, const mxArray *prhs[])
{
    arma_rng::set_seed_random();
    int max_threads_mkl = mkl_get_max_threads();
    mkl_set_num_threads(max_threads_mkl);

    mat x = Mat<double>(mxGetPr(prhs[0]), (uword) mxGetM(prhs[0]), (uword) mxGetN(prhs[0]), false, true);
    sortrows_f(x, 0, x.n_rows - 1);
    
    uvec unique_v = join_cols(ones<uvec>(1), any(x.rows(0, x.n_rows-2) != x.rows(1, x.n_rows-1), 1));
    mat output = x.rows(find(unique_v));
    plhs[0] = mxCreateDoubleMatrix(output.n_rows, output.n_cols, mxREAL);
    armaSetPr(plhs[0], output);
}
```
