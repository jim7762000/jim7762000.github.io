---
layout: post
title: Parsing html page with javascript
---

Here is a demonstration to grab lottery number by using RSelenium. Since RCurl cannot parse the table produced by javascript, we need to construct a server to interpret javascript and grab the information we need. The tool of building server is RSelenium. It is simple to generate the html page by javascript, then we can access the table we want.

``` R
library(RCurl)
library(XML)
library(RSelenium)
# RSelenium::checkForServer()
library(data.table)
library(dplyr)
library(magrittr)

## Access Data
readLotteryHistory = function(year){
        lottery_url = paste0("http://www.cpzhan.com/lotto649/all-results?year=", year)
        remDr$navigate(lottery_url)
        webElem <- remDr$findElement(using = 'class name', value = "mytable")
        tblSource = webElem$getElementAttribute("outerHTML")[[1]]
        tables = readHTMLTable(tblSource)
        tables
}

year_today = year(Sys.Date())
RSelenium::startServer()
remDr = RSelenium::remoteDriver()
remDr$open()
lottery = 2004:year_today %>% sapply(function(year) readLotteryHistory(year)) %>% rbindlist(.)
remDr$close()
remDr$closeServer()
names_lottery = c("year", "number", "date", "ball_1", "ball_2", "ball_3", "ball_4", "ball_5", "ball_6", "ball_s")
date_name = c("year", "month", "day")
lottery = lottery %>% setnames(old = names_lottery) %>% .$date %>%
                    as.POSIXct(.) %>% sapply(function(x) c(year(x), month(x), mday(x))) %>%
                    t() %>% data.table() %>% setnames(old = date_name) %>% cbind(lottery) %>%
                    select(c(1:3, 7:13)) %>% mutate(number = 1:nrow(lottery)) %>%
                    setcolorder(c(names_lottery[2], date_name, names_lottery[4:10]))
lottery_HT = lottery[, lapply(.SD, as.numeric)] # history table

#       number year month day ball_1 ball_2 ball_3 ball_4 ball_5 ball_6 ball_s
#    1:      1 2004     1   5     24      4     39     43     29      5     13
#    2:      2 2004     1   8     21     36     18     23      7     14     26
#    3:      3 2004     1  12     25     16     29     11     16     33     33
#    4:      4 2004     1  15     21     19      9     25      9     20      4
#    5:      5 2004     1  19     23      7      5     20     29     13     35
#   ---
# 1163:   1163 2015     1  30     26      6     15     47     20     34     18
# 1164:   1164 2015     2   3     21      5      7      9     40      7     45
# 1165:   1165 2015     2   6      3     31     49     20     36     27     40
# 1166:   1166 2015     2  10     19      6     33     17      6     26     46
# 1167:   1167 2015     2  13     15      1     19     43      6     26     29
```

The encoding in windows is trouble, I have spent 2 hour on it, but it is still unsolved. Therefore, if you want to parse the html involving non-UTF8 characters, just aware it.

