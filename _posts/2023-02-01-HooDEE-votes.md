---
layout: post
title: HooDEE counting of votes!
author: "Ehouarn Le Faou"
date: 2023-02-01
tag: "mix Various"
image: "/img/blog/blog4.png"
---

# Data

First of all, let’s remember the logos that were voted on:

<img src="{{ site.baseurl }}/img/blog/HooDEE_Counting-of-votes_files/figure-markdown_strict/unnamed-chunk-7-1.png"  style="width:50.0%" />

The data table collected is as follows: each voter (row) gave each logo
(column) a note, which is an integer between 0 and 6 (rows), 0 being the
worst score and 6 the best. Here is an extract:

    data_raw = read.table("votes.txt", h = T)
    head(data_raw)

    ##   logo1 logo2 logo3 logo4 logo5 logo6 logo7 logo8 logo9 logo10
    ## 1     5     4     6     1     2     1     2     0     0      0
    ## 2     0     0     0     3     2     4     1     6     0      5
    ## 3     2     6     6     6     5     3     1     2     3      1
    ## 4     0     0     0     3     6     3     0     0     0      0
    ## 5     4     3     3     1     5     4     4     3     0      3
    ## 6     1     1     0     1     6     4     1     1     1      1

No data transformation is required.

The number of voters who participated is determined:

    nObs = nrow(data_raw)
    print(nObs)

    ## [1] 122

Which is quite a success! Unfortunately, we did not set up systems to
prevent ballot box stuffing, since each participant had the opportunity
to vote more than once if they wanted to. We hope that the unfairness of
this kind of behavior was widely recognized and that no cheating was
committed.

# Ranking

To establish the outcome of a majority judgment vote, it is simply
necessary to establish the median of the notes given to each proposal.
The proposal with the highest median wins. In the case of a tie, the
percentage of votes above this median is used to break it.

Here we go:

    data.res = data_raw %>% 
      gather("logo", "note", 1:10) %>% 
      group_by(logo) %>%
      summarise(med = median(note)) %>%
      arrange(desc(med)) %>%
      mutate(rank = rank(-med))

    print(data.res)

    ## # A tibble: 10 × 3
    ##    logo     med  rank
    ##    <chr>  <dbl> <dbl>
    ##  1 logo1      3   3.5
    ##  2 logo3      3   3.5
    ##  3 logo4      3   3.5
    ##  4 logo5      3   3.5
    ##  5 logo6      3   3.5
    ##  6 logo8      3   3.5
    ##  7 logo10     2   7.5
    ##  8 logo2      2   7.5
    ##  9 logo7      1   9  
    ## 10 logo9      0  10

------------------------------------------------------------------------

<strong>Technical details:</strong> this pipe starts by rearranging the
table by creating only two columns, a `logo` column and a `note` column.
Each row corresponds to a note given to a logo by a voter. Then the
function `group_by` will allow to make calculations by grouping the sets
of values to be considered together by logo: we treat the notes of each
logo separately. The function `summarise` allows then to calculate the
median of the notes (by logo, thanks to `group_by`). This creates a
smaller table where each logo will be represented by a row and where we
will have a logo column and a median column. We then use the function
`arrange` to sort the rows in descending order according to the median.
Finally we use the function `mutate` to add a column `rank` which
assigns to each row according to the median column a rank.

We notice that there is an equality between 6 logos, which all have the
same median.

We therefore redo the analysis by taking into account the proportion of
scores above the median:

    data.res = data_raw %>% 
      gather("logo", "note", 1:10) %>% 
      group_by(logo) %>%
      summarise(med = median(note),
                prop = sum(note>med)/nObs) %>%
      arrange(desc(med+prop)) %>%
      mutate(rank = rank(-med-prop))

    print(data.res) 

    ## # A tibble: 10 × 4
    ##    logo     med  prop  rank
    ##    <chr>  <dbl> <dbl> <dbl>
    ##  1 logo4      3 0.475   1.5
    ##  2 logo5      3 0.475   1.5
    ##  3 logo3      3 0.443   3.5
    ##  4 logo6      3 0.443   3.5
    ##  5 logo1      3 0.393   5.5
    ##  6 logo8      3 0.393   5.5
    ##  7 logo10     2 0.475   7  
    ##  8 logo2      2 0.434   8  
    ##  9 logo7      1 0.393   9  
    ## 10 logo9      0 0.467  10

There is still a tie, but now only between logos 4 and 5.

We therefore redo the analysis by looking at the proportion of scores
above the median + 1.

    data.res = data_raw %>% 
      gather("logo", "note", 1:10) %>% 
      group_by(logo) %>%
      summarise(med = median(note),
                prop = sum(note>(med+1))/nObs) %>%
      arrange(desc(med+prop)) %>%
      mutate(rank = rank(-med-prop))

    print(data.res) 

    ## # A tibble: 10 × 4
    ##    logo     med  prop  rank
    ##    <chr>  <dbl> <dbl> <dbl>
    ##  1 logo5      3 0.369     1
    ##  2 logo4      3 0.352     2
    ##  3 logo3      3 0.311     3
    ##  4 logo6      3 0.303     4
    ##  5 logo8      3 0.238     5
    ##  6 logo1      3 0.180     6
    ##  7 logo10     2 0.336     7
    ##  8 logo2      2 0.172     8
    ##  9 logo7      1 0.328     9
    ## 10 logo9      0 0.262    10

And we have a winner! It is the logo 5.

# Plotting

Here are some more or less interesting plots to have a graphic
visualization of these results.

## PCA

PCA on the votes, each point is a vote. The proximity of the
corresponding ellipses of two logos represents the tendency of voters to
vote identically for them.

    res.CA<-CA(data_raw,graph=FALSE)
    ellipseCA(res.CA,ellipse=c('col'),title="PCA",col.row='#050505',col.col='#0C9C16')

![](/img/blog/HooDEE_Counting-of-votes_files/figure-markdown_strict/unnamed-chunk-7-1.png)

## Stacked barplot

Distribution in stacked barplot of the note distribution for each logo:

    data.plot1 = data_raw %>% 
      gather("logo", "note", 1:10) %>% 
      group_by(logo, note) %>%
      summarise(n = n()) %>%
      full_join(expand.grid(logo = paste0("logo", 1:10), note = 0:6, n2 = 1)) %>%
      select(-n2) %>%
      mutate(n = replace(n, is.na(n), 0),
             note = as.factor(note))

    data.plot1$logo <- factor(data.plot1$logo, levels = data.res$logo)

    g1 = ggplot(data.plot1, aes(x = logo, y = n, fill = note)) + 
      geom_bar(position="stack", stat="identity") +
      geom_hline(yintercept = nObs/2) + 
      scale_fill_brewer(palette = "Greens") +
      scale_y_reverse() +
      ggtitle("Distribution of notes by logo, in order of placement in the ranking")

    print(g1)

![](/img/blog/HooDEE_Counting-of-votes_files/figure-markdown_strict/unnamed-chunk-9-1.png)

## Histogram

Distribution of notes in the form of a histogram.

    data.plot2 = data_raw %>% 
      gather("logo", "note", 1:10)

    data.plot2$logo <- factor(data.plot2$logo, levels = data.res$logo)

    g2 = ggplot(data.plot2, aes(x = note, y = ..density.., fill = logo)) + 
      geom_histogram(bins = 7) +
      facet_wrap(facets = vars(logo), nrow = 2, ncol = 5) +
      scale_fill_manual(values = colorRampPalette(c("darkgreen", "grey"))(10)) +
      ggtitle("Distribution of notes by logo")
      
    print(g2)

![](/img/blog/HooDEE_Counting-of-votes_files/figure-markdown_strict/unnamed-chunk-10-1.png)
