---
layout: post
title: "使用R來建立web service"
---

1. 簡單的GET服務，傳入x, y跟species (optional)來取得png raw vector：

``` R
library(httpuv)
library(urltools)
library(pipeR)
library(Cairo)
library(png)
library(lattice)

app <- list(
  call = function(req) {
    if(req$REQUEST_METHOD != "GET")
      return(list(status = 400L, headers = list('Content-Type' = 'plain/text'), body = "Bad Request"))
    
    params <- param_get(req$QUERY_STRING, c("x", "y", "species"))
    if (is.na(params$x) || is.na(params$y))
      return(list(status = 400L, headers = list('Content-Type' = 'plain/text'),
                  body = "Bad Request with wrong query string"))
				  
    iris2 <- iris
    if (!is.na(params$species)) {
      if (nchar(params$species) > 0 && params$species %in% unique(iris$Species))
        iris2 <- subset(iris, Species == params$species)
      else
        return(list(status = 400L, headers = list('Content-Type' = 'plain/text'),
                    body = "Bad Request with wrong species"))
    }
    
    Cairo(640, 640, "/dev/null", bg = "white")
    print(xyplot(iris2[[params$x]], iris2[[params$y]], xlab = params$x, ylab = params$y))
    output <- Cairo.capture() %>>% writePNG
    dev.off()
    
    return(list(status = 200L, headers = list('Content-Type' = 'image/png'), body = output))
  }
)

runServer("0.0.0.0", 9454, app, 250)
```

驗證程式：

``` R
library(httr)
library(pipeR)
library(jsonlite)

GET("http://localhost:9454", query = list(x = "Sepal.Length", y = "Sepal.Width")) %>>% 
  content("raw") %>>% writeBin("test.png")
# test.png is a png

GET("http://localhost:9454", query = list(x = "Sepal.Length", y = "Sepal.Width")) %>>% 
  content("raw") %>>% writeBin("test.png")
# test.png is a png

GET("http://localhost:9454", query = list(x = "Sepal.Length", y = "Sepal.Width", species = "")) %>>% 
  content("text")
# "Bad Request with wrong species"

POST("http://localhost:9454", query = list(table = "xxx", id = "123")) %>>% content("text")
# Bad Request
```

2. 簡單的POST服務，傳入包含x, y值的json檔案畫圖，然後回傳

``` R
app <- list(
  call = function(req) {
    if (req$REQUEST_METHOD != "POST")
      return(list(status = 400L, headers = list('Content-Type' = 'plain/text'), body = "Bad Request"))
    
    plotDF <- fromJSON(req$rook.input$read_lines())
    
    if (!all(c("x", "y") %in% names(plotDF)))
      return(list(status = 400L, headers = list('Content-Type' = 'plain/text'),
                  body = "Bad Request with wrong arguments in json"))
    
    Cairo(640, 640, "/dev/null", bg = "white")
    print(xyplot(y ~ x, plotDF))
    output <- Cairo.capture() %>>% writePNG
    dev.off()
    
    return(list(status = 200L, headers = list('Content-Type' = 'image/png'), body = output))
  }
)

runServer("0.0.0.0", 9454, app, 250)
```

驗證程式：

``` R
library(httr)
library(pipeR)
library(jsonlite)

POST("http://localhost:9454", body = toJSON(data.frame(x = 1:6, y = 2:7))) %>>% 
  content("raw") %>>% writeBin("test.png")
# test.png is a png
```
