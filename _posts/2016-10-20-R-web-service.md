---
layout: post
title: "R web service"
---

R也可以開啟一個簡單的server

讓使用者透過PUT或是GET去query想要的資訊

以下是實作

使用RStudio的`httpuv`這個套件來做

先用一個給什麼Query都回成JSON的app

``` R
library(httpuv)
library(stringr)
library(RCurl)
library(httr)
library(pipeR)

app <- list(call = function(request) {
  query <- request$QUERY_STRING %>>% str_replace_all("^\\?", "")
  output <- httr:::parse_query(query) %>>% lapply(URLdecode) %>>% 
    `names<-`(sapply(names(.), URLdecode)) %>>% toJSON
  if (query == "" || !is.character(query)) {
    return(list(status = 200L, headers = list('Content-Type' = 'text/json'), body = ""))
  } else {
    return(list(status = 200L, headers = list('Content-Type' = 'text/json'), body = output))
  }
})

openServer <- function(app){
  tryCatch({
    server <- startServer("127.0.0.1", 8988, app=app)
    on.exit(stopServer(server))
    while(TRUE) {
      service()
      Sys.sleep(0.001)
    }
  })
}
openServer(app)
```

在一個R跑上面這個script，然後再開一個R去跑下面的script就可以看到成果：

``` R
library(httr)
library(pipeR)

"http://127.0.0.1:8988/?hello=world&hello2=celest" %>>% URLencode %>>% GET %>>% content("text")
# text_content() deprecated. Use content(x, as = 'text')
# No encoding supplied: defaulting to UTF-8.
# [1] "{\"hello\":[\"world\"],\"hello2\":[\"celest\"]}"
```

下面給一個或許可以用來拉資料的方案XD

``` R
app2 <- list(call = function(request) {
  query <- request$QUERY_STRING %>>% str_replace_all("^\\?", "")
  if (str_detect(query, "data=")) {
    dataName <- httr:::parse_query(query) %>>% `[[`("data") %>>% URLdecode
  }
  
  if (query == "" || !is.character(query)) {
    return(list(status = 200L, headers = list('Content-Type' = 'text/json'), body = ""))
  } else {
    return(list(status = 200L, headers = list('Content-Type' = 'text/json'), 
                body = toJSON(get(dataName))))
  }
})
openServer(app2)

# test code
library(httr)
library(pipeR)
library(jsonlite)

paste0("http://127.0.0.1:8988/?data=", "iris") %>>% GET %>>% content("text") %>>% fromJSON %>>% head
#   Sepal.Length Sepal.Width Petal.Length Petal.Width Species
# 1          5.1         3.5          1.4         0.2  setosa
# 2          4.9         3.0          1.4         0.2  setosa
# 3          4.7         3.2          1.3         0.2  setosa
# 4          4.6         3.1          1.5         0.2  setosa
# 5          5.0         3.6          1.4         0.2  setosa
# 6          5.4         3.9          1.7         0.4  setosa
```
