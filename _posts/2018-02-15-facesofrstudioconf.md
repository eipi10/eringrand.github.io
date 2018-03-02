---
layout: post
title: Faces of rstudioconf
description: ""
tags: rstats twitter rstudioconf
---

I was reminded today by [Daniela](https://twitter.com/d4tagirl) that everyone should blog - and on top of that you can totally blog something small and simple just to get something out there.

In the spirit of small and simple, last week I tweeted out this cool image...

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">I am writing up my notes from <a href="https://twitter.com/hashtag/rstduiconf?src=hash&amp;ref_src=twsrc%5Etfw">#rstduiconf</a> (blog post coming!) and wanted to have a quick picture to go with my thoughts. Remembering <a href="https://twitter.com/ma_salmon?ref_src=twsrc%5Etfw">@ma_salmon</a> &#39;s post on Faces of R (here: <a href="https://t.co/C1sRW3hwVL">https://t.co/C1sRW3hwVL</a> ), I decided to make a Faces of Rstudioconf! <a href="https://t.co/vtNVn2RyHV">pic.twitter.com/vtNVn2RyHV</a></p>&mdash; Erin Grand (@astroeringrand) <a href="https://twitter.com/astroeringrand/status/961466502821052416?ref_src=twsrc%5Etfw">February 8, 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

I have not yet organized all my thoughts from the conference (spoilers, it was awesome, I learned so much!), but that will not stop by from posting the code I borrowed from [Maelle](https://twitter.com/ma_salmon) to create the pretty image. So, here you go!

```r
library(rtweet)
library(magick)
library(tidyverse)
library(gmp)
```

```{r}
search_terms <- c("rstudioconf", "rstudioconf2018")

tweets <- purrr::map_df(search_terms, search_tweets, n=2000, include_rts=FALSE, parse=TRUE) 

users_tweets <- lookup_users(unique(tweets$user_id))

users <- users_tweets %>%
  select(user_id, 
         profile_image_url, 
         screen_name,
         name, 
         followers_count, 
         profile_image_url
         ) %>%
  distinct()

save_image <- function(df){
  image <- try(image_read(df$profile_image_url), silent = FALSE)
  if(class(image)[1] != "try-error"){
    image %>%
      image_scale("50x50") %>%
      image_write(paste0("~pictures/", df$screen_name,".jpg"))
  }
}

users <- filter(users, !is.na(profile_image_url))
users_list <- split(users, 1:nrow(users))
walk(users_list, save_image)


files <- dir("pictures/", full.names = TRUE)
set.seed(42)
files <- sample(files, length(files))
gmp::factorize(length(files))

no_rows <- 14
no_cols <- 31

make_column <- function(i, files, no_rows){
  image_read(files[(i*no_rows+1):((i+1)*no_rows)]) %>%
  image_append(stack = TRUE) %>%
    image_write(paste0("cols/", i, ".jpg"))
}

walk(0:(no_cols-1), make_column, files = files, no_rows = no_rows)

image_read(dir("cols/", full.names = TRUE)) %>%
image_append(stack = FALSE) %>%
  image_write("2018-02-7-facesofrstudioconf.jpg")
```

![](https://github.com/eringrand/projects/blob/master/Rstudio%20Conf%20Twitter%20Pictures/2018-02-7-facesofnasadatanauts.jpg?raw=true)