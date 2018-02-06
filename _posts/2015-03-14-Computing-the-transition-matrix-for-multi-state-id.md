---
layout: post
title: Computing the transition matrix for multi-state individual
---

We have a repeated-measuring data. We want to take average every 3 periods. Here is code to do it.

``` R
## A simple way to compute transition matrix if every individual does not have multiple state.
library(data.table)
library(dplyr)
library(magrittr)

# data generation
N_patient = 1000

dat = sample(48, N_patient, replace = TRUE) %>% {
    cbind(rep(1:N_patient, times=.), sapply(., seq, from = 1) %>% unlist())
    } %>% cbind(sample(6, nrow(.), replace = TRUE)) %>%
    data.table() %>% setnames(c("id", "obs_time", "dose"))

dat_transform = dat$dose %>%
  spMatrix(length(.), 6, 1:length(.), ., rep(1,length(.))) %>%
  as.matrix() %>% data.table() %>% setnames(paste0("dose_", 1:6)) %>%
  cbind(select(dat, 1:2), .)

## inverse transformation of dat_transform
# dat = dat_transform %>%
#   mutate(dose = dose_1 + dose_2 * 2 + dose_3 * 3 + dose_4 * 4 +
#     dose_5 * 5 + dose_6 * 6) %>% select(c(1,2, 9))

st = proc.time()
dat_count = dat %>% group_by(id) %>%
  mutate(previous_dose = c(0, dose[-length(dose)])) %>%
  filter(obs_time > 1) %>%
  group_by(obs_time, dose, previous_dose) %>%
  summarise(count = n())

transitMatrix = dat_count %>% group_by(obs_time, previous_dose) %>%
  mutate(transitP = count / sum(count)) %>% ungroup() %>%
  split(.$obs_time) %>%
    lapply(function(x) spMatrix(6, 6, x$previous_dose, x$dose,
      x$transitP))
proc.time() - st
#   user  system elapsed
#   0.72    0.05    0.78
```

``` R
## A method to compute transition matrix if every individual does have multiple state.
library(data.table)
library(dplyr)
library(magrittr)
# data generation
N_patient = 3000
dat = sample(48, N_patient, replace = TRUE) %>% {
    cbind(rep(1:N_patient, times=.), unlist(sapply(., seq, from = 1)))
    } %>% cbind(replicate(6, sample(0:1, nrow(.), TRUE))) %>%
    tbl_dt() %>% setnames(c("id", "duration", paste0("M_", 1:6))) %>%
    arrange(id, duration)

st = proc.time()
transitMatrix_eachTime = dat %>% split(.$id) %>% lapply(function(dd){
    out = lapply(1:47, function(x) matrix(0, 6, 6))
    if (nrow(dd) > 1)
    {
      tmp = dd %>% select(3:8) %>% as.matrix()
      for(i in 2:nrow(dd))
      {
        transitMatrix = matrix(0, 6, 6);
        if (sum(tmp[i-1,]) > 0)
          transitMatrix[tmp[i-1,]==1,] =
            t(replicate(sum(tmp[i-1,]), tmp[i,]))
        out[[i-1]] = transitMatrix
      }
    }
    out
  }) %>%
  Reduce(function(x, y) lapply(1:length(x),
    function(v) x[[v]]+y[[v]]), .) %>%
  lapply(function(x) x / ifelse(rowSums(x)> 0, rowSums(x), 1))
proc.time() - st
#   user  system elapsed
#  32.10    0.19   33.26

library(Rcpp)
library(RcppArmadillo)
sourceCpp(code = '
// [[Rcpp::depends(RcppArmadillo)]]
#include <RcppArmadillo.h>
using namespace Rcpp;
using namespace arma;

// [[Rcpp::export]]
List transitMatrix_f(umat M, uword maxDuration)
{
  uword maxDuration_id = M.n_rows;
  List transitMatrixList(maxDuration-1);
  umat transitMatrix(M.n_cols, M.n_cols);
  urowvec previous_M(M.n_cols);
  for (uword i = 1; i < maxDuration; i++)
  {
    transitMatrix.zeros();
    if ( i < maxDuration_id)
    {
      previous_M = M.row(i-1);
      if (any(previous_M==1))
        transitMatrix.rows(find(previous_M==1)) = repmat(M.row(i), sum(previous_M), 1);
    }
    transitMatrixList[i-1] = wrap(transitMatrix);
  }
  return transitMatrixList;
}')
library(RcppEigen)
sourceCpp(code = '
// [[Rcpp::depends(RcppEigen)]]
#include <RcppEigen.h>
using namespace Rcpp;
using Eigen::Map;
using Eigen::MatrixXd;
using Eigen::VectorXd;

// [[Rcpp::export]]
void list_sum_f(List Xr, List Yr) {
  for(int i = 0; i < Xr.size(); i++)
    Yr[i] = as< Map<MatrixXd> >(Xr[i]) + as< Map<MatrixXd> >(Yr[i]);
}

// [[Rcpp::export]]
List listAddition(List Xr) {
  int n = Xr.size();
  List list_sum = Xr[0];
  for(int j = 1; j < n; j++)
    list_sum_f(Xr[j], list_sum);
  return list_sum;
}')

st = proc.time()
maxDuration = max(dat$duration)
transitMatrix_eachTime2 = dat %>% split(.$id) %>% lapply(function(x){
    transitMatrix_f(x %>% select(-id, -duration) %>% as.matrix(),
      maxDuration)
  }) %>% listAddition() %>%
  lapply(function(x) x / ifelse(rowSums(x)> 0, rowSums(x), 1))
proc.time() - st
#   user  system elapsed
#  14.68    0.20   15.27

library(reshape2)
library(Matrix)
st = proc.time()
dat_previous = dat %>% group_by(id) %>%
  mutate_(.dots = paste0("c(0, M_", 1:6, "[-length(M_1)])")) %>%
  setnames(old = tail(names(.), 6), new = paste0("M_", 1:6, "p"))

dat_transform_1 = dat_previous %>%
  melt(id = c("id", "duration"), measure = paste0("M_", 1:6)) %>%
  filter(value == 1, duration > 1) %>% select(-value) %>%
  transform(variable = as.numeric(substr(variable, 3, 3))) %>%
  setnames("variable", "M") %>% setkey(id, duration)

dat_transform_2 = dat_previous %>%
  melt(id = c("id","duration"), measure = paste0("M_", 1:6, "p")) %>%
  filter(value == 1) %>% select(-value) %>%
  mutate(variable = as.numeric(substr(variable, 3, 3))) %>%
  setnames("variable", "M_p") %>% setkey(id, duration)

dat_combined = dat_transform_1[dat_transform_2, allow.cartesian=TRUE] %>% filter(!is.na(M), !is.na(M_p))

transitMatrix_eachTime3 = dat_combined %>% group_by(duration, M, M_p) %>%
  summarise(count = n()) %>% group_by(duration, M_p) %>%
  mutate(transitProb = count / sum(count)) %>% ungroup() %>%
  split(.$duration) %>%
  lapply(function(x) spMatrix(6, 6, x$M_p, x$M, x$transitProb))
proc.time() - st
#   user  system elapsed
#   2.15    0.31    2.62

st = proc.time()
dat_transform = dat %>%
  melt(id = c("id", "duration"), measure = paste0("M_", 1:6)) %>%
  filter(value == 1) %>% select(-value) %>%
  transform(variable = as.numeric(substr(variable, 3, 3))) %>%
  setnames("variable", "M") %>% setkey(id, duration)

dat_combined = dat_transform %>% filter(duration > 1) %>%
  inner_join(dat_transform %>% transform(duration = duration + 1),
  by = c("id", "duration"))

transitMatrix_eachTime5 = dat_combined %>%
  group_by(duration, M.x, M.y) %>% summarise(count = n()) %>%
  group_by(duration, M.y) %>%
  mutate(transitProb = count / sum(count)) %>% ungroup() %>%
  split(.$duration) %>%
  lapply(function(x) spMatrix(6, 6, x$M.y, x$M.x, x$transitProb))
proc.time() - st
#   user  system elapsed
#   1.25    0.19    1.45

library(tidyr)
st = proc.time()
dat_transform = dat %>% gather(M, value, M_1:M_6) %>%
  filter(value == 1) %>% select(-value) %>%
  transform(M = as.numeric(substr(M, 3, 3)))

dat_combined = dat_transform %>% filter(duration > 1) %>%
  inner_join(dat_transform %>% transform(duration = duration + 1),
  by = c("id", "duration"))

transitMatrix_eachTime5 = dat_combined %>%
  group_by(duration, M.x, M.y) %>% summarise(count = n()) %>%
  group_by(duration, M.y) %>%
  mutate(transitProb = count / sum(count)) %>% ungroup() %>%
  split(.$duration) %>%
  lapply(function(x) spMatrix(6, 6, x$M.y, x$M.x, x$transitProb))
proc.time() - st
#   user  system elapsed
#   1.28    0.12    1.40

all.equal(transitMatrix_eachTime, transitMatrix_eachTime2)
# TRUE
all.equal(transitMatrix_eachTime, transitMatrix_eachTime3 %>%
  lapply(as.matrix) %>% set_names(NULL))
# TRUE
all.equal(transitMatrix_eachTime, transitMatrix_eachTime4 %>%
  lapply(as.matrix) %>% set_names(NULL))
# TRUE
all.equal(transitMatrix_eachTime, transitMatrix_eachTime5 %>%
  lapply(as.matrix) %>% set_names(NULL))
# TRUE

# > transitMatrix_eachTime[[1]] # the transition matrix at period 2
#           [,1]      [,2]      [,3]      [,4]      [,5]      [,6]
# [1,] 0.1579643 0.1652346 0.1625909 0.1711831 0.1784534 0.1645737
# [2,] 0.1698612 0.1692003 0.1639128 0.1625909 0.1771315 0.1573034
# [3,] 0.1635638 0.1775266 0.1569149 0.1675532 0.1682181 0.1662234
# [4,] 0.1661085 0.1654387 0.1634293 0.1654387 0.1694575 0.1701273
# [5,] 0.1722746 0.1648721 0.1561238 0.1641992 0.1709287 0.1716016
# [6,] 0.1675862 0.1620690 0.1648276 0.1655172 0.1765517 0.1634483
```
