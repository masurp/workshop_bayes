Intro to Bayesian Statistic: How to install brms and dependencies
================
Philipp Masur
2022-09

-   <a href="#introduction" id="toc-introduction">Introduction</a>
-   <a href="#macos" id="toc-macos">macOS</a>
    -   <a href="#configure-c-toolchain"
        id="toc-configure-c-toolchain">Configure C++ toolchain</a>
    -   <a href="#install-rstan" id="toc-install-rstan">Install Rstan</a>
-   <a href="#windows" id="toc-windows">Windows</a>
    -   <a href="#configure-the-c-toolchain"
        id="toc-configure-the-c-toolchain">Configure the C++ toolchain</a>
    -   <a href="#install-rstan-1" id="toc-install-rstan-1">Install Rstan</a>
-   <a href="#install-brms" id="toc-install-brms">Install brms</a>
-   <a href="#install-further-packages"
    id="toc-install-further-packages">Install further packages</a>
-   <a href="#a-simple-example" id="toc-a-simple-example">A simple
    example</a>

# Introduction

In this tutorial, we are going to install `brms` and its dependencies.
When you fit a model with brms, the package actually calls `Rstan` in
the background, which, in turn, is an R interface to the statistical
programming language `Stan`.

Stan is built in the programming language C++ and models have to be
compiled using C++ to be run. This is all taken care of by brms, so you
just need to run brm(…) to fit any model.

`brms` uses a syntax for specifying model formulae that is based on the
syntax of the commonly known `lme4` package (for multilevel modeling).
The comparatively easy syntax of brms is then converted into Stan code
automatically.

That said, since the models have to be compiled in C++, you need to set
up your computer so that it can actually use C++. This has to be done
only once, before installing `brms`. Unfortunately, as almost always,
how you install C++ depends on your operating system. In the following,
we will go through the process for MAC and WINDOWS.

# macOS

## Configure C++ toolchain

To configure the C++ toolchain on macOS, you just need to install the
Xcode Command Line Tools and/or `gfortran`.

To install the Xcode Command Line Tools, you can run the following in
the Terminal (open Finder \> Applications \> Terminal, type in the
following command and press ENTER):

``` r
xcode-select --install
```

Alternatively you can also install Xcode from the Apple Store using this
link: <https://apps.apple.com/us/app/xcode/id497799835?mt=12>

macOS will download and install the Xcode CLT (it will take a while,
make sure you have a stable internet connection).

If your Mac has an Intel chip, you need to install gfortran v8.2 (for
Mojave) independent of your macOS version. You can download the
installer from
<https://github.com/fxcoudert/gfortran-for-macOS/releases/tag/8.2>.

If your Mac has an Apple M1 chip, you will need to install gfortran v11.
You can download it from here:
<https://github.com/fxcoudert/gfortran-for-macOS/releases/tag/11-arm-alpha2>
(download the .pkg package listed under Assets at the bottom of the
page).

## Install Rstan

Just in case you previously tried to install Rstan without success, run
the following code to clean installations and configuration files.

``` r
remove.packages("rstan")
if (file.exists(".RData")) file.remove(".RData")
```

Restart R.

Assuming you are using R 4.0 or later, just run in the R console:

``` r
install.packages("rstan")
```

Now, verify the installation by running:

``` r
example(stan_model, package = "rstan", run.dontrun = TRUE)
```

If the model compiles and starts sampling, you are set.

# Windows

## Configure the C++ toolchain

To configure the C++ toolchain, follow the instructions here for your R
version (3.6 or 4.0):
<https://github.com/stan-dev/rstan/wiki/Configuring-C---Toolchain-for-Windows>.

## Install Rstan

Just in case you previously tried to install Rstan without success, run
the following code to clean installations and configuration files.

``` r
remove.packages("rstan")
if (file.exists(".RData")) file.remove(".RData")
```

Restart R.

Then just run:

``` r
install.packages("rstan", repos = "https://cloud.r-project.org/", dependencies = TRUE)
```

Now, verify the installation by running:

``` r
example(stan_model, package = "rstan", run.dontrun = TRUE)
```

If the model compiles and starts sampling like in the gif below, you are
set.

# Install brms

Next, you can install the actual package `brms` by using the standard
package installation procedure.

``` r
install.packages("brms")
```

You should now be able to use `brms` to estimate Bayesian regression
models.

# Install further packages

As we are going to use some additional packages, please also install
these now:

``` r
install.packages(c("bayestestR", "tidybayes", "bayesplot", "insight"))
```

# A simple example

``` r
library(tidyverse)
library(rstan)
library(brms)
library(bayestestR)
library(insight)

# Fit model
model <- brm(Sepal.Length ~ Petal.Length, data = iris)

# Results
summary(model)

# Plot posterior distribution
posteriors <- get_parameters(model)
qplot(posteriors$b_Petal.Length)
```
