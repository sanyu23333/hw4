hw4
================
Xuan Huang
2022-11-18

# HPC

## Problem 1: Make sure your code is nice

Rewrite the following R functions to make them faster. It is OK (and
recommended) to take a look at Stackoverflow and Google

``` r
# Total row sums
fun1 <- function(mat) {
  n <- nrow(mat)
  ans <- double(n) 
  for (i in 1:n) {
    ans[i] <- sum(mat[i, ])
  }
  ans
}

fun1alt <- function(mat) {
  answer1 <- rowSums(mat)
  answer1
}

# Cumulative sum by row
fun2 <- function(mat) {
  n <- nrow(mat)
  k <- ncol(mat)
  ans <- mat
  for (i in 1:n) {
    for (j in 2:k) {
      ans[i,j] <- mat[i, j] + ans[i, j - 1]
    }
  }
  ans
}

fun2alt <- function(mat) {
  answer2 <- t(apply(mat, 1, cumsum))
  answer2
}


# Use the data with this code
set.seed(2315)
dat <- matrix(rnorm(200 * 100), nrow = 200)

# Test for the first
microbenchmark::microbenchmark(
  fun1(dat),
  fun1alt(dat), unit = "milliseconds", check = "equivalent"
)
```

    ## Warning in microbenchmark::microbenchmark(fun1(dat), fun1alt(dat), unit =
    ## "milliseconds", : less accurate nanosecond times to avoid potential integer
    ## overflows

    ## Unit: milliseconds
    ##          expr      min        lq       mean    median        uq      max neval
    ##     fun1(dat) 0.161704 0.1715235 0.17843364 0.1749675 0.1829215 0.252273   100
    ##  fun1alt(dat) 0.004961 0.0054530 0.01259602 0.0058425 0.0065805 0.642224   100

``` r
# Test for the second
microbenchmark::microbenchmark(
  fun2(dat),
  fun2alt(dat), unit = "milliseconds", check = "equivalent"
)
```

    ## Unit: milliseconds
    ##          expr      min       lq      mean    median       uq      max neval
    ##     fun2(dat) 1.177643 1.185351 1.2075734 1.1956830 1.214461 1.343488   100
    ##  fun2alt(dat) 0.279989 0.330296 0.4787918 0.3452815 0.369000 6.725517   100

``` r
# The last argument, check = “equivalent”, is included to make sure that the functions return the same result.
```

## Problem 2: Make things run faster with parallel computing

The following function allows simulating PI

``` r
sim_pi <- function(n = 1000, i = NULL) {
  p <- matrix(runif(n*2), ncol = 2)
  mean(rowSums(p^2) < 1) * 4
}

# Here is an example of the run
set.seed(156)
sim_pi(1000) # 3.132
```

    ## [1] 3.132

In order to get accurate estimates, we can run this function multiple
times, with the following code:

``` r
# This runs the simulation a 4,000 times, each with 10,000 points
set.seed(1231)
system.time({
  ans <- unlist(lapply(1:4000, sim_pi, n = 10000))
  print(mean(ans))
})
```

    ## [1] 3.14124

    ##    user  system elapsed 
    ##   0.691   0.189   0.890

Rewrite the previous code using parLapply() to make it run faster. Make
sure you set the seed using clusterSetRNGStream():

``` r
library(parallel)

system.time({
  cl <- makePSOCKcluster(4L)
  clusterSetRNGStream(cl = cl, iseed = 1231)  
  ans <- unlist(parLapply(cl = cl, 1:4000, sim_pi, n = 10000))
  print(mean(ans))
  stopCluster(cl)
  ans
})
```

    ## [1] 3.141578

    ##    user  system elapsed 
    ##   0.004   0.002   0.503

# SQL

-   Setup a temporary database by running the following chunk

``` r
# install.packages(c("RSQLite", "DBI"))

library(RSQLite)
library(DBI)

# Initialize a temporary in memory database
con <- dbConnect(SQLite(), ":memory:")

# Download tables
film <- read.csv("https://raw.githubusercontent.com/ivanceras/sakila/master/csv-sakila-db/film.csv")
film_category <- read.csv("https://raw.githubusercontent.com/ivanceras/sakila/master/csv-sakila-db/film_category.csv")
category <- read.csv("https://raw.githubusercontent.com/ivanceras/sakila/master/csv-sakila-db/category.csv")

# Copy data.frames to database
dbWriteTable(con, "film", film)
dbWriteTable(con, "film_category", film_category)
dbWriteTable(con, "category", category)
```

## Question 1

How many many movies is there avaliable in each rating catagory.

``` sql
SELECT rating, COUNT(*) AS count
FROM film
GROUP BY rating
```

| rating | count |
|:-------|------:|
| G      |   180 |
| NC-17  |   210 |
| PG     |   194 |
| PG-13  |   223 |
| R      |   195 |

5 records

## Question 2

What is the average replacement cost and rental rate for each rating
category.

``` sql
SELECT 
  rating, 
  AVG(replacement_cost) AS averageRepCost,
  AVG(rental_rate) AS averageRentalRate
FROM film
GROUP BY rating
```

| rating | averageRepCost | averageRentalRate |
|:-------|---------------:|------------------:|
| G      |       20.12333 |          2.912222 |
| NC-17  |       20.13762 |          2.970952 |
| PG     |       18.95907 |          3.051856 |
| PG-13  |       20.40256 |          3.034843 |
| R      |       20.23103 |          2.938718 |

5 records

## Question 3

Use table film_category together with film to find the how many films
there are witth each category ID

``` sql
SELECT f.category_id, count(f.category_id) AS NOfFilms
FROM film_category AS f
  INNER JOIN film AS g ON f.film_id = g.film_id
GROUP BY f.category_id
```

| category_id | NOfFilms |
|:------------|---------:|
| 1           |       64 |
| 2           |       66 |
| 3           |       60 |
| 4           |       57 |
| 5           |       58 |
| 6           |       68 |
| 7           |       62 |
| 8           |       69 |
| 9           |       73 |
| 10          |       61 |

Displaying records 1 - 10

## Question 4

Incorporate table category into the answer to the previous question to
find the name of the most popular category.

``` sql
SELECT f.category_id, h.name, count(f.category_id) AS NOfFilms
FROM film_category AS f
  INNER JOIN film AS g ON f.film_id = g.film_id
  INNER JOIN category AS h ON f.category_id = h.category_id
GROUP BY f.category_id
ORDER BY NOfFilms DESC
```

| category_id | name        | NOfFilms |
|------------:|:------------|---------:|
|          15 | Sports      |       74 |
|           9 | Foreign     |       73 |
|           8 | Family      |       69 |
|           6 | Documentary |       68 |
|           2 | Animation   |       66 |
|           1 | Action      |       64 |
|          13 | New         |       63 |
|           7 | Drama       |       62 |
|          14 | Sci-Fi      |       61 |
|          10 | Games       |       61 |

Displaying records 1 - 10

Based on the result, the most popular category is sports.
