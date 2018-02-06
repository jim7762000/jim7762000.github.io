---
layout: post
title: Taking average every k periods
---

We have a repeated-measuring data. We want to take average every 3 periods. Here is code to do it.

``` R
library(plyr)
library(dplyr)
library(data.table)
library(magrittr)
# data generation
n = 200
dat = data.table(id = 1:n, len = sample(2:15, n, replace = TRUE)) %>%
    mdply(function(id, len) data.table(id = rep(id, len),
    values = rnorm(len)))
dat = select(dat, c(id, values))
# mean
k = 3
start_time = Sys.time()
result = dat %>% group_by(id) %>%
  mutate(newgroup = rep(1:ceiling(length(values)/k),
    each = k, length = length(values))) %>%
  group_by(id, newgroup) %>% summarise(mean(values))
Sys.time() - start_time

library(dplyr)
library(data.table)
library(magrittr)
# data generation

dat_gen_f = function(N_patient, max_obs_time, n_vars){
  dat = sample(max_obs_time, N_patient, replace = TRUE) %>% {
    cbind(rep(1:N_patient, times=.), sapply(., seq, from = 1) %>% unlist())
    } %>% cbind(matrix(rnorm(nrow(.)*n_vars),, n_vars)) %>% data.table()
  setnames(dat, c("id", "obs_times", paste0("V", 1:n_vars)))
}

mean_dat_f = function(dat, k){
  result = dat %>% group_by(id) %>%
    mutate(newgroup = rep(1:ceiling(length(obs_times)/k), each = k,
    length=length(obs_times)),
    n_combine = (length(obs_times) %/% k) %>% {c(rep(k, . * k),
    rep(length(obs_times) - . * k, length(obs_times) - . * k))}) %>%
    ungroup() %>% mutate(times_combine = paste((newgroup-1)*3+1,
    (newgroup-1)*3 + n_combine, sep="-"))
  result = result %>% select(match(c(names(dat)[names(dat)!="obs_times"],
    "times_combine"), names(result))) %>% extract(, lapply(.SD, mean),
    by = "id,times_combine")
  result
}

start_time = Sys.time()
dat = dat_gen_f(30000, 20, 15)
Sys.time() - start_time
# Time difference of 1.503086 secs
start_time = Sys.time()
result = mean_dat_f(dat, 3)
Sys.time() - start_time
# Time difference of 4.236243 secs

start_time = Sys.time()
dat = dat_gen_f(13820, 15, 1)
Sys.time() - start_time
# Time difference of 0.4750271 secs
start_time = Sys.time()
result = mean_dat_f(dat, 3)
Sys.time() - start_time
# Time difference of 1.848106 secs
```
