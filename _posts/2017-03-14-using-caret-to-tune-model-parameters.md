---
layout: post
title: Using caret to tune the model parameters
---

看到陳景翔老師在社團PO了caret這個套件([原文](https://www.facebook.com/groups/1210634969026548/permalink/1276150242475020/))，興起就去玩了一下

這套件東西滿多的，一時半會也沒辦法全部看懂

不過就以repeatedcv方式tuning的方式，大概不會跳脫下面示範的範疇

``` R
library(caret)
checkInstall("xgboost") # check xgboost is installed!

numCV <- 3L
cvFold <- 10L
ratioTrain <- 0.6
# 30 tuning model and 1 final model
numSeeds <- 4L * numCV
# 4 is (tuneLength + 1L), where tuneLength is the parameter of train with default value 3
set.seed(1)
seeds <- replicate(numCV * cvFold + 1L, sample.int(2^31-1, numSeeds), simplify = FALSE)

# split training / testing set
trIndex <- createDataPartition(iris$Species, 1L, ratioTrain, FALSE)
irisTrain <- iris[trIndex, ]
irisTest <- iris[setdiff(1L:nrow(iris), trIndex), ]

# 10-fold and repeat 3 times with seeds
ctrl <- trainControl("repeatedcv", cvFold, numCV, seeds = seeds)

#### auto-tuning ####
xgFit1 <- train(Species ~ ., data = irisTrain, trControl = ctrl,
                method = "xgbTree", num_class = nlevels(iris$Species))
xgFit1

# training result (auto-tuning)
confusionMatrix(iris.Train$Species, predict(xgFit1, iris.Train))
# testing result (auto-tuning)
confusionMatrix(iris.Test$Species, predict(xgFit1, iris.Test))

#### specific grid ####
# check the model parameters
modelLookup("xgbTree")
# expand training grid
trGrid <- expand.grid(nrounds = c(50, 100), max_depth = c(10, 12), eta = c(0.1, 0.2),  
                      gamma = 0, colsample_bytree = c(0.5, 0.6), min_child_weight = 1)
# start to train model
xgFit2 <- train(Species ~ .,  data = iris.Train,
                trControl = ctrl,  method = "xgbTree",
                tuneGrid = trGrid,  num_class = nlevels(iris$Species))
xgFit2

# training result (auto-tuning)
confusionMatrix(iris.Train$Species, predict(xgFit2, iris.Train))
# testing result (auto-tuning)
confusionMatrix(iris.Test$Species, predict(xgFit2, iris.Test))
```
