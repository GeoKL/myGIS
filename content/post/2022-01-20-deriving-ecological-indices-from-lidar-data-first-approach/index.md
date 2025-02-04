---
title: Deriving ecological indices from LiDAR data (first approach)
author: KL
date: '2022-01-20'
slug: []
categories:
  - Assignment
tags:
  - LiDAR
  - indices
subtitle: ''
description: ''
image: ''
toc: yes
output:
  blogdown::html_page:
    keep_md: yes
weight: 100
---

# Deriving ecological indices from LiDAR data (first approach, not finished)

## Research Question:

Is it possible to derive a suitable set of predictor variables from LiDAR 
data to obtain a reliable prediction of the microclimate parameters temperature and humidity?'

## Task:

- Read the ressources related to forest and identify those which you will use for attempting the task.
- Decide which algorithms and indices are adequate to answer the research question
- Apply and document this findings with on base of the scripts of the this unit experiences 

The Paper  "Compatibility of Aerial and Terrestrial LiDAR for
Quantifying Forest Structural Diversity" by LaRue et al. (2020) thematizes mainly the comparison of airial (ALS) and teresstrial (TLS) LiDAR techniques for
quantifying forest structural diversity since Structural Diversity is, as they state, a "key feautrue of forest ecosystems" (LaRue, 2020, S.1).
Structural Diversity therefore also can be used to predict microclimate (ebd., S.1). Their findings "indicated that aerial LiDAR could be of 
use in quantifying broad-scale variation in structural diversity across macroscales (ebd., S.1). So according to LaRue et al. (2020), our 
Research question should be answered with yes, it is possible. In their supplementary material, LaRue et al. list many different metrics 
for structural diversity and how they are calculated in R. This will be usefull later on.

To find the right predictors to obtain a prediction of temperature and humidity, I additionally studied chapter 8 of "The lidR package by Jean-Romain Roussel, 
Tristan R.H. Goodbody, Piotr Tompalski 2021-01-15" and tried some of the mentioned metrics as follows. Also the review "Lidar data as indicators for forest biological diversity: a review" by Ida Marielle Mienna et al.(2018) helped to get an overview of different possible predictor variables.
 

Setup:

```r
# activate envimaR package
require(envimaR)

# set root directory
rootDIR = "E:/edu/agis"

appendpackagesToLoad = c("lidR","forestr","lmom", "rLiDAR", "mapview", "raster", "rlas", "sp", "sf", "rgl","future")

# define additional subfolders comment if not needed
appendProjectDirList =  c("data/lidar/","data/lidar/l_raster","data/lidar/l_raw","data/lidar/l_norm")

## define current projection (It is not magic you need to check the meta data or ask your instructor) 
## ETRS89 / UTM zone 32N
proj4 = "+proj=utm +zone=32 +datum=WGS84 +units=m +no_defs +ellps=WGS84 +towgs84=0,0,0"
epsg_number = 25832

# load setup Skript
source(file.path(envimaR::alternativeEnvi(root_folder = rootDIR),"src/agis_setup.R"),echo = TRUE)
```

Loading and normalizing the data:

```r
#---- Get all *.las files of a folder into a list
las_files = list.files(envrmt$path_l_raw,
                       pattern = glob2rx("*.las"),
                       full.names = TRUE)


# plotting the data
las = lidR::readLAS(las_files[1])
las
```

```
## class        : LAS (v1.3 format 1)
## memory       : 2 Gb 
## extent       : 477500, 478217.5, 5631730, 5632500 (xmin, xmax, ymin, ymax)
## coord. ref.  : NA 
## area         : 0.55 kunits²
## points       : 26.35 million points
## density      : 47.7 points/units²
```

```r
plot(las, bg = "darkblue", color = "Z",backend="rgl")
```

```r
las=lidR::normalize_height(las,lidR::tin())
```

```r
plot(las, bg = "darkblue", color = "Z",backend="rgl")
```

Metrics:

```r
# example metrics:  average and max. height of points at cloud/grid/voxel level:
cloud_metrics(las, func = ~mean(Z)) 
```

```
## [1] 12.29297
```

```r
hmean <- grid_metrics(las, ~mean(Z), 10) # calculate mean at 10 m
plot(hmean, col = height.colors(50))
```

<img src="index_files/figure-html/unnamed-chunk-5-1.png" width="672" />

```r
plot(voxel_metrics(las, func = ~mean(Z))) # this just gives out the original cloud, since voxel is just one point

cloud_metrics(las, func = ~max(Z))
```

```
## [1] 44.508
```

```r
plot(grid_metrics(las, func = ~max(Z)))
```

<img src="index_files/figure-html/unnamed-chunk-5-2.png" width="672" />

```r
plot(voxel_metrics(las, func = ~max(Z)))  # this just gives out the original cloud again

# at voxel level this does not really make sense, since there is only one value, so max and mean are the same
```

Grid level seems to be most sufficient to predict microclimate,
since voxel only takes one point in account (so we do not know anything about the environment of the point) 
and cloud level takes all the points at once so that we cannot make a difference between different areas of the 
study area.

```r
# another example for a metric to calculate the mean elevation and standard deviation and mean of intensity on grid level:
f <- function(z, i) {
  list(
    mean = mean(z), 
    sd = sd(i),
    imean = mean(i))
}
plot(grid_metrics(las, func = ~f(Z, Intensity)))
```

<img src="index_files/figure-html/unnamed-chunk-6-1.png" width="672" />

```r
# with the function stdmetrics, we can calculate many often used metrics (including f.e. height max and mean) at once: 
str_grid =grid_metrics(las, func = lidR::.stdmetrics, res = 10)
plot(str_grid)
```

<img src="index_files/figure-html/unnamed-chunk-6-2.png" width="672" />

```r
# point density (might be an important ecological indice, since the structural density of the woods has large impact on f.e. plant growth which is directly connected to microclimate)
density <- grid_metrics(las, ~length(Z)/16, 4) # calculate density
plot(density, col = gray.colors(50,0,1)) # some plotting
```

<img src="index_files/figure-html/unnamed-chunk-6-3.png" width="672" />

## Literature:
LaRue, Elizabeth A. and Wagner, Franklin W. and Fei, Songlin and Atkins, Jeff W. and Fahey, Robert T. and Gough, Christopher M. and Hardiman, Brady S. (2020): Compatibility of Aerial and Terrestrial LiDAR for Quantifying Forest Structural Diversity. Remote Sensing Vol. 12.

Jean-Romain Roussel, 
Tristan R.H. Goodbody, Piotr Tompalski (2021): The lidR package.<https://jean-romain.github.io/lidRbook/>.

Ida Marielle Mienna, Katrine Eldegard, Ole Martin Bollandsås, Terje Gobakken1 and Hans Ole
Ørka (2018): Lidar data as indicators for forest biological diversity: a review.
