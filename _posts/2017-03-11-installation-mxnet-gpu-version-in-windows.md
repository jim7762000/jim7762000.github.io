---
layout: post
title: "在Windows環境下安裝GPU版本的mxnet"
---

需要的components有：

1. [CUDA Toolkit 8.0](https://developer.nvidia.com/cuda-toolkit) 下載安裝
2. [mxnet gpu prebuild](https://github.com/yajiedesign/mxnet/releases) 下載`20170310_mxnet_x64_vc14_gpu.7z`跟`vc14 base package`
3. [mxnet release zip](https://github.com/dmlc/mxnet/releases) 

為了說明，我就在D槽開一個資料夾，叫做mxnet

先把整個mxnet repository clone到`D:\mxnet\mxnet`，解壓縮mxnet release的zip，其路徑應該是`D:\mxnet\mxnet-版本`

然後把`20170310_mxnet_x64_vc14_gpu.7z`跟`prebuildbase_win10_x64_vc14.7z`解壓縮到`D:\mxnet\prebuild`裡面

再來把`D:\mxnet\prebuild\3rdparty`裡面11個dll複製到`D:\mxnet\mxnet-版本\R-package\inst\libs\x64`(沒有資料夾就自己create)

另外要再建一個``D:\mxnet\mxnet-版本\R-package\inst\include`資料夾

把`D:\mxnet\nnvm\nnvm\include\nnvm`, `D:\mxnet\include\mxnet`, `D:\mxnet\dmlc-core\include\dlmc`以及`D:\mxnet\include\mshadow`複製到裡面

最後，新增一個bat檔案

``` bash
echo import(Rcpp) > R-package/NAMESPACE
echo import(methods) >> R-package/NAMESPACE
R CMD INSTALL R-package
Rscript -e "require(mxnet); mxnet:::mxnet.export(\"R-package\")"
rm -rf R-package/NAMESPACE
Rscript -e "require(roxygen2); roxygen2::roxygenise(\"R-package\")"
R CMD INSTALL R-package
```

然後執行這個bat檔案就安裝完了 (roxygen2 0.6.1會有問題，請用舊版的roxygen2)

接下來就可以跑看看gpu可不可以用了(修改至mxnet/example/image-classfication裡面的R檔)：

``` R
library(mxnet)

download_ <- function(data_dir) {
  dir.create(data_dir, showWarnings = FALSE)
  setwd(data_dir)
  if ((!file.exists('train-images-idx3-ubyte')) ||
      (!file.exists('train-labels-idx1-ubyte')) ||
      (!file.exists('t10k-images-idx3-ubyte')) ||
      (!file.exists('t10k-labels-idx1-ubyte'))) {
    download.file(url='http://data.mxnet.io/mxnet/data/mnist.zip',
                  destfile='mnist.zip')
    unzip("mnist.zip")
    file.remove("mnist.zip")
  }
  setwd("..")
}

get_iterator <- function(data_shape) {
  get_iterator_impl <- function() {
    data_dir = 'mnist/'
    flat <- TRUE
    if (length(data_shape) == 3) flat <- FALSE
    
    train           = mx.io.MNISTIter(
      image       = paste0(data_dir, "train-images-idx3-ubyte"),
      label       = paste0(data_dir, "train-labels-idx1-ubyte"),
      input_shape = data_shape,
      batch_size  = 128,
      shuffle     = TRUE,
      flat        = flat)
    
    val = mx.io.MNISTIter(
      image       = paste0(data_dir, "t10k-images-idx3-ubyte"),
      label       = paste0(data_dir, "t10k-labels-idx1-ubyte"),
      input_shape = data_shape,
      batch_size  = 128,
      flat        = flat)
    
    ret = list(train=train, value=val)
  }
  get_iterator_impl
}


# multi-layer perceptron
get_mlp <- function() {
  data <- mx.symbol.Variable('data')
  fc1  <- mx.symbol.FullyConnected(data = data, name='fc1', num_hidden=128)
  act1 <- mx.symbol.Activation(data = fc1, name='relu1', act_type="relu")
  fc2  <- mx.symbol.FullyConnected(data = act1, name = 'fc2', num_hidden = 64)
  act2 <- mx.symbol.Activation(data = fc2, name='relu2', act_type="relu")
  fc3  <- mx.symbol.FullyConnected(data = act2, name='fc3', num_hidden=10)
  mlp  <- mx.symbol.SoftmaxOutput(data = fc3, name = 'softmax')
  mlp
}

get_lenet <- function() {
  data <- mx.symbol.Variable('data')
  # first conv
  conv1 <- mx.symbol.Convolution(data=data, kernel=c(5,5), num_filter=20)
  tanh1 <- mx.symbol.Activation(data=conv1, act_type="tanh")
  pool1 <- mx.symbol.Pooling(data=tanh1, pool_type="max",
                             kernel=c(2,2), stride=c(2,2))
  # second conv
  conv2 <- mx.symbol.Convolution(data=pool1, kernel=c(5,5), num_filter=50)
  tanh2 <- mx.symbol.Activation(data=conv2, act_type="tanh")
  pool2 <- mx.symbol.Pooling(data=tanh2, pool_type="max",
                             kernel=c(2,2), stride=c(2,2))
  # first fullc
  flatten <- mx.symbol.Flatten(data=pool2)
  fc1 <- mx.symbol.FullyConnected(data=flatten, num_hidden=500)
  tanh3 <- mx.symbol.Activation(data=fc1, act_type="tanh")
  # second fullc
  fc2 <- mx.symbol.FullyConnected(data=tanh3, num_hidden=10)
  # loss
  lenet <- mx.symbol.SoftmaxOutput(data=fc2, name='softmax')
  lenet
}

data_loader <- get_iterator(c(28, 28, 1))
download_('mnist/')
net <- get_lenet()
data <- data_loader()
train <- data$train
val <- data$value  
devs <- mx.gpu(0)

model <- mx.model.FeedForward.create(
  X                  = train,
  eval.data          = val,
  ctx                = devs,
  symbol             = net,
  begin.round        = 0,
  eval.metric        = mx.metric.top_k_accuracy,
  num.round          = 10,
  learning.rate      = 0.05,
  array.batch.size   = 128,
  optimizer          = "sgd",
  initializer        = mx.init.Xavier(factor_type="in", magnitude=2),
  batch.end.callback = mx.callback.log.train.metric(50)
)
```
