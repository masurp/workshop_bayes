Intro to Bayesian Statistic: Multilevel Regression Analysis
================
Philipp Masur
2022-09

-   <a href="#introduction" id="toc-introduction">Introduction</a>
-   <a href="#preparation" id="toc-preparation">Preparation</a>
    -   <a href="#loading-packages" id="toc-loading-packages">Loading
        packages</a>
    -   <a href="#getting-data" id="toc-getting-data">Getting data</a>
    -   <a href="#lets-explore-some-of-the-variables"
        id="toc-lets-explore-some-of-the-variables">Let’s explore some of the
        variables</a>
-   <a href="#multilevel-regression-analysis"
    id="toc-multilevel-regression-analysis">Multilevel Regression
    Analysis</a>
    -   <a href="#fitting-a-multilevel-model-with-brms"
        id="toc-fitting-a-multilevel-model-with-brms">Fitting a multilevel model
        with <code>brms</code></a>
    -   <a href="#posterior-distributions"
        id="toc-posterior-distributions">Posterior distributions</a>
    -   <a href="#setting-meaningful-priors"
        id="toc-setting-meaningful-priors">Setting meaningful priors</a>
    -   <a href="#rerunning-the-analyses"
        id="toc-rerunning-the-analyses">Rerunning the analyses</a>
    -   <a href="#convergence-checks" id="toc-convergence-checks">Convergence
        checks</a>
    -   <a href="#hypothesis-tests" id="toc-hypothesis-tests">Hypothesis
        tests</a>
-   <a href="#other-fun-stuff" id="toc-other-fun-stuff">Other fun stuff</a>

# Introduction

This tutorial introduces the process of fitting multlevel models as
implemented in [`brms`](https://paul-buerkner.github.io/brms/). The
formula syntax is very similar to that of the package `lme4` to provide
a familiar and simple interface for performing multilevel regression
analyses. So if you are familiar with fitting multilevel models in
`lme4`, using `brms` will be a no-brainer.

# Preparation

## Loading packages

For this tutorial, we will again load a number of different packages,
with `brms` at the core.

``` r
library(tidyverse)
library(brms)
library(insight)
library(bayestestR)
library(bayesplot)
library(psych)
```

## Getting data

For this tutorial, we will use a data set on pupils as provided by the
UCLA stats department. We can load it directly by using the function
`read_dta()` from the package `haven`

``` r
library(haven)
d <- read_dta("https://stats.idre.ucla.edu/stat/examples/imm/imm10.dta")
head(d)
```

## Let’s explore some of the variables

To get a better understanding of the data set, let’s run some
descriptive analyses and visualizations of key variables.

``` r
# Schools
d %>% 
  group_by(schid) %>%
  tally

# Public or private?
d %>% group_by(schid) %>%
  select(public) %>%
  distinct

# Socio-Demographics
table(d$sex)
```

We have 10 different schools with student numbers ranging from 20 to 67.

``` r
d %>%
  select(homework, math, ses) %>%
  describe
```

We have three variables that could be interesting to investigate, as
well as a number of moderators that we could further include. Let us for
now, look at the relationship between the frequency of doing *homework*
and performing in a *math* test. To get a better understanding of the
variables, we plot their relationships across schools. We will further
rescale the outcome variable to range only from 1 to 10.

``` r
# Overall(no pooling)
d %>%
  ggplot(aes(x = homework, y = math)) +
  geom_point(alpha = .3)+ 
  geom_smooth(method = "lm") +
  theme_minimal()

# Per school (complete pooling)
d %>%
  ggplot(aes(x = homework, y = math)) +
  geom_point(alpha = .3)+ 
  geom_smooth(method = "lm") +
  facet_wrap(~schid) +
  theme_minimal()

# Rescaling
d <- d %>%
  mutate(math = math/10)
```

# Multilevel Regression Analysis

In a multilevel regression framework, we not only specify the typical
regression formula (e.g., `y ~ x`), we also need to specify so-called
random effects, i.e., which of the effects (including the intercept) are
allowed to vary across the grouping variables (in this case: schools).

## Fitting a multilevel model with `brms`

If you are familiar with the syntax of `lme4`, specifying a multilevel
model in `brms` is simple. Next to the traditional part of the formula,
we can specify random effects like so:

-   Random intercept only: `+ (1|group)`
-   Random intercept, random slope: `+ (1 + x|group)`

In our example, we are going to estimate a random intercept and slope
model in which both intercept and the effect of homework on math are
allowed to vary across schools. Let’s also estimate the complementary
Frequentist model with `lme4` to investigate differences.

``` r
# Fit Bayesian model
m1 <- brm(math ~ homework + (1 + homework|schid), data = d)

# Fit frequentist model
m1_freq <- lme4::lmer(math ~ homework + (1 + homework|schid), data = d)

# Investigate results
summary(m1)
summary(m1_freq)
```

As we can see, both models come to similar conclusion with regard to the
relationship between homework and math performance. The Frequentist
model suggest that the effect is *not* significant. The Bayesian
posterior distribution suggest that the most probable value is positive,
but the 95% HDI includes zero. We may hence likewise conclude that the
relationship is not meaningful. Yet, we also see that there is
considerable variance in the effect across schools (something we will
come back to later).

## Posterior distributions

Let’s quickly have a look at the types of posterior distributions that
we can extract from the model. We get considerably more posteriors from
a Bayesian Multilevel model compared to a Bayesian linear regression
model. For example, we can also visualize the posterior distribution for
the variance around the main effect across school. It is here were a
particular strength of Bayesian analysis lies: We get posterior
distribution for various parameters of our model and can thus also
formulate and test hypotheses related to them!

``` r
# Inspection of the posterior distributions
param <- as_draws_df(m1) %>%
  as_tibble
names(param)
head(param)

# Quick visual inspection of the main effect posterior with HDIs
param$b_homework %>%
  hdi( ci = c(.9, .95, .99)) %>%
  plot +
  scale_fill_brewer(palette = "Blues") +
  theme_minimal() +
  labs(title = "Effect of homework on math")

# Quick visual inspection of the variance of the main effect across schools
param$sd_schid__homework^2 %>%  
  hdi( ci = c(.9, .95, .99)) %>%
  plot +
  scale_fill_brewer(palette = "Blues") +
  theme_minimal() +
  labs(title = "Variance of the effect")
```

## Setting meaningful priors

Similar to our earlier tutorial, let us think about adjusting the
default priors. Again, we can use the function `get_prior` to better
understand what priors we can specify.

``` r
get_prior(formula = math ~ homework + (1 + homework |schid), data = d)
```

In the multilevel framework, we have several different types of
parameters for which we can set prior assumptions. First and foremost
(and similar to the standard regression analysis), we have to specify
our assumptions for the fixed effects (i.e., for the intercept and the
relationship between our variables of interest). Next, we can also set
priors for the random effects (i.e., both for random intercepts and
random slopes. ). And finally, we can also set priors for the
correlation between random intercepts and slopes and for sigma.

Bear in mind: These are the possible priors for a simple bivariate
model. It becomes gradually more complex with more predictors,
interactions, more random effect structures (e.g., cross-classified
models, hierarchically nested models…).

``` r
priors_complex <- c(
  # fixed effects
  prior(normal(...), class = Intercept),
  prior(normal(...), class = b, coef = homework),
  # random effects 
  prior(cauchy(...), class = sd, coef = Intercept, group = schid),
  prior(cauchy(...), class = sd, coef = homework, group = schid),
  # correlation between random intercept (and if necessary slope)
  prior(lkj(...), class = cor, group = schid),
  # sigma
  prior(cauchy(...), class = sigma)
)
```

## Rerunning the analyses

Instead of specify all priors, we can also just specify those for which
we have prior assumptions and leave the remainig ones uninformative
flat. For this example, I will only specify the fixed effects.

``` r
# Intercept
x <- seq(-10, 10, by = .01)
y <- dnorm(x, 5, 5)
ggplot(NULL, aes(x, y)) + geom_line() + xlim(0, 10)

# Effect
x <- seq(-10, 10, by = .01)
y <- dnorm(x, 0, 2)
ggplot(NULL, aes(x, y)) + geom_line() + xlim(-5, 5)

# Set priors
priors_simple <- c(
  prior(normal(5, 10), class = Intercept),
  prior(normal(0, 2), class = b, coef = homework)
)
```

Then, we refit our model. I also change some other parameters to adjust
the results to my liking.

``` r
# Fit model
m2 <- brm(math ~ homework + (1 + homework|schid), 
          prior = priors_simple,
          sample_prior = T,  # to include prior draws
          warmup = 1500,
          iter = 6500,       # a bit better estimation
          chains = 2, 
          cores = 2,
          data = d, 
          seed = 42)         # for replicability

# Investigate results
summary(m2)

# Quick comparison between models
hdi(get_parameters(m1))
hdi(get_parameters(m2)) # not really better, but also not worse ;)
```

Similar to our other tutorial, we can extract posterior samples and plot
the difference between prior and posterior distributions.

``` r
# Get prior and posterior samples
posteriors <- get_parameters(m2) %>% 
  as_tibble %>%
  mutate(type = "posterior")
priors <- prior_draws(m2) %>% 
  as_tibble %>% 
  select(-sd_schid, cor_schid) %>%
  mutate(type = "prior")

# Plot
bind_rows(posteriors, priors)%>%
  ggplot(aes(x = b_homework, fill = type)) +
  geom_density(alpha = .8) +
  theme_classic() +
  labs(x = expr(theta), y = "", fill = "") +
  scale_fill_manual(values = c("steelblue", "grey"))
```

It is pretty obvious that the data updated our rather vague prior belief
considerably.

## Convergence checks

**Exercise 1:** Check whether the MCMC chains converge. Use both
statistical parameters and visualizations!

``` r
# Code here
```

## Hypothesis tests

**Exercise 2:** Test whether the following hypothesis holds given the
estimated model (m2). Also use the available tools to visualize this
inference approach.

> Students will perform 2.5% better on the math test, when they do their
> homework more often (i.e., exactly 1 one more on the frequency scale).
> We consider increases between 2% and 3% to be equivalent to such an
> effect.

``` r
# Code here
```

# Other fun stuff

Because we get various posteriors for different parameter in our model.
We can investigate our model in much more detail. For example, we can
extract posterior distributions for the random effects (i.e., deviations
from the fixed effects per school). We can use these to plot posterior
histrograms for each school and evaluate whether any of the school
aligns with our hypothesis above.

``` r
# Extrating parameters
param <- as_draws_df(m2) %>%
  as_tibble

# Posterior distribution per school
param %>%
  select(b_homework, contains("homework]")) %>%
  sample_n(size = 200) %>%
  gather(key, value, -b_homework) %>%
  separate(key, c("school", "parameter"), sep = ",") %>%
  mutate(value = b_homework + value) %>%
  ggplot(aes(x = value)) +
  geom_rect(aes(xmin=.2, xmax=.3, ymin=0, ymax=Inf), fill = "grey", alpha = .4) +
  geom_histogram(fill = "steelblue", color = "white") +
  geom_vline(xintercept = 0, linetype = "dashed") +
  facet_wrap(~school, nrow = 5) +
  theme_bw() +
  labs(x = "Possible parameter values", )
```

Of course we can also plot the probable regression lines per school.
With a bit of data wrangling we get an interesting plot that shows quite
vivily how different the schools are with regard to the relationship
between homework and math.

``` r
# Sample of regression lines per school
param %>%
  select(b_Intercept, b_homework, contains("r_schid[")) %>%
  sample_n(size = 200) %>%
  gather(key, value, -b_homework, -b_Intercept) %>%
  separate(key, c("school", "parameter"), sep = ",") %>%
  group_by(school, parameter) %>%
  mutate(id = 1:n()) %>%
  spread(parameter, value) %>%
  mutate(intercept = b_Intercept + `Intercept]`,
         homework = b_homework + `homework]`,
         fixef_int = median(intercept),
         fixef_hom = median(homework)) %>%
  ggplot() +
  geom_abline(aes(intercept = intercept, 
                  slope = homework),
              color = "grey50", size = 1/4, alpha = .2) +
  geom_abline(aes(intercept = fixef_int, 
                  slope = fixef_hom),
              color = "steelBlue", size = 1) +
  facet_wrap(~school, nrow = 2) +
  xlim(0, 7) +
  ylim(0, 10) +
  theme_bw()
```

**Exercise 3:** To practice fitting multilevel models with `brms`, try
to recreate the following example in a Bayesian Framework:
<https://osf.io/czfv5>. The study represents an experience samping
design. Participants were prompted several times a day, right after they
used their smartphone. The short questionnaire asked about their trust
towards the respective conversation partner (e.g., during a WhatsApp
chat) and how much they disclosed themselves.

Think about ways to test the main hypotheses and how to visualize the
analyses and results. Below, I provided some first code to load the data
and how to merge them.

``` r
d.situ <- read.csv2("https://osf.io/m586g/download", header = TRUE, na.strings = "NA") 
d.pers <- read.csv2("https://osf.io/mdjrc/download", header = TRUE, na.strings = "NA")

d_long <- left_join(d.situ, d.pers) %>%    
  na.omit %>%                              
  select(id, age, sex, time, it, sd) %>% 
  as.tibble

head(d_long)

# Code here
```
