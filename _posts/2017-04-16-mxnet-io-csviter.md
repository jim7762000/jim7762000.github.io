---
layout: post
title: MxNet所提供的csv data iterator
---

這篇是參考mxnet的一個example來的[來源](https://github.com/dmlc/mxnet/tree/e7514fe1b3265aaf15870b124bb6ed0edd82fa76/example/kaggle-ndsb2)

而該篇所使用的資料是這一個比賽NDSB-II所提供的，[請點我](https://www.kaggle.com/c/second-annual-data-science-bowl/data)

登入Kaggle把Data裡面的四個zip下載下來

1. `sample_submission_validate.csv.zip`
1. `train.csv.zip`
1. `train.zip`
1. `validate.zip`

MxNet提供了一個Python去做Preprocessing

因為MxNet沒有提供R版本，所以我就寫了一版R的processing放在github上[Repo連結](https://github.com/ChingChuan-Chen/mxnet-kaggle-ndsb2-example)

從Kaggle下載zip下來之後，放到一個資料夾(為了說明方便，這個資料夾就叫做root path)裡面

並從我的github repository clone下我建立的R專案，先跑`preprocessing.R`

接下來，我們就可以跑MxNet的train.R了：

``` R
# Train.R for Second Annual Data Science Bowl
# Deep learning model with GPU support
# Please refer to https://mxnet.readthedocs.org/en/latest/build.html#r-package-installation
# for installation guide

require(mxnet)
require(data.table)

##A lenet style net, takes difference of each frame as input.
get.lenet <- function() {
  source <- mx.symbol.Variable("data")
  source <- (source-128) / 128
  frames <- mx.symbol.SliceChannel(source, num.outputs = 30)
  diffs <- list()
  for (i in 1:29) {
    diffs <- c(diffs, frames[[i + 1]] - frames[[i]])
  }
  diffs$num.args = 29
  source <- mxnet:::mx.varg.symbol.Concat(diffs)
  net <-
    mx.symbol.Convolution(source, kernel = c(5, 5), num.filter = 40)
  net <- mx.symbol.BatchNorm(net, fix.gamma = TRUE)
  net <- mx.symbol.Activation(net, act.type = "relu")
  net <-
    mx.symbol.Pooling(
      net, pool.type = "max", kernel = c(2, 2), stride = c(2, 2)
    )
  net <-
    mx.symbol.Convolution(net, kernel = c(3, 3), num.filter = 40)
  net <- mx.symbol.BatchNorm(net, fix.gamma = TRUE)
  net <- mx.symbol.Activation(net, act.type = "relu")
  net <-
    mx.symbol.Pooling(
      net, pool.type = "max", kernel = c(2, 2), stride = c(2, 2)
    )
  # first fullc
  flatten <- mx.symbol.Flatten(net)
  flatten <- mx.symbol.Dropout(flatten)
  fc1 <- mx.symbol.FullyConnected(data = flatten, num.hidden = 600)
  # Name the final layer as softmax so it auto matches the naming of data iterator
  # Otherwise we can also change the provide_data in the data iter
  return(mx.symbol.LogisticRegressionOutput(data = fc1, name = 'softmax'))
}

network <- get.lenet()
batch_size <- 32

# CSVIter is uesed here, since the data can't fit into memory
data_train <- mx.io.CSVIter(
  data.csv = "train-64x64-data.csv", data.shape = c(64, 64, 30),
  label.csv = "train-systole.csv", label.shape = 600,
  batch.size = batch_size
)

data_validate <- mx.io.CSVIter(
  data.csv = "validate-64x64-data.csv",
  data.shape = c(64, 64, 30),
  batch.size = 1
)

# Custom evaluation metric on CRPS.
mx.metric.CRPS <- mx.metric.custom("CRPS", function(label, pred) {
  pred <- as.array(pred)
  label <- as.array(label)
  for (i in 1:dim(pred)[2]) {
    for (j in 1:(dim(pred)[1] - 1)) {
      if (pred[j, i] > pred[j + 1, i]) {
        pred[j + 1, i] = pred[j, i]
      }
    }
  }
  return(sum((label - pred) ^ 2) / length(label))
})

# Training the stytole net
mx.set.seed(0)
stytole_model <- mx.model.FeedForward.create(
  X = data_train,
  ctx = mx.gpu(0),
  symbol = network,
  num.round = 65,
  learning.rate = 0.001,
  wd = 0.00001,
  momentum = 0.9,
  eval.metric = mx.metric.CRPS
)

# Predict stytole
stytole_prob = predict(stytole_model, data_validate)

# Training the diastole net
network = get.lenet()
batch_size = 32
data_train <-
  mx.io.CSVIter(
    data.csv = "./train-64x64-data.csv", data.shape = c(64, 64, 30),
    label.csv = "./train-diastole.csv", label.shape = 600,
    batch.size = batch_size
  )

diastole_model = mx.model.FeedForward.create(
  X = data_train,
  ctx = mx.gpu(0),
  symbol = network,
  num.round = 65,
  learning.rate = 0.001,
  wd = 0.00001,
  momentum = 0.9,
  eval.metric = mx.metric.CRPS
)

# Predict diastole
diastole_prob = predict(diastole_model, data_validate)

accumulate_result <- function(validate_lst, prob) {
  t <- read.table(validate_lst, sep = ",")
  p <- cbind(t[,1], t(prob))
  dt <- as.data.table(p)
  return(dt[, lapply(.SD, mean), by = V1])
}

stytole_result = as.data.frame(accumulate_result("./validate-label.csv", stytole_prob))
diastole_result = as.data.frame(accumulate_result("./validate-label.csv", diastole_prob))

train_csv <- read.table("./train-label.csv", sep = ',')

# we have 2 person missing due to frame selection, use udibr's hist result instead
doHist <- function(data) {
  res <- rep(0, 600)
  for (i in 1:length(data)) {
    for (j in round(data[i]):600) {
      res[j] = res[j] + 1
    }
  }
  return(res / length(data))
}

hSystole = doHist(train_csv[, 2])
hDiastole = doHist(train_csv[, 3])

res <- read.table("data/sample_submission_validate.csv", sep = ",", header = TRUE, stringsAsFactors = FALSE)

submission_helper <- function(pred) {
  for (i in 2:length(pred)) {
    if (pred[i] < pred[i - 1]) {
      pred[i] = pred[i - 1]
    }
  }
  return(pred)
}

for (i in 1:nrow(res)) {
  key <- unlist(strsplit(res$Id[i], "_"))[1]
  target <- unlist(strsplit(res$Id[i], "_"))[2]
  if (key %in% stytole_result$V1) {
    if (target == 'Diastole') {
      res[i, 2:601] <- submission_helper(diastole_result[which(diastole_result$V1 == key), 2:601])
    } else {
      res[i, 2:601] <- submission_helper(stytole_result[which(stytole_result$V1 == key), 2:601])
    }
  } else {
    if (target == 'Diastole') {
      res[i, 2:601] <- hDiastole
    } else {
      res[i, 2:601] <- hSystole
    }
  }
}

write.table(res, file = "submission.csv", sep = ",", quote = FALSE, row.names = FALSE)
```

本篇的重點在於下面這段R，用MxNet提供的mx.io.CSVIter去batch的訓練Net模型

而這裡的`train-64x64-data.csv`，每一行都是經過resized的`30`張圖片，所以`data.shape`是`64 x 64 x 30`

而`label`則每一行是長度`600`的binary vector，其shape設定成`600`

然後給好`batch.size`，MxNet就可以批次的從csv抓資料出來train模型了

不用一股腦地把資料全部匯入到R裡面再做，不然再多的記憶體也用不完Orz

``` R
data_train <- mx.io.CSVIter(
  data.csv = "train-64x64-data.csv", data.shape = c(64, 64, 30),
  label.csv = "train-systole.csv", label.shape = 600,
  batch.size = batch_size
)
```
