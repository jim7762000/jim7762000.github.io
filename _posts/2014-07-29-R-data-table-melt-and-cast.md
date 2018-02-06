---
layout: post
title: R data.table - melt and cast
---

`data.table` is a powerful tool for exploring data.
Here we provides a example for melting and casting data.

Code:

``` R
library(data.table)
library(reshape2)
library(dplyr)
dat = list(); length(dat) = 3
dat[[1]] = data.table(name=c("A","B","D"),value=c(23,45,100),day="day1")
dat[[2]] = data.table(name=c("A","C","D"),value=c(77,11,35),day="day2")
dat[[3]] = data.table(name=c("B","D","E"),value=c(11,44,55),day="day3")

dat2 = rbindlist(dat)
#    name value  day
# 1:    A    23 day1
# 2:    B    45 day1
# 3:    D   100 day1
# 4:    A    77 day2
# 5:    C    11 day2
# 6:    D    35 day2
# 7:    B    11 day3
# 8:    D    44 day3
# 9:    E    55 day3
dat3 = dcast.data.table(dat2, name ~ day)
#    name day1 day2 day3
# 1:    A   23   77   NA
# 2:    B   45   NA   11
# 3:    C   NA   11   NA
# 4:    D  100   35   44
# 5:    E   NA   NA   55
dat4 = filter(melt(dat3, "name"), !is.na(value))
#    name variable value
# 1:    A     day1    23
# 2:    B     day1    45
# 3:    D     day1   100
# 4:    A     day2    77
# 5:    C     day2    11
# 6:    D     day2    35
# 7:    B     day3    11
# 8:    D     day3    44
# 9:    E     day3    55
```

`cast` can change raw data into a wide table and `melt` can change data in wide table format back to raw data. I think that it is faster and easier than other functions on `data.frame`. (Some functions like `reshape`, `cast` and `melt` in `reshape`.)


