---
layout: post
title: Rcpp一些type轉換心得
---

最近把RcppBlaze推上CRAN了

寫一下在做RcppBlaze::blaze_wrap學到的東西XD

主要是`Rcpp::traits::r_sexptype_traits`跟`Rcpp::traits::storage_type`的應用

先show一個`Rcpp::traits::r_sexptype_traits`簡單的例子

做兩個向量的相乘，在過程中轉成armadillo的column vector去做

因為Rcpp Export不能是一個template function

所以雖然你可以創一個像是eleMultiBase這樣的函數

可是沒辦法export它，所以只能另外在寫兩個函數去包裹...

然後在R裡面才判斷型別做輸出

``` c++
#include <RcppArmadillo.h>
// [[Rcpp::depends(RcppArmadillo)]]

template <int RTYPE, typename T>
void copy(Rcpp::Vector<RTYPE>& x, arma::Col<T>& y) {
  y.set_size(x.size());
  for (int i = 0; i < x.size(); ++i)
    y[i] = x[i];
}

template <int RTYPE, typename T>
SEXP eleMultiBase(Rcpp::Vector<RTYPE>& x, Rcpp::Vector<RTYPE>& y) {
  arma::Col<T> x2, y2;
  copy<RTYPE, T>(x, x2);
  copy<RTYPE, T>(y, y2);
  /* or use following lines
  arma::Col<T> x2 = Rcpp::as< arma::Col<T> >(x), 
    y2 = Rcpp::as< arma::Col<T> >(y);
  */
  return Rcpp::wrap(x2 % y2);
}

// [[Rcpp::export]]
SEXP eleMulti_double(Rcpp::NumericVector x, Rcpp::NumericVector y) {
  typedef typename Rcpp::NumericVector::stored_type valueType;
  const int RTYPE = Rcpp::traits::r_sexptype_traits< valueType >::rtype;
  return eleMultiBase<RTYPE, valueType>(x, y);
}

// [[Rcpp::export]]
SEXP eleMulti_int(Rcpp::IntegerVector x, Rcpp::IntegerVector y) {
  typedef typename Rcpp::IntegerVector::stored_type valueType;
  const int RTYPE = Rcpp::traits::r_sexptype_traits< valueType >::rtype;
  return eleMultiBase<RTYPE, valueType>(x, y);
}

/*** R
  eleMulti <- function(x, y){
    if (typeof(x) == "integer") {
      return(eleMulti_int(1L:5L, 2L:6L))
    } else if (typeof(x) == "double"){
      return(eleMulti_double(1L:5L, 2L:6L))
    } else {
      stop("Non-supported type!")
    }
  }
  eleMulti(1:5, 2:6)
  eleMulti(1L:5L, 2L:6L)
*/
```

而`Rcpp::traits::storage_type`其實在寫套件的時候比較方便

我這想不到一個實際例子用在R上面的

不過在處理`RComplex`的時候就很方便

如果`y`是`Rcpp::ComplexVector`，要轉到`std::vector<std::complex>`這個型別的話，做法如下：

``` c++
#include <Rcpp.h>
#include <vector>
#include <complex>

// [[Rcpp::export]]
std::vector< std::complex<double> > complexVec(Rcpp::ComplexVector y) {
  std::vector< std::complex<double> > x(y.size());
  const int RTYPE = Rcpp::traits::r_sexptype_traits< std::complex<double> >::rtype;
  for( size_t i=0UL; i<(size_t)y.size(); ++i )
    x[i] = Rcpp::internal::caster< typename Rcpp::traits::storage_type<RTYPE>::type, std::complex<double> >( y[i] );
  return x;
}

/*** R
  len <- 5L
  y <- complex(len, rnorm(len), rnorm(len))
  complexVec(y)
*/
```

因為`RComplex`不是型別，所以只能透過`caster`跟`storage_type`去做適當轉換

想當初這個卡了我好久QQ，最後是爬RcppEigen的解法，才寫出對應的解決方法...

這篇分享到這，有任何疑問歡迎留言XD
