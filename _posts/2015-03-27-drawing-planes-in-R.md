---
layout: post
title: Drawing planes in R
---

A simple log for drawing 2 planes in 3D plot.

``` R
library(dplyr)
library(data.table)
library(magrittr)
library(tidyr)
library(lattice)
xygrid = expand.grid(x = -30:30, y = -30:30)
dat = xygrid %>% mutate(z1 = x+3*y, z2 = 0)
dat_piled = dat %>% gather(group, z_value, z1:z2) %>% tbl_dt(FALSE)
cols = c("cyan", "yellow")

panel.custom = function(x, y, z, xlim, ylim, zlim, xlim.scaled,
  ylim.scaled, zlim.scaled, ...) {
  panel.3dwire(x = x, y = y, z = z, xlim = xlim, ylim = ylim,
    zlim = zlim,xlim.scaled = xlim.scaled,ylim.scaled = ylim.scaled,
    zlim.scaled = zlim.scaled, col = "transparent", ...)
      }

wireframe(z_value ~ x + y, dat_piled, groups=group,
  par.settings = simpleTheme(col = cols),
  auto.key = list(columns = 2, text = paste("z", 1:2),
    rectangles = TRUE, points = FALSE),
  panel.3d.wireframe = panel.custom )
```
