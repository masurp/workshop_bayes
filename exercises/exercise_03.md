Intro to Bayesian Statistic: Simple Regression Analysis
================
Philipp Masur
2022-09

-   <a href="#introduction" id="toc-introduction">Introduction</a>
-   <a href="#preparation" id="toc-preparation">Preparation</a>
    -   <a href="#loading-packages" id="toc-loading-packages">Loading
        packages</a>
    -   <a href="#loading-and-recoding-the-data"
        id="toc-loading-and-recoding-the-data">Loading and recoding the data</a>
    -   <a href="#understanding-the-data-set"
        id="toc-understanding-the-data-set">Understanding the data set</a>
-   <a href="#simple-regression-analysis"
    id="toc-simple-regression-analysis">Simple Regression Analysis</a>
    -   <a href="#fitting-a-simple-regression-with-brms"
        id="toc-fitting-a-simple-regression-with-brms">Fitting a simple
        regression with <code>brms</code></a>
    -   <a href="#setting-priors" id="toc-setting-priors">Setting priors</a>
-   <a href="#convergence-diagnostics"
    id="toc-convergence-diagnostics">Convergence diagnostics</a>
    -   <a href="#trace-plots" id="toc-trace-plots">Trace plots</a>
    -   <a href="#rhat" id="toc-rhat">Rhat</a>
    -   <a href="#autocorrelation" id="toc-autocorrelation">Autocorrelation</a>
-   <a href="#bayesian-inference" id="toc-bayesian-inference">Bayesian
    Inference</a>
    -   <a href="#highest-density-interval-hdi"
        id="toc-highest-density-interval-hdi">Highest Density Interval (HDI)</a>
    -   <a href="#region-of-practical-equivalence-rope"
        id="toc-region-of-practical-equivalence-rope">Region of Practical
        Equivalence (ROPE)</a>
    -   <a href="#probability-of-direction"
        id="toc-probability-of-direction">Probability of direction</a>
    -   <a href="#posterior-predictive-check"
        id="toc-posterior-predictive-check">Posterior Predictive Check</a>
-   <a href="#exercise" id="toc-exercise">Exercise</a>

# Introduction

This tutorial introduces the basic model fitting process as implemented
in [`brms`](https://paul-buerkner.github.io/brms/). This package
provides an interface to fit Bayesian generalized (non-)linear
multivariate multilevel models using Stan. Lucky for us, the formula
syntax is very similar to that of the package `lme4` to provide a
familiar and simple interface for performing regression analyses. So if
you are familiar with fitting multilevel models in `lme4`, using `brms`
will be a no-brainer. The interesting part is when we work with the
posterior and add prior information to the models.

If you are interested in working through some more fundamental basics of
Bayesian Statistics, consider going through [this
tutorial](https://github.com/masurp/workshop_bayes/blob/main/exercises/exercise_01.md).

# Preparation

## Loading packages

For this tutorial, we will load a number of different packages. First,
we are going to load the `tidyverse` package set to have convenient data
wrangling and visualization functions available. **Attention:** Always
load the `tidyverse` before loading `brms`. For some weird reason, vice
versa will break the functionality of the `tidyverse`. Second, we thus
load the package `brms`, which is of course the core package for fitting
Bayesian regression analysis. We are also going to load the packages
`insight` and `bayestestR`, which both stem from the
[`easystats`](https://easystats.github.io/easystats/) package set. The
former provides functions to extract information from brms objects. The
latter provides functions for Bayesian inferences. We also are going to
load the package `bayesplot()`, which provides good functions for MCMC
diagnostic checks. Finally, we are going to load the handy package
`psych`, simply because it helps us understanding our data better.

``` r
library(tidyverse)
library(brms)
library(insight)
library(bayestestR)
library(bayesplot)
library(psych)
```

## Loading and recoding the data

We are going to use data by Masur, DiFranzo & Bazarova (2021), which we
encountered in the examples during the lecture. The data was collected
during an online experiment in 2019. The experiment was a 3x3
between-subject design that manipulated the prevaling *collective norm*
in relation to visual disclosure. Participants were randomized into 9
different groups and confronted with a realistic, but controlled social
media feed of \~50 posts. The feeds different with regard to a) the
number of posts that featured a visual disclosure, i.e., whether the
(fake) user showed his/her face in the post (0%, 20%, 80%), and b) the
number of profile pictures that featured a visual disclosure, i.e.,
whether the (fake) user showed his/her face in the profile picture (0%,
20%, 80%). After going and interacting with the feed, participants
completed a post-survey that included measures to assess norm
perceptions, intention to disclose oneself visually as well as some
control and potential moderators.

We published the data set openly on the Open Science Framework (osf.io).
The function `read.csv` can actually load such data sets directly into R
(without technically downloading it). We directly transform the data set
into a `tibble`, which is easier to handle compared to the standard
`data.frame`.

``` r
d <- read.csv("https://osf.io/kfm6x/download", header = TRUE, na.strings = "NA") %>% 
  as_tibble
head(d)
```

In this tutorial, we are only going to focus on a subset of the
variables. More specifically, we are going to investigate the effect of
the (1) manipulations on the (2) norm perceptions and the (3) intention
to disclose oneself. We will further add (4) the disposition for
critical thinking and some (4) socio-demographics as control variables.
In the code below, I am simply selecting the relevant variables based on
their names (the variables conveniently have been named in a
straight-forward manner) and pass those to the function `rowMeans`,
which automatically computes the mean index. As a final step, we select
only the variables that we are going to use in our analysis.

``` r
# Check variable naming
names(d)

# Recoding and selecting relevant variables
d <- d %>% 
  mutate(id = 1:n(),
         perceptions = rowMeans(d %>% select(contains("NOR"), -norm)),
         behavior = rowMeans(d %>% select(contains("BEH"))),
         crit_thinking = rowMeans(d %>% select(contains("CRI_THI"))), na.rm = T) %>%
  select(id, age, gender, edu, condition, norm, profile, perceptions, behavior, crit_thinking)

# New data set
head(d)
```

## Understanding the data set

Let???s quickly run some descriptive analysis to better understand the
data set.

``` r
# Data set dimensions
dim(d)

# Distribution across experimental groups
d %>%
  group_by(norm, profile) %>%
  tally %>%
  mutate(prop = n/nrow(d))

# Socio-demographics
describe(d$age)
table(d$gender)
table(d$edu)

# Small recoding
d <- d %>%
  mutate(gender = ifelse(gender == 2, NA, gender),
         edu = edu-1)
```

As we can see, we have 677 participants and 10 variables in the data
set. Participants are fairly equally distributed across all nine
conditions, which include further at least 71 participants.

# Simple Regression Analysis

## Fitting a simple regression with `brms`

To run a (first) Bayesian simple regression analysis, we basically do
the exact same thing as if we would use `lm()` to fit a Frequentist
simple regression analysis. We only substitute `lm()` with `brm()`. The
actually fitting process will take a bit longer that you might be used
to as MCMC chains have to be simulated.

Likewise, we can use the summary function to produce a first informative
overview of the results. To keep it simple, we are going to investigate
the relationship between two of the post-manipulation measures: Norm
perceptions (perceptions) and intention to disclose oneself (behavior).
The assumption is that the more people perceive the majority to disclose
themselves, the more likely they are to disclose themselves.

``` r
m1 <- brm(behavior ~ perceptions, data = d)
summary(m1)
```

As we can see, the resulting summary even looks similar to what we would
expect from a Frequentist analysis. Yet, it is important to understand
that the table under *Population-Level Effects* are not point estimates,
but summaries of posterior distributions. The ???estimate??? is in fact the
median of this distribution and the ???l-95% CI??? and the ???u-95% CI??? are
the 95% highest density intervals (HDI). With the *Rhat* values, we also
get a first idea about whether the MCMC chains converged (remember:
values close to 1 suggest convergence).

Before we dive deeper into how we can adjust and change the defaults
settings, let???s quickly inspect the posterior distribution of our main
effect visually. First, let???s extract the posterior draws from the
`brmsfit`-object. We can do so in various ways, but the `brms`-default
is using the function `as_draws_df()`.

``` r
(posterior <- as_draws_df(m1))
```

As we can see, we get a data set with 4000 rows. The columns represent
the different parameter that were estimated (Intercept, b, sigma). We
further gain information about the chosen (default) prior and learn that
4 chains were created. So we can deduce that each chains contains 1000
draws. These are the `brms` defaults - we will learn later how to adjust
these to fit our needs. For now it is sufficient to now that we
basically have 4 posterior draws per parameter with 1000 values
(combined 4000).

We can now use this data set and visualize the posterior distribution
for our effect of interest. The simplest way would be to simple plot a
histogram. An alternative is to plot these draws as regression lines in
a standard scatter plot. Here, we better sample from our posterior as
4000 lines would hardly fit on the scatterplot.

``` r
# Histogram
posterior %>%
  ggplot(aes(x = b_perceptions)) +
  geom_histogram(color = "white", fill = "steelblue") +
  theme_classic() +
  labs(x = "Effect of norm perceptions on intention to disclose")

# Regression lines
posterior %>%
  sample_n(size = 200) %>%
  ggplot() +
  geom_point(data = d,
             aes(x = perceptions, y = behavior),
             size = 1, alpha = .5) +
  geom_abline(aes(intercept = b_Intercept, 
                  slope = b_perceptions, 
                  group = .draw),
              color = "grey50", size = 1/4, alpha = .1) +
  theme_classic()
```

Although you can technically run any analysis like this, the real
benefit of using Bayesian analysis comes with adjusting priors,
investigating the convergence of the MCMC draws in more detail and using
the posterior in different, more tailored-to-the-hypothesis/interest
type of way.

## Setting priors

In a first step, we may want to make the priors more informative as we
probably have some subjective belief about our effect of interest that
could be modeled as a distribution and included in the model. But what
priors do we actually have to specify in this simple example? The
`get_prior()` function can help here:

``` r
get_prior(formula = behavior ~ perceptions, data = d)
```

When you run `get_prior()`, a table is returned with a few columns. Each
row corresponds to one model parameter. For now, we focus on the first
three (prior, class, coef).

-   *prior*: tells you which prior probability distributions are set as
    default by brms. For our model, the first two default priors are
    (flat), i.e.??uniform distributions (all values are equally
    probable). The other two priors are Student-t distributions. This is
    what we will want to change to something more appropriate.

-   *class:* indicates the type of model parameter: Intercept is the
    model intercept, b indicates a predictor term, and sigma the model???s
    residual standard deviation.

-   *coef:* indicates to which model predictor the prior is assigned. In
    our model above, there???s only one predictor (perceptions), and
    that???s what you can see in the output.

Now, we want to change these priors to more appropriate prior
assumptions. For our model, we need three priors: for the Intercept, the
effect (b) and for sigma. We can simply store them as vector (`c()`) and
define them with the function `prior()`. Within the function, we need to
specify the prior in ???Stan??? language and also specify to which class the
prior belongs to. We probably have a more intuitive feeling for
standardized effect sizes, so we specifiy the priors in terms of Cohen???s
*r*. Yet, we need to bear in mind that we then also have to standardize
our coefficients before we refit the model.

``` r
# Quick visual check for my prior assumption about the effect
x <- seq(-1, 1, by = .001)
y <- dnorm(x, .2, 1)
ggplot(NULL, aes(x, y)) +
  geom_line()

# Quick visual check for the standard deviation/varince (= sigma)
library(extraDistr)
x <- seq(0, 10, by = 0.1)
y <- dhcauchy(x, sigma = 1)
ggplot(NULL, aes(x, y)) +
  geom_line()

# Specifiying priors
my_priors <- c(
  prior(normal(0, 10), class = "Intercept"),
  prior(normal(0.2, 1), class = "b"),
  prior(cauchy(0, 1), class = "sigma")
)
```

Now, we can use this vector and add it to the model.

``` r
# Standardization to align with priors
d <- d %>%
  mutate(perceptions_std = scale(perceptions),
         behavior_std = scale(behavior))

# Model fitting
m2 <- brm(formula = behavior_std ~ perceptions_std, 
          prior = my_priors,
          chains = 2,        # only to reduce computation time, should be 4 at least
          iter = 12000,      # increase number of draws for more precise estimation
          warmup = 2000,     # set warmup
          sample_prior = T,  # This ensure that we can also draw from the prior (helps with visualization)
          data = d)

# Results
summary(m2)
```

The median effect size of the relationship between perceptions and
behaviors is b = .55 (95% HDI\[.49, .62\]). Did anyone use different
priors? What are your results?

Let???s quickly check how the data updated our prior assumption.

``` r
# Get prior and posterior samples
posteriors <- get_parameters(m2) %>% as_tibble %>%
    mutate(type = "posterior")
priors <- prior_draws(m2) %>% as_tibble %>% 
    magrittr::set_colnames(names(posteriors)) %>%
    mutate(type = "prior")

# Plot
bind_rows(posteriors, priors)%>%
  ggplot(aes(x = b_perceptions_std, fill = type)) +
  geom_density(alpha = .8) +
  theme_classic() +
  xlim(-1, 1) +
  labs(x = expr(theta), y = "", fill = "") +
  scale_fill_manual(values = c("steelblue", "grey"))
```

# Convergence diagnostics

Before we turn to various approaches to Bayesian inference, we need to
check more carefully whether our MCMC chains actually converged. We do
so visually as well as by inspecting some statistics.

## Trace plots

First, we investigate so-called trace plots, which show the random walk
of the algorithm and check whether the chains overlap sufficiently. It
should look like a somewhat ???smooth caterpillar???.

``` r
draws <- as_draws_df(m2)
mcmc_trace(draws, 
           pars = c("b_Intercept", "b_perceptions_std", "sigma"), 
           n_warmup = 0,
           facet_args = list(nrow = 2))
```

Based on these plots, we are not concerned that the chains did not
converge. This is how these trace plots should look like.

## Rhat

Next, we extract the Rhat values, a parameter for the variance between
the chains in relation to the variance within the chain. It should be
close to 1 to suggest convergence.

``` r
rhat(m2) %>%
  as_tibble(rownames = "par")
```

Again, no problem at all.

## Autocorrelation

Finally, we check whether there is strong auto-correlation in the
iterations of the chains. It looks like there is a bit of
auto-correlation in the first chain. We can refit the model using
thinning (that is using only every 4th draw) and check whether this
improves the result.

``` r
mcmc_acf(draws, 
         pars = c("b_Intercept", "b_perceptions_std", "sigma"),
         lags = 20)

# Refit with thinning
m2b <- brm(formula = behavior_std ~ perceptions_std, 
          prior = my_priors,
          chains = 2,       
          iter = 12000,      
          warmup = 2000,     
          thin = 4,
          data = d)

drawsb <- as_draws_df(m2b)
mcmc_acf(drawsb, 
         pars = c("b_Intercept", "b_perceptions_std", "sigma"),
         lags = 20)
```

Now the auto-correlation plot look more like they should.

# Bayesian Inference

## Highest Density Interval (HDI)

The highest density intervals (sometimes also HCR = Highest density
region or HCI = Highest credibility interval) indicates which points of
a distribution are most credible/probable. We can simply compute
different HDIs with the function `hdi()` from the `bayestestR` package.
We can also simply plot the output.

``` r
# Estimating hdis
(hdis <- hdi(draws$b_perceptions_std, ci = c(.90, .95, .99)))

# Plotting hdis
plot(hdis)
```

Based on the HDIs we can do simple Null-Hypothesis-Testing, by checking
wether the NULL (whether exactly zero or any other value) is included in
the HDI.

## Region of Practical Equivalence (ROPE)

Testing whether something is different than zero is often not very
meaningful. We can hence also test whether the HDI lies outside a
certain region that we assume to be equal to our NULL (e.g., a boundary
around zero). The interesting aspect of this, that we can use a ROPE in
different ways:

1)  testing whether the effect is likely to be outside of a certain
    range
2)  testing whether the effect is likely to be inside of a certain range
    (aking to equivalence testing)

``` r
# Computing percentage within rope for a potential null = 0 region
rope(m2, range = c(-.1,.1), ci = c(.9, .95))

# Computing percentage within rope for a potential null = large effect region
rope(m2, range = c(.4,.6), ci = c(.9, .95))
```

Of course, we can also plot a ROPE testing approach.

``` r
a <- plot(rope(m2, range = c(-.1, .1), ci = .95)) +
  geom_vline(xintercept = 0, linetype = "dashed") +
  xlim(-.2, .8) +
  labs(caption = "Note: ROPE [-.10, .10]") +
  theme_minimal() +
  theme(legend.position = "none")

b <- plot(rope(m2, range = c(.4, .6), ci = .95)) +
  geom_vline(xintercept = 0, linetype = "dashed") +
  xlim(-.2, .8) +
  labs(caption = "Note: ROPE [.40, .60]", title = "") +
  theme_minimal()

cowplot::plot_grid(a, b, axis = "l", align = "v", rel_widths = c(1.5,2))
```

## Probability of direction

As mentioned in the lecture, we can also simply compute the probability
of the effect being in a certain direction.

``` r
dir <- p_direction(draws$b_perceptions_std, null = 0)
plot(dir)

# Just to exemplify a case that is less certain
plot(p_direction(rnorm(1000, 0.1, .2)))
```

## Posterior Predictive Check

As a final step, we can simulate replicated data under the fitted model
and then compare these to the observed data. These so-called posterior
predictive checks help to identify systematic discrepancies between real
and simulated data.

``` r
pp_check(m2, ndraws = 100) + theme_minimal()
```

As we can see here, the fitted model does not yet replicate the data
well. The reason is somewhat straight-forward: We are looking at
post-experimental measures who have been affected by the manipulations.
So let???s dive deeper into the data set!

# Exercise

**Let???s practice what we just learned!**

In the example above, we have investigated the relationship between two
of the post-experiment measures. Of course, it is way more interesting
to test the actual effect of the manipulations. Let us create a full
regression model in which we include both manipulations as independent
variables (perhaps even including their interaction) as well as all
control variables that we may find useful to include. Similar to
Frequentist linear regression, we can of course use a stepwise approach
and start with simpler models first and then gradually add predictors.
The function `bayes_R2()` allows to compute a Bayesian equivalent of
![R^2](https://latex.codecogs.com/png.image?%5Cdpi%7B110%7D&space;%5Cbg_white&space;R%5E2 "R^2")
and could thus be used to compare explanatory power. So in sum, engage
with the following steps:

1.  Build meaningful models to predict the intention to disclose
2.  Specify meaningful priors if possible (if you leave out those where
    you have now assumptions, the model will use a weakly informative
    default).
3.  Extract posterior for the relevant effects:
    -   Effect of the norm manipulation on intention to disclose
    -   Effect of the profile manipulation on the intention to disclose
    -   Interaction between both?
4.  Test whether these effects lie outside of a meaningful ROPE (e.g.,
    -.1 to .1 = larger than small effects)

``` r
# Code here
```
