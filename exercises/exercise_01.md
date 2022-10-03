Intro to Bayesian Statistic: Basics and distributions
================
Philipp Masur
2022-09

-   <a href="#basics-of-data-simulation"
    id="toc-basics-of-data-simulation">Basics of data simulation</a>
    -   <a href="#creating-vectors-of-numbers"
        id="toc-creating-vectors-of-numbers">Creating vectors of numbers</a>
    -   <a href="#sampling-from-a-vector"
        id="toc-sampling-from-a-vector">Sampling from a vector</a>
    -   <a href="#replicating-procedures"
        id="toc-replicating-procedures">Replicating procedures</a>
-   <a href="#sampling-from-distributions"
    id="toc-sampling-from-distributions">Sampling from distributions</a>
    -   <a href="#random-generation-of-numbers-drawn-from-a-distribution"
        id="toc-random-generation-of-numbers-drawn-from-a-distribution">Random
        generation of numbers drawn from a distribution</a>
    -   <a href="#density-functions" id="toc-density-functions">Density
        functions</a>
-   <a href="#the-coin-toss-example" id="toc-the-coin-toss-example">The coin
    toss example</a>
    -   <a href="#prior-belief" id="toc-prior-belief">Prior belief</a>
    -   <a href="#likelihood" id="toc-likelihood">Likelihood</a>
    -   <a href="#posterior" id="toc-posterior">Posterior</a>

In this tutorial, we are going to explore some basics of Bayesian
statistics, including exercises to understand what it means to sample
from a distribution, mathematically deriving posterior distributions,
etc. If you are new to Bayesian statistics, consider working through
[these slides]() which introduce the basic concepts and exemplify
Bayesian Analysis based on real-world data.

# Basics of data simulation

Before we engage with some Bayesian thinking, let us explore a bit how
we can simulate data in R. All of these functions are included in base
R, so no need for extra packages.

## Creating vectors of numbers

We can of course create data by simply using the function `c()` (=
combine) to create a vector. Naturally it would be to tedious to
literally create all vectors by hand. A first simple solution that is
often handy is to repeat certain values with the function `rep()`. Also,
don’t forget that we can create consecutive numbers (or letters) with
the colon.

``` r
# A vector of 6 values
c(1, 1, 1, 0, 0, 0)
c(rep(1, 3), rep(0, 3))

# Consecutive numbers
c(1:10)
```

## Sampling from a vector

Data generation processes often have a certain randomness. For example,
it could be the case that we want to “sample” one or several numbers
from a larger vector. For this, we can use the function `sample()`,
which takes the vector as the first argument and the size (= numbers of
items to choose) as the second argument.

``` r
# Random numbers from a vector
sample(c(1:100), size = 2)
```

## Replicating procedures

If we want to repeatedly sample from a vector (or later from a
distribution), we need to “replicate” the procedure. Bear in mind:
Replicating using the function `replicate()` is not the same thing as
repeating using the function `rep()`!

``` r
# Repeating
rep(sample(c(1:100),2), 2)

# Replicating
replicate(n = 2, expr = sample(c(1:100), 2))
replicate(n = 2, expr = sample(c(1:100), 2), simplify = F)
replicate(n = 2, expr = sample(c(1:100), 2)) %>% as.vector
```

# Sampling from distributions

For exemplifying some Bayesian Basics, it is important to understand
that we can also sample values from a distribution (that can be
mathematically described).

## Random generation of numbers drawn from a distribution

R provides several functions that all work in the same way and allow to
produce different types of distribution.

``` r
# Uniform distribution
runif(n = 1, min = 0, max = 10)

# Normal (gaussian) distribution
rnorm(n = 1, mean = 0, sd = 1)
rnorm(10, 5, 2)

# Beta distribution
rbeta(10, 1, 1)

# Binomial distribution
rbinom(n = 1, size = 10, .5)
replicate(10, rbinom(n = 1, size = 10, .5))
```

**Question:** Produce a beta distribution with n = 1000 that corresponds
to a normal distribution. How would you plot this distribution? (Don’t
forget to load packages that you might need)

``` r
# Code here
```

## Density functions

Remember the coin toss? How can we estimate the probability of observing
4 heads out of 10 tosses if the probability of success is assumed to be
50%? Well, the `dinom()` function can help us to estimate the
probability!

``` r
dbinom(x = 4, size = 10, prob = .5)
```

To visualize the full binomial distribution (aka the likelhood for the
coin toss example), we need to estimate the probability of observing 4
heads out of 10 tosses for all potential success probabilities.

``` r
# Create a sequence of probabilites from 0 to 1
x <- seq(0, 1, by = .001)

# Estimate the probabilities of observing 4 out of 10 for all success probabilities
y <- dbinom(4, 10, x)

# Plot
ggplot(NULL, aes(x = x, y = y)) +
  geom_line() +
  theme_classic()
```

**Question:** How could we produce a beta density plot with the shapes
![\alpha](https://latex.codecogs.com/png.image?%5Cdpi%7B110%7D&space;%5Cbg_white&space;%5Calpha "\alpha")
= 2 and
![\beta](https://latex.codecogs.com/png.image?%5Cdpi%7B110%7D&space;%5Cbg_white&space;%5Cbeta "\beta")
= 20 across probabilities from 0 to 1? How would you plot this?

``` r
# Code here
```

# The coin toss example

Let us recompute the coin toss example that we discussed in the lecture
earlier.

## Prior belief

In a first step, let us compute our prior belief that the coin is
probably fair. In other words, there should be a high probability that
we observe heads 50% of the time. We can do so by creating a beta
distribution, centered around .50 and a steep decrease for values lower
and higher. We immediately standardize the prior values to make sure we
can compare them to a potential likelihood later.

``` r
d <- tibble(prob = seq(0, 1, by = .001),     
            prior = dbeta(prob, 30, 30)) %>%  
  mutate(prior = prior/sum(prior))   # standardization

# Plot
d %>%
  ggplot(aes(x = prob, 
             y = prior)) +
  geom_line(color = "red") +
  theme_minimal() +
  labs(x = expr(theta), 
       y = "", 
       title = "Prior",
       subtitle = "Beta(30, 30)")
```

**Question:** How do we need to change the code to make our prior belief
less (or more) certain?

``` r
# Code here
```

## Likelihood

As learned earlier, we write code the simulates a coin toss. By
repeatedly sampling from a vector with two options (“heads” and
“tails”), we get a vector that represents our coin tosses. I am using
the function `set.seed()` here to make this coin tossing reproducable.
Feel free to use a different value and see how it affects the outcome.

``` r
# Coin toss
set.seed(5)
toss <- sample(c("tails", "heads"), size = 10, replace = T)

# Summary
table(toss)

# Frequency Plot
ggplot(NULL, 
       aes(x = toss)) + 
  geom_bar(fill = "steelblue", width = .5) + 
  scale_y_continuous(n.breaks = 5, limits = c(0, 8)) +
  theme_minimal() +
  labs(x = "")
```

This is our date (= observable evidence). Now we would like to compute a
likelihood distribution that reflects this observation. As we learned,
coin tosses can be represented via a binomial distribution (remember
logistic regression?). In my case, I observed heads only twice in 10
trials. I hence use the function `dbinom()` and enter these values as
first to arguments. We again standardize to make it comparable to the
prior distribution (aka putting them on the same scale).

``` r
# Compute likelihood
d <- d %>%
  mutate(likelihood = dbinom(x = table(toss)[1], size = 10, prob = prob), 
         likelihood = likelihood/sum(likelihood)) 

# Plot
d %>%
  gather(key, value, -prob) %>%
  ggplot(aes(x = prob, y = value, color = key)) +
  geom_line() +
  scale_color_manual(values = c("darkblue", "red")) +
  theme_minimal() +
  labs(x = expr(theta), 
       y = "", 
       color = "",
       title = "Prior & Likelihood")
```

As we can see, our likelihood is quite different from our prior
assumption. After all, based on our 10 trials, we can’t really assume
that we have a fair coin (but, we all know that sample size matters of
course!).

## Posterior

How can we now compute the posterior distribution? Simply by multiplying
the prior with the likelihood. This works, because the beta distribution
(prior) is a conjugate prior for the binomial distribution (likelihood).
By multiplying them, we get another beta distribution. We again
standardize to make all three comparable.

``` r
d <- d %>%
  mutate(posterior = (prior*likelihood)/sum(prior*likelihood))

d %>%
  gather(key, value, -prob) %>%
  mutate(factor(key, 
         levels = c("prior", "likelihood", "posterior"))) %>%
  ggplot(aes(x = prob, y = value, color = key)) +
  geom_line() +
  scale_color_manual(values = c("darkblue", "black", "red")) +
  theme_minimal() +
  labs(x = expr(theta), y = "", color = "",
       title = "Prior, likelihood & posterior")
```

As we can see, the likelihood adjusted the prior belief slightly. The
posterior distribution now suggest a somewhat biased coin that shows
tails slightly more often than heads.

Yet, it should be clearly visible that we do not fully base our
posterior belief on the data, the prior actually has a considerably
influence on our posterior belief. And why should it not? I am sure
everyone will agree that the data here is less informative than our
experience of how coins generally fall…

**Question:** Let’s practice. Let’s say we roll the dice again ten
times. Can we use our previous posterior as new prior and our new data
as new likelihood? How does the posterior look like after ? And what
happens if we toss the coin 100 times?

Remember, we can computer the posterior beta distribution like so:

![beta(30, 30) \* Binomial(2, 10, \theta)](https://latex.codecogs.com/png.image?%5Cdpi%7B110%7D&space;%5Cbg_white&space;beta%2830%2C%2030%29%20%2A%20Binomial%282%2C%2010%2C%20%5Ctheta%29 "beta(30, 30) * Binomial(2, 10, \theta)")

![beta(2 + 30, 10-2 + 30)](https://latex.codecogs.com/png.image?%5Cdpi%7B110%7D&space;%5Cbg_white&space;beta%282%20%2B%2030%2C%2010-2%20%2B%2030%29 "beta(2 + 30, 10-2 + 30)")

``` r
# Code here
```
