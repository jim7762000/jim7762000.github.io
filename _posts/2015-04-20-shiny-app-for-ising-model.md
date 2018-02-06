---
layout: post
title: Shiny App for Ising Model
---

Here is a demonstration for ising model with an interative interface created by R package `shiny`.

``` R
library(lattice)
library(shiny)
app = shinyApp(
  ui = shinyUI(pageWithSidebar(
    headerPanel('Ising Model'),
    sidebarPanel(
      sliderInput('girdSize', 'Gird Size', 2, 200, 30),
      selectInput('updateAlgorithm', 'Algorithm', 
        c("Metropolis-Hasting Algorithm", "Gibbs sampling"), 
        "Metropolis-Hasting Algorithm"),
      numericInput('iteration', 'Iteration', 50000, 1, 4000000),
      numericInput('updateFreq', 'Draw Model Every N Iteration', 1000, 1, 10000),
      sliderInput('temperature', 'Reciprocal of Temperature', -5, 5, 2, step = 0.1),
      actionButton('reset', 'Reset'), 
      actionButton('stop', 'Stop'),
      actionButton('start', 'Start')
    ),
    mainPanel(
      h3(textOutput("currentIteration")),
      plotOutput('IsingPlot', width = "600px", height = "600px")
    )
  )),
  server = function(input, output, session) {
    vals = reactiveValues()
    range_f = function(X, loc) c(X[loc[1], c(loc[2]-1, loc[2]+1)], X[c(loc[1]-1, loc[1]+1), loc[2]])
    resetIsingMatrix = observe({
      input$reset
      runProcess$suspend()
      isolate({
        vals$IsingMatrix = replicate(input$girdSize+2, rbinom(input$girdSize+2, 1, .5))
        vals$IsingMatrix[c(1, input$girdSize+2),] = vals$IsingMatrix[c(input$girdSize+1, 2),]
        vals$IsingMatrix[,c(1, input$girdSize+2)] = vals$IsingMatrix[,c(input$girdSize+1, 2)]
      })
    }, priority=30)
    
    setup_to_run = observe({
      input$start
      isolate({
        if (is.null(vals$IsingMatrix))
        {
          vals$IsingMatrix = replicate(input$girdSize+2, rbinom(input$girdSize+2, 1, .5))
          vals$IsingMatrix[c(1, input$girdSize+2),] = vals$IsingMatrix[c(input$girdSize+1, 2),]
          vals$IsingMatrix[,c(1, input$girdSize+2)] = vals$IsingMatrix[,c(input$girdSize+1, 2)]
        }
        if (input$updateAlgorithm == "Metropolis-Hasting Algorithm")
        {
          vals$algo_f = function(mat, temperature){
            N = nrow(mat) - 2
            loc = floor(N*runif(2)) + 2
            if (runif(1) < exp(2*(2-sum(range_f(mat, loc) == mat[loc[1], loc[2]]))*temperature))
              mat[loc[1],loc[2]] = 1 - mat[loc[1],loc[2]]
            mat[c(1, N+2),] = mat[c(N+1, 2),]
            mat[,c(1, N+2)] = mat[,c(N+1, 2)]
            mat
          }
        } else
        {
          vals$algo_f = function(mat, temperature){
            N = nrow(mat) - 2
            loc = floor(N*runif(2)) + 2
            S = 2-sum(range_f(mat, loc) == mat[loc[1], loc[2]])
            if (runif(1) < 1/(exp(-S*temperature)**2+1))
              mat[loc[1],loc[2]] = 1 - mat[loc[1],loc[2]]
            mat[c(1, N+2),] = mat[c(N+1, 2),]
            mat[,c(1, N+2)] = mat[,c(N+1, 2)]
            mat
          }
        }
        vals$iteration = input$iteration
        vals$updateFreq = input$updateFreq
        vals$temperature = input$temperature
        vals$iter = 0
      })
      runProcess$resume()
    }, priority=20)
    
    runProcess = observe({
      if (input$start == 0) return()
      isolate({
        result = vals$IsingMatrix
        i = 0
        while (i < vals$updateFreq)
        {
          result = vals$algo_f(result, vals$temperature)
          i = i + 1
        }
        vals$IsingMatrix = result
        vals$iter = vals$iter + vals$updateFreq
      })
      if (isolate(vals$iter) < isolate(vals$iteration))
        invalidateLater(500, session)
    }, priority=10)
    
    output$currentIteration = renderText({
      paste0("Current iteration: ", vals$iter)
    })
    
    stopProcess = observe({
      input$stop
      runProcess$suspend()
    })
    
    output$IsingPlot = renderPlot({
      levelplot(vals$IsingMatrix[2:(input$girdSize+1), 2:(input$girdSize+1)], 
        col.regions = c("red", "green"), 
        colorkey  = FALSE, xlab = "", ylab = "")
    })
    
    session$onSessionEnded(function() {
      runProcess$suspend()
    })
  }
)
runApp(app)
```
