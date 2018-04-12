zero length groups
================
Romain François
12/04/2018

## install

This is based on the `feature-341-zero-length-groups`
branch:

``` r
devtools::install_github("tidyverse/dplyr", ref = "feature-341-zero-length-groups")
```

## Motivation

See the [full discussion](https://github.com/tidyverse/dplyr/issues/341)
and the original [stackoverflow
question](http://stackoverflow.com/questions/22523131).

`dplyr` had the `drop` attribute in grouped data frames all along, but
it did not really do anything about it, so in effect it has always been
like if `drop` was set to TRUE.

`drop=FALSE` is about keeping groups even if they have no
data.

``` r
df <- data.frame( x = 1:2, g = factor(c("a", "b"), levels = c("a", "b", "c")))
df
```

    ##   x g
    ## 1 1 a
    ## 2 2 b

``` r
tally(group_by(df, g, drop = TRUE))
```

    ## # A tibble: 2 x 2
    ##   g         n
    ##   <fct> <int>
    ## 1 a         1
    ## 2 b         1

``` r
tally(group_by(df, g, drop = FALSE))
```

    ## # A tibble: 3 x 2
    ##   g         n
    ##   <fct> <int>
    ## 1 a         1
    ## 2 b         1
    ## 3 c         0

With `drop=FALSE` the level “c” appears in the summary, with n=0.

## Which groups to keep

The problem is that we don’t always group on factors, and so the
question that has kept the 341 issue alive for 4 years is which groups.

``` r
df <- data.frame( 
  x  = 1:8,
  y  = rep(1:4, each=2),
  f1 = factor( rep(c("a", "b"), each = 4), levels = c("a", "b", "c")) 
)
df
```

    ##   x y f1
    ## 1 1 1  a
    ## 2 2 1  a
    ## 3 3 2  a
    ## 4 4 2  a
    ## 5 5 3  b
    ## 6 6 3  b
    ## 7 7 4  b
    ## 8 8 4  b

  - f1 has two distinct values in the data, but 3 levels
  - y has 4 distinct values, but only two for the level “a” of f1 and 2
    for the “b” level, and obviously 0 for the “c” level

<!-- end list -->

``` r
group_by(df, f1, drop = FALSE) %>% 
  summarise( n_distinct(y) )
```

    ## # A tibble: 3 x 2
    ##   f1    `n_distinct(y)`
    ##   <fct>           <int>
    ## 1 a                   2
    ## 2 b                   2
    ## 3 c                   0

So I’m not sure about what happens when we group by f1 and y, and when
we group by y and f1.

What we have currently in the `feature-341-zero-length-groups` is the
cartesian product of levels of factors and unique values of non factors,
so in that case we end with 12 groups in both case

``` r
group_by(df, y, f1, drop = FALSE) %>% tally() %>% arrange(desc(n))
```

    ## # A tibble: 12 x 3
    ## # Groups:   y [4]
    ##        y f1        n
    ##    <int> <fct> <int>
    ##  1     1 a         2
    ##  2     2 a         2
    ##  3     3 b         2
    ##  4     4 b         2
    ##  5     1 b         0
    ##  6     1 c         0
    ##  7     2 b         0
    ##  8     2 c         0
    ##  9     3 a         0
    ## 10     3 c         0
    ## 11     4 a         0
    ## 12     4 c         0

``` r
group_by(df, f1, y, drop = FALSE) %>% tally() %>% arrange(desc(n))
```

    ## # A tibble: 12 x 3
    ## # Groups:   f1 [3]
    ##    f1        y     n
    ##    <fct> <int> <int>
    ##  1 a         1     2
    ##  2 a         2     2
    ##  3 b         3     2
    ##  4 b         4     2
    ##  5 a         3     0
    ##  6 a         4     0
    ##  7 b         1     0
    ##  8 b         2     0
    ##  9 c         1     0
    ## 10 c         2     0
    ## 11 c         3     0
    ## 12 c         4     0

In the thread issue, there are a lot of “we should keep all the groups
only for factors”, but I have no idea what to make of this.

## verbs support

``` r
gdf <- data.frame( 
  x  = 1:8,
  y  = rep(1:4, each=2),
  f1 = factor( rep(c("a", "b"), each = 4), levels = c("a", "b", "c")) 
) %>% group_by(f1, y, drop = FALSE)
```

With minor changes, here is what dplyr verbs do in the presence of 0
length groups:

### filter

the attributes are recalculated after the filtering, so we keep all the
levels of factors but might lose groups that are not from factors.

``` r
filter( gdf, x > 4)
```

    ## # A tibble: 4 x 3
    ## # Groups:   f1, y [6]
    ##       x     y f1   
    ##   <int> <int> <fct>
    ## 1     5     3 b    
    ## 2     6     3 b    
    ## 3     7     4 b    
    ## 4     8     4 b

the alternative would be to keep all the groups. might be tricky.

### mutate

mutate executes the expression for each group so the same rule as usual
applies, the result of the expression must be either length 0 (size of
the group) or length 1 so that it’s recycled 0 times 😱

``` r
mutate( gdf, x = print(x))
```

    ## [1] 1 2
    ## [1] 3 4
    ## integer(0)
    ## integer(0)
    ## integer(0)
    ## integer(0)
    ## [1] 5 6
    ## [1] 7 8
    ## integer(0)
    ## integer(0)
    ## integer(0)
    ## integer(0)

    ## # A tibble: 8 x 3
    ## # Groups:   f1, y [12]
    ##       x     y f1   
    ##   <int> <int> <fct>
    ## 1     1     1 a    
    ## 2     2     1 a    
    ## 3     3     2 a    
    ## 4     4     2 a    
    ## 5     5     3 b    
    ## 6     6     3 b    
    ## 7     7     4 b    
    ## 8     8     4 b

### summarise

summarise applies the expression to each group, e.g.

``` r
summarise( gdf, nx = n_distinct(x) )
```

    ## # A tibble: 12 x 3
    ## # Groups:   f1 [?]
    ##    f1        y    nx
    ##    <fct> <int> <int>
    ##  1 a         1     2
    ##  2 a         2     2
    ##  3 a         3     0
    ##  4 a         4     0
    ##  5 b         1     0
    ##  6 b         2     0
    ##  7 b         3     2
    ##  8 b         4     2
    ##  9 c         1     0
    ## 10 c         2     0
    ## 11 c         3     0
    ## 12 c         4     0

### arrange

The metadata attributes are recalculated. No issue.

### bind\_rows

The metadata attribute are calculated after the values are bind. The
`drop` attribute of the first object decides.

### joins

joins only make lazy grouped data frames. The drop attribute is passed
through, so when the metadata is materialised later it respect `drop`.
