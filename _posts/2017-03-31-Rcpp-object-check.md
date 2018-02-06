---
layout: post
title: Rcpp object checking
---

自己在編寫套件的時候，會想要把錯誤發生可能降到最低

所以常常做很多事前check，結果發現寫來寫去都是一樣的code在複製

所以終於下定決心來寫一個general的code去做這個

這樣C++就可以直接把所有輸入都改成用SEXP輸入做check

最後再轉成需要的class去做後續，C++ code如下：

``` c++
#include <string>
#include <sstream>
#include <Rcpp.h>

template <typename T>
std::string num2str(T Number) {
  std::ostringstream ss;
  ss << Number;
  return ss.str();
}

// [[Rcpp::export]]
void checkValue(SEXP x, const std::string varName = "x", const int RTYPE = 14, const int len = 1){
  int n = LENGTH(x);
  if (len > 0) {
    if (n != len)
      Rcpp::stop("The length of " + varName + " must be " + num2str(len) + "!\n");
  }
  if (TYPEOF(x) != RTYPE) {
    switch(RTYPE) {
    case LGLSXP:
      Rcpp::stop(varName + " must be logical type!\n");
    case INTSXP:
      Rcpp::stop(varName + " must be integer type!\n");
    case REALSXP:
      Rcpp::stop(varName + " must be double type!\n");
    case STRSXP:
      Rcpp::stop(varName + " must be string type!\n");
    case CPLXSXP:
      Rcpp::stop(varName + " must be complex type!\n");
    default:
      Rcpp::stop("Not supported type!\n");
    }
  }
  for (int i = 0; i < n; i++) {
    switch(TYPEOF(x)) {
    case LGLSXP:
      if (LOGICAL(x)[i] == NA_LOGICAL)
        Rcpp::stop(varName + " must not contain NA!\n");
      break;
    case INTSXP:
      if (INTEGER(x)[i] == NA_INTEGER)
        Rcpp::stop(varName + " must not contain NA!\n");
      break;
    case REALSXP:
      if (ISNA(REAL(x)[i]) || ISNAN(REAL(x)[i]) || !R_FINITE(REAL(x)[i]))
        Rcpp::stop(varName + " must not contain NA, NaN or Inf!\n");
      break;
    case STRSXP:
      if (STRING_ELT(x, i) == NA_STRING)
        Rcpp::stop(varName + " must not contain NA!\n");
      break;
    case CPLXSXP:
      if (ISNA(COMPLEX(x)[i].r) || ISNAN(COMPLEX(x)[i].r) || !R_FINITE(COMPLEX(x)[i].r) || 
          ISNA(COMPLEX(x)[i].i) || ISNAN(COMPLEX(x)[i].i) || !R_FINITE(COMPLEX(x)[i].i))
        Rcpp::stop(varName + " must not contain NA, NaN or Inf!\n");
      break;
    default:
      Rcpp::stop("Not supported type!\n");
    }
  }
}

// [[Rcpp::export]]
double testFunc(SEXP x){
  switch(TYPEOF(x)) {
  case INTSXP:
    checkValue(x, "x", INTSXP, -1); // len < 0 to skip checking length
    return(sum(Rcpp::as<Rcpp::IntegerVector>(x)));
  case REALSXP:
    checkValue(x, "x", REALSXP, -1); // len < 0 to skip checking length
    return(sum(Rcpp::as<Rcpp::NumericVector>(x)));
  default:
    Rcpp::stop("Not supported type!\n");
  }
}
```

我們也create一個testFunc，去做對double/integer的vector做總和

那我們就在input時做一個checking

至於其他case，為了方便測試，我們這裡就用R來測試

但別忘了這個函數真正要用地方是在C++，測試的code如下：

``` R
Rcpp::sourceCpp("checkValue.cpp")
testFunc(sample.int(10, 5)) # pass
testFunc(rnorm(5)) # pass
# check type
expect_error(testFunc(c(TRUE, FALSE)), "Not supported type!")
# cehck NA
expect_error(testFunc(c(sample.int(10, 3), NA_integer_)), "x must not contain NA!")
# check NA, NaN, Inf, -Inf
expect_error(testFunc(c(rnorm(3), NA_real_)), "x must not contain NA, NaN or Inf!")
expect_error(testFunc(c(rnorm(3), NaN)), "x must not contain NA, NaN or Inf!")
expect_error(testFunc(c(rnorm(3), Inf)), "x must not contain NA, NaN or Inf!")

library(testthat)
# values for RTYPE
# LGLSXP        10  # logical variables
# INTSXP        13  # integer variables
# REALSXP       14  # real variables
# CPLXSXP       15  # complex variables
# STRSXP        16  # string variables

# pass
checkValue(c(1, 2), "x", 14, 2)
checkValue(c(1L, 2L), "x", 13, 2)
checkValue(c(TRUE, FALSE), "x", 10, 2)
checkValue(c("1", "2"), "x", 16, 2)

# check length
expect_error(checkValue(c(1, 2), "x", 14, 1), "The length of x must be 1!")
expect_error(checkValue(c(1, 2), "x", 14, 3), "The length of x must be 3!")

# check type
expect_error(checkValue(c(1, 2), "x", 13, 2), "x must be integer type!")

# check NA, NaN, Inf, -Inf
expect_error(checkValue(c(1, NA_real_), "x", 14, 2), "x must not contain NA, NaN or Inf!")
expect_error(checkValue(c(1, NaN), "x", 14, 2), "x must not contain NA, NaN or Inf!")
expect_error(checkValue(c(1, Inf), "x", 14, 2), "x must not contain NA, NaN or Inf!")
expect_error(checkValue(c(1, -Inf), "x", 14, 2), "x must not contain NA, NaN or Inf!")
  
# check NA (NaN => NA, Inf => NA for integer type, so it is not necessary to check int with NaN or Inf.)
expect_error(checkValue(c(TRUE, NA), "x", 10, 2), "x must not contain NA.")
expect_error(checkValue(c(1L, NA_integer_), "x", 13, 2), "x must not contain NA.")
expect_error(checkValue(c("1", NA_character_), "x", 16, 2), "x must not contain NA.")

# check complex
expect_error(checkValue(complex(2, 1:2, 0:1), "x", 14, 2), "x must be double type!")
expect_error(checkValue(complex(2, c(1, NA), 0:1), "x", 15, 2), "x must not contain NA, NaN or Inf!")
expect_error(checkValue(complex(2, c(1, NaN), 0:1), "x", 15, 2), "x must not contain NA, NaN or Inf!")
expect_error(checkValue(complex(2, c(1, Inf), 0:1), "x", 15, 2), "x must not contain NA, NaN or Inf!")
```

包成一個R函數for R check R object用：

``` R
library(testthat)
checkValueR <- function(x, type, length = -1L, verbose = FALSE){
  stopifnot(type %in% c("logical", "integer", "double", "complex", "character"))
  funcCall <- match.call()
  RTYPE <- switch(type, "logical" = 10L, "integer" = 13L, "double" = 14L,
                  "complex" = 15L, "character" = 16L)
  varName <- as.character(funcCall$x)
  if (length(varName) > 1L) {
    if (verbose)
      message("x input is not a variable name instead function call, so use x as variable name.")
    varName <- "x"
  }
  checkValue(x, varName, RTYPE, length)
  invisible(NULL)
}

# check length
expect_error(checkValueR(c(1, 2), "double", 1), "The length of x must be 1!")
expect_error(checkValueR(c(1, 2), "double", 3), "The length of x must be 3!")

# check type
expect_error(checkValueR(c(1, 2), "integer", 2), "x must be integer type!")

# check NA, NaN, Inf, -Inf
expect_error(checkValueR(c(1, NA), "double", 2), "x must not contain NA, NaN or Inf!")
expect_error(checkValueR(c(1, NaN), "double", 2), "x must not contain NA, NaN or Inf!")
expect_error(checkValueR(c(1, Inf), "double", 2), "x must not contain NA, NaN or Inf!")
expect_error(checkValueR(c(1, -Inf), "double", 2), "x must not contain NA, NaN or Inf!")

# check NA
expect_error(checkValueR(c(TRUE, NA), "logical", 2), "x must not contain NA.")
expect_error(checkValueR(c(1L, NA_integer_), "integer", 2), "x must not contain NA.")
expect_error(checkValueR(c("1", NA_character_), "character", 2), "x must not contain NA.")

# check complex
expect_error(checkValueR(complex(2, 1:2, 0:1), "double", 2), "x must be double type!")
expect_error(checkValueR(complex(2, c(1, NA), 0:1), "complex", 2), "x must not contain NA, NaN or Inf!")
expect_error(checkValueR(complex(2, c(1, NaN), 0:1), "complex", 2), "x must not contain NA, NaN or Inf!")
expect_error(checkValueR(complex(2, c(1, Inf), 0:1), "complex", 2), "x must not contain NA, NaN or Inf!")
```

