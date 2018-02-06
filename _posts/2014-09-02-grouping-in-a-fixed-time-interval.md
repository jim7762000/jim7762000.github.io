---
layout: post
title: Grouping in a fixed time interval
---

There is a data give the Time and ID to find the Session. The value of variable session is given by the `Time` variable. The session of first time in ID is 1, session is 2 if the different time between its time and the first time of session time is longer than 1 hour, and otherwise session is 1 until next time is out of range of 1 hour.

For example, the data looks like:

``` R
ID                Time Session
1  2014-08-28 00:00:00       1
1  2014-08-28 00:23:33       1
1  2014-08-28 00:59:59       1
1  2014-08-28 01:02:17       2
1  2014-08-28 02:30:22       3
1  2014-08-28 03:29:59       3
2  2014-08-28 00:00:01       1
2  2014-08-28 03:25:49       2
2  2014-08-28 03:49:13       2
2  2014-08-28 04:29:15       3
```

We generate ID and time to test the performance.

``` R
library(data.table)
library(dplyr)
library(magrittr)
library(reshape2)
set.seed(100)
n = 2100
start_time = strptime("2014-08-01 00:00:00", "%Y-%m-%d %H:%M:%S")
dat = data.table(ID = rep(1:n, ceiling(runif(n) * 10) + 1)) %>%
  mutate(tmp_v = 1) %>% group_by(ID) %>%
  mutate(Time = as.POSIXct(sort(start_time +
    round(86400 * runif(length(tmp_v)))))) %>%
  select(one_of(c("ID", "Time"))) %>% tbl_dt(FALSE)

# first method
split_session_f = function(dat){
Session <- rep(0, nrow(dat))
  id = dat$ID[1]
  session.start = dat$Time[1]
  Session[1] = 1
  for (row in 2:nrow(dat)) {
    if (id != dat$ID[row]) {
      session.start = dat$Time[row]
      Session[row] = 1
      id = dat$ID[row]
    } else {
      if (as.numeric(dat$Time[row]-session.start, unit='hours') >= 1) {
        Session[row] = Session[row-1] + 1
        session.start = dat$Time[row]
      } else {
        Session[row] = Session[row-1]
      }
    }
  }
  Session
}

# second method
split_session_sub_f = function(time){
  output = rep(0, length(time))
  start_time_index = 1; session_num = 1
  repeat
  {
    loc = difftime(time, time[start_time_index], unit='hours') < 1 & output == 0
    output[loc] = session_num
    session_num = session_num + 1
    start_time_index = start_time_index + sum(loc)
    if(start_time_index > length(time))
      break
  }
  output
}
split_session_f2 = function(dat){
  tapply(dat$Time, dat$ID, split_session_sub_f) %>%
    unlist() %>% set_names(NULL)
}

# third method
split_session_f3 = function(dat){
  dat %>% group_by(ID) %>%
    mutate(Session = split_session_sub_f(Time)) %>%
    use_series("Session")
}

all.equal(split_session_f(dat), split_session_f2(dat))
# TRUE
all.equal(split_session_f2(dat), split_session_f3(dat))
# TRUE

## test the performance
library(rbenchmark)
benchmark(split_session_f(dat), split_session_f2(dat),
  split_session_f3(dat), replications = 20,
  columns = c("test", "replications", "elapsed", "relative"),
  order = "relative")
#                    test replications elapsed relative
# 3 split_session_f3(dat)           20   43.43    1.000
# 2 split_session_f2(dat)           20   44.99    1.036
# 1  split_session_f(dat)           20  121.34    2.794
```
