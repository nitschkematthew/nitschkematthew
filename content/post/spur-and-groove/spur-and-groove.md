---
title: "Spur and Groove"
date: 2019-07-31T15:54:19+12:00
draft: false
---

## Using R, ggplot2 and ggridges to render reefs

I recently came across an artistic use for the geom_ridgeline() function by Garrick Aden-Buie, where he created a neat twitter banner using R, ggplot2, and ggridges. Click [here](https://www.garrickadenbuie.com/blog/my-ggridges-twitter-header/) to check it out!

I thought it would be cool to adapt the code and try something similar on a satellite view of some of my favourite reefs. Lets see if we can achieve a similar effect, and recreate some of the spur and groove that Heron and Wistari reefs are so well known for, but in geoms and ridgelines.

{{< figure src="images/heron_wistari_ridge_AI.png" title="Figure 1" >}}

First lets load in our R libraries

```
library(png) # For reading in our base image
library(ggplot2) # Plotting 
library(ggridges) # ggplot2 extension that enables ridgelines
library(dplyr) # Standard data wrangling
library(purrr) # Data mapping
library(reshape2) # Data melting
library(zoo) # Rolling means
```

Next lets set a seed so that we can recreate the same image again and again.

```
set.seed(1234)
```

Our starting image was retrieved from the [Sentinel-hub EO browser](https://apps.sentinel-hub.com/eo-browser/?lat=-23.5287&lng=151.8901&zoom=10&time=2019-07-29&preset=1_TRUE_COLOR&datasource=Sentinel-2%20L1C), using the L2A product, true colour bands (4, 3, and 2).

{{< figure src="/static/img/2019-01-20, Sentinel-2B L1C, True color.png" title="Heron_Wistari_sentinel" >}}

I then flattened the bands into a single channel grayscale image and boosted the contrast in a photo editor. This will create larger ridges then an image with a flatter profile. You can do this for in many different photo editors (photoshop, lightroom, gimp), and even free web tools: I like [Photopea](https://www.photopea.com/).

{{< figure src="/static/img/2019-01-20, Sentinel-2B L1C, True color_bw.png" title="Heron_Wistari_sentinel_bw" >}}

We then read the image file into R

```
heron_wistari <- readPNG("2019-01-20, Sentinel-2B L1C, True color.png") # Tip: Image needs a lot of contrast
```

The next step is to convert the image into long format. Here we use the reshape2 function melt(). At the same time, we use dplyrs mutate() function to add some noise to the pixel values so that the ridgelines, even in areas of the image that had little colour variation (e.g. in the deep water), have some character.

```
hw_df <- heron_wistari %>% 
  reshape2::melt(varnames = c("rows","cols")) %>%
  mutate(pixel = value + rnorm(length(value), sd = 0.01),
  pixel = case_when(pixel > 0 ~ pixel, TRUE ~ 0)
  )
```

Next we do some filtering, again using dplyr. Without this step we would have a ridgeline for every row (equal to the image height) of pixels in the original image. hw_df$rows is the column that contains the row coordinates, so we filter the rows in sequence from 0 to 421, which is the image height, and keep only 1 row in 8. You can play around with this step until you get the look you are after.

Here we also traverse the rows by grouping by rows, splitting the dataframe to contain individual rows, and use zoo rollmean() to smooth out the pixel colour data.

```
hw_df <- hw_df %>%
  filter(rows %in% seq(0, 421, 8)) %>% # Change to equal height of image
  group_by(rows) %>% 
  split(.$rows) %>% 
  purrr::map_df(~ {
    mutate(., pixel = zoo::rollmean(pixel, k = 20, fill = 0))
  })
```

All that is left is to call ggplot2, pipe in the data (note that the cols have to be sent in reverse order to how we have them in the dataframe), and map pixels to the geom_ridgeline() height parameter.

Most of the code below is just fiddling around until you find the aesthetic you want.

```
theme_color <- "#002b36"

hw_df %>% 
  ggplot() + 
  aes(cols, -rows, height = pixel, group = rows) + 
  geom_ridgeline(
    scale = 70, 
    alpha = 0.6,
    color = "#d2f4ff",
    fill = "#77ddff") +
  theme_minimal() +
  theme(legend.position = "none",
    aspect.ratio = 421/1263,
    axis.text = element_blank(),
    panel.grid = element_blank(),
    panel.grid.major.x = element_blank(),
    panel.grid.minor.x = element_blank(),
    axis.ticks = element_blank(),
    axis.line = element_blank(), axis.title = element_blank(),
    plot.background = element_rect(fill = theme_color, color = NA))
```

{{< figure src="/static/img/heron_wistari_ridge.png" title="Heron_Wistari_ggridges_draft" >}}

