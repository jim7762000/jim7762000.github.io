---
layout: post
title: "用R做Oracle跟mongodb的loader"
---

這篇是用R做一個loader

把Oracle資料倒去Mongodb中

並且使用設定去將特定column改成list的方式做儲存

``` R
library(ROracle)
library(stringr)
library(data.table)
library(pipeR)
library(mongolite)

Sys.setenv(TZ = "Asia/Taipei", ORA_SDTZ = "Asia/Taipei")

numRows <- 5e7

host <- "192.168.0.120"
port <- 1521
sid <- "orcl"
connectString <- paste0("(DESCRIPTION=",
  "(ADDRESS=(PROTOCOL=TCP)(HOST=", host, ")(PORT=", port, "))",
  "(CONNECT_DATA=(SID=", sid, ")))")
con <- dbConnect(dbDriver("Oracle"), username = "system", 
                 password = "qscf12356", dbname = connectString)

# first commit
diff_seconds_90days <- difftime(Sys.time(), Sys.Date()-90, units = "secs") %>>% as.integer
sizePerGroup <- sample(11:2500, ceiling(numRows / mean(1:2500)) * 1.2, TRUE) %>>% 
  `[`(. > 10) %>>% `[`(1:which(cumsum(.) >= numRows)[1])
sizePerGroup[length(sizePerGroup)] <- sizePerGroup[length(sizePerGroup)] + numRows - sum(sizePerGroup)

var2 <- sprintf("A%05i", 1:length(sizePerGroup))
var3 <- sprintf("B%03i", 1:800)
dat <- mapply(function(ss, v2value){
  as.data.table(list(t = Sys.time() - sample(diff_seconds_90days, 1, TRUE), 
                     v2 = v2value, v3 = sample(var3, ss, TRUE), v4 = rnorm(ss)))
}, sizePerGroup, var2, SIMPLIFY = FALSE) %>>% rbindlist %>>%
  setnames(toupper(names(.))) # 記得全部欄位都要大寫

st <- proc.time()
# write data to Oracle
dbWriteTable(con, "TEST_BIG_TABLE", dat)
proc.time() - st
#   user  system elapsed 
#  29.97    3.83 1138.38
## 需要19分鐘才能夠把資料都上傳完畢

# check data size
cntDF <- dbGetQuery(con, "SELECT COUNT(*) ROWCNT FROM TEST_BIG_TABLE")
cat(sprintf("%8i", cntDF$ROWCNT))
# 50000000
## 確定是五千萬筆
dbDisconnect(con)

rm(dat, var2, var3, sizePerGroup)
gc()

## 從oracle到mongodb
# 這裡上傳的想法是根據t, v3拆成小小的資料集來做
# 資料會取得v3部分子集(這裡設定一次200個)，t一個
# 舉例來說，v3會取得B001-B200，t1選2016-07-18
# 我就從Oracle拿出v3 in B001-B200的跟t是介在2016-07-18 00:00:00到2016-07-18 23:59:59的資料
# 然後根據t, v2的組合(一個t只會對應一個v2)去把v3,v4合併成一個array，並產生新的column
# 合併出來的一筆資料就會像：
# t: '2016-08-19 08:56:13'
# v2: 'A0001'
# v3: ['B001', 'B096', ...]
# v4: [-0.972354, 0.456785, ...]
# 再把這樣的資料傳到mongodb就大功告成了
#
# 取得分批的參數
paras <- list(sprintf("B%03i", 1:800), seq.Date(Sys.Date()-90, Sys.Date(), "days") %>>% 
                format("'%Y-%m-%d %H:%M:%S'"))
parasName <- paste0(":para", 1:length(paras))
# 分批的size
batchSize <- 200
# group by 的變數
groupbyVars <- "t,v2"
# sql
sql <- paste("SELECT tbl.* FROM TEST_BIG_TABLE tbl WHERE tbl.v3 in :para1",
             "AND tbl.t >= to_date(:para2, 'YYYY-MM-DD HH24:MI:SS')",
             "AND tbl.t < to_date(:para2, 'YYYY-MM-DD HH24:MI:SS') + 1", sep = "\n")
# check sql is valid
stopifnot(all(unique(str_extract_all(sql, ":para\\d")[[1]]) %in% parasName))

# 分割出para1 set
paras1_set <- split(paras[[1]], rep(1:(ceiling(length(paras[[1]]) / batchSize)), 
                                    each = batchSize, length = length(paras[[1]]))) %>>%
  sapply(function(set) paste0("('", paste0(set, collapse = "','"), "')"))
# control the upload progress
loc <- rep(1, length(paras))

st <- proc.time()
contUpload <- TRUE
while(contUpload){
  message(sprintf("Now loc is %s ...", paste0(loc, collapse = ",")))
  
  # generate sql to query the subset
  sql_implement <- sql
  for (i in 2:length(paras))
    sql_implement <- str_replace_all(sql_implement, paste0(":para", i), paras[[i]][loc[i]])
  sql_implement <- str_replace_all(sql_implement, ":para1", paras1_set[loc[1]])
  
  # query data
  con <- dbConnect(dbDriver("Oracle"), username = "system", 
                   password = "qscf12356", dbname = connectString)
  oraDT <- dbGetQuery(con, sql_implement) %>>% data.table %>>% setnames(tolower(names(.)))
  dbDisconnect(con)
  
  if (nrow(oraDT) > 0){
    # binding variables
    groupbyVarsSplit <- str_split(groupbyVars, ",")[[1]]
    bindingVars <- setdiff(names(oraDT), groupbyVarsSplit)
    expr <- paste(bindingVars, paste0("list(", bindingVars, ")"), sep = "=") %>>% 
      c("numRecords=.N") %>>% paste(collapse = ",") %>>% (paste0(".(", ., ")"))
    mongoDT <- oraDT[ , eval(parse(text = expr)), by = groupbyVars]
    rm(oraDT); gc()
  
    # upload data to mongo
    ignoreOutput <- capture.output(mongoConn <- mongo("test_big_table", "big_table", 
                                                      "mongodb://drill:drill@192.168.0.128:27017"))
    
    # check the data
    timeClassNames <- c("Date", "POSIXct", "POSIXt", "POSIXlt")
    timeCol <- sapply(mongoDT, function(x) any(class(x) %in% timeClassNames)) %>>% (names(.[.]))
    groupbyVars2 <- setdiff(groupbyVarsSplit, timeCol)
    fieldStr <- sprintf('{"_id": 0, "numRecords": 1, %s}', 
                        paste(sprintf('"%s": 1', groupbyVars2), collapse = ","))
    groupbyVarSet <- sapply(groupbyVars2, function(x) unique(mongoDT[[x]]) %>>% 
                              (paste0("\"", ., "\"")) %>>% paste(collapse = ","))
    queryInfo <- paste0("\"", groupbyVars2 ,"\": {\"$in\": [", groupbyVarSet, "]}") %>>%
      paste(collapse = ",") %>>% (paste0("{", ., "}"))
    chkDT <- mongoConn$find(queryInfo, fieldStr)
    
    # filter the data not upload
    if (nrow(chkDT) > 0)
    {
      uploadDT <- merge(mongoDT, chkDT, all.x = TRUE, by = groupbyVars2, suffixes = c("", ".chk")) %>>%
        `[`(numRecords != numRecords.chk | is.na(numRecords.chk)) %>>% `[`(j = numRecords.chk := NULL)
    } else {
      uploadDT <- mongoDT
    }
    rm(mongoDT); gc()
    
    # start to insert data
    numFail <- 0
    if (nrow(uploadDT) > 0){
      mapply(function(i, j) paras[[i]][j], 2:length(paras), tail(loc, length(loc)-1)) %>>%
        paste(collapse = ",") %>>% (paste0("para1: \"", paras1_set[loc[1]], "\",", .)) %>>%
        sprintf(fmt = "Now upload data with set: %s") %>>% message
      
      while (numFail <= 10) {
        uploadStatus <- mongoConn$insert(uploadDT)
        if (uploadStatus)
          break
        Sys.sleep(0.1)
        numFail <- numFail + 1
      }
      if (numFail > 10)
      {
        mongoConn$drop()
        write(sprintf("Fail to upload the data with sql:\n%s", sql_implement), 'fail_upload_DB', 
              append = file.exists('fail_upload_DB'))
      }
    }
    rm(uploadDT); gc()
    # disconnect to mongo
    remove(mongoConn)
  }

  loc[1] <- loc[1] + 1
  if (loc[1] > length(paras1_set)){
    loc[1] <- 1
    loc[2] <- loc[2] + 1
  }
  for (i in 2:length(loc)){
    if (loc[i] > length(paras[[i]])){
      loc[i] <- 1
      if (i == length(loc)){
        contUpload <- FALSE
      } else {
        loc[i+1] <- loc[i+1] + 1
      }
    }
  }
}
proc.time() - st
#   user  system elapsed 
# 253.87    5.33 2971.89
## 拆成360份，上傳大概是2.5倍的時間，50分鐘

mongoConn <- mongo("test_big_table", "big_table", "mongodb://drill:drill@192.168.0.128:27017")
# count data size
sprintf("%6i", mongoConn$count()) # 305174
## 最後資料量從5000萬降到剩下30.5萬

# get data and convert to normal table
outDT <- mongoConn$find(limit = 50) %>>% data.table
outDT_trans <- outDT[ , .(v3 = unlist(v3), v4 = unlist(v4)), by = "t,v2"]
# disconnect to mongo
remove(mongoConn)
```

因為我的mongodb router server跟config server都只有一台

所以上傳過程中兩個server都非常忙碌

如果能有夠多機器去分擔router server跟config server應該還可以提升資料上傳速度
