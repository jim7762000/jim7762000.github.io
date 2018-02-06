---
layout: post
title: Combinations of protein in Rcpp
---

There is a sequence of protein like `A B1/B2 C1/C2 K/D E F1/F2`, `K` does not connect to next point, so it is cut at K. Therefore, the combinations of protein is in the following:

``` R
A B1 C1 K
A B2 C1 K
A B1 C2 K
A B2 C2 K

A B1 C1 D E F1
A B2 C1 D E F1
A B1 C2 D E F1
A B2 C2 D E F1
A B1 C1 D E F2
A B2 C1 D E F2
A B1 C2 D E F2
A B2 C2 D E F2
```

The code to generate sequence of protein and list the all combinations is below.

``` R
library(Rcpp)
library(RcppArmadillo)
sourceCpp(code = '
// [[Rcpp::depends(RcppArmadillo)]]
#define ARMA_64BIT_WORD
#include <RcppArmadillo.h>
using namespace Rcpp;
using namespace arma;

// [[Rcpp::export]]
List combns_f2(IntegerVector comnbs_for_each_loc, IntegerVector exist_k,
 List names_eliminate_k){
 Col<int> X(comnbs_for_each_loc.begin(), comnbs_for_each_loc.size());
 Col<int> Y(exist_k.begin(), exist_k.size(), false);
 uvec loc_Y = find(Y >= 1);
 X(loc_Y) -= 1;
 loc_Y.insert_rows(loc_Y.n_elem, X.n_elem*ones<uvec>(1)-1);
 List result_list(loc_Y.n_elem);
 CharacterVector char_k = CharacterVector::create("K");
 for (int i = 0; i < loc_Y.n_elem; i++)
 {
   uword size_mat = 1;
   for (int k = 0; k <= loc_Y(i); k++)
     size_mat *= (uword) X(k);
   CharacterMatrix tmp_result(size_mat, loc_Y(i)+1);
   for (int j = 0; j < (int) loc_Y(i)+1; j++)
   {
     if ( i < loc_Y.n_elem-1 && j == (int) loc_Y(i))
       tmp_result(_, j) = rep(char_k, size_mat);
     else
       tmp_result(_, j) = rep(as<CharacterVector>(names_eliminate_k[j]),
         size_mat / X(j));
   }
   result_list[i] = tmp_result;
 }
 return result_list;
}

// [[Rcpp::export]]
List combns_f(IntegerVector comnbs_for_each_loc, IntegerVector exist_k){
 Col<int> X(comnbs_for_each_loc.begin(), comnbs_for_each_loc.size());
 Col<int> Y(exist_k.begin(), exist_k.size(), false);
 uvec loc_Y = find(Y >= 1);
 X(loc_Y) -= 1;
 loc_Y.insert_rows(loc_Y.n_elem, X.n_elem*ones<uvec>(1)-1);
 List result_list(loc_Y.n_elem);
 for (int i = 0; i < loc_Y.n_elem; i++)
 {
   uword size_mat = 1;
   for (int k = 0; k <= loc_Y(i); k++)
     size_mat *= (uword) X(k);
   Mat<int> tmp_result(size_mat, loc_Y(i)+1);
   for (int j = 0; j < (int) loc_Y(i)+1; j++)
   {
     if ( i < loc_Y.n_elem-1 && j == (int) loc_Y(i))
       tmp_result.col(j) = repmat( Y(loc_Y(i)) *
         ones<Col<int> >(1), size_mat, 1);
     else
       tmp_result.col(j) = repmat(
         linspace<Col<int> >(1, X(j), X(j)), size_mat / X(j), 1);
   }
   result_list[i] = wrap(tmp_result);
 }
 return result_list;
}')

# x_test = list("A", c("B1", "B2"), c("C1", "C2"), c("K", "D"), "E",
#   c("F1", "F2"))
set.seed(77)
x = lapply(setdiff(LETTERS[1:14], "K"), function(a) paste0(a, 1:sample(1:5, 1)))
x = lapply(x, function(y) if(runif(1) < 0.4){c(y, "K")} else{y})

t1 = proc.time()
size_x = sapply(x, length)
exist_k = as.integer(sapply(x, function(x) which(x=="K")))
exist_k[which(is.na(exist_k))] = 0
result = combns_f(size_x, exist_k)
result_transform = vector('list', length(result))
tmp_x = x
for (j in 1:length(result))
{
  result_transform[[j]] = sapply(1:ncol(result[[j]]),
    function(i) tmp_x[[i]][result[[j]][,i]])
  if (j < length(result))
    tmp_x[[which(exist_k>=1)[j]]] =
      setdiff(tmp_x[[which(exist_k>=1)[j]]], "K")
}
proc.time() - t1
#    user  system elapsed
#    9.31    1.74   11.90
object.size(result) # 704257576 bytes
object.size(result_transform) # 1408520016 bytes

t2 = proc.time()
size_x = sapply(x, length)
exist_k = as.integer(sapply(x, function(x) which(x=="K")))
exist_k[which(is.na(exist_k))] = 0
result2 = combns_f2(size_x, exist_k, lapply(x, setdiff, y = "K"))
proc.time() - t2
#    user  system elapsed
#    1.86    0.15    2.03
object.size(result2) # 1408520016 bytes
all.equal(result_transform, result2)
# TRUE
```

This code alert me that I should go to the destination directly, not windingly. It saves a lot of time.

