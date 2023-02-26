---
title: A Toy Example of Using Rstan
author: Xinran Miao
layout: post
active: blog
permalink: /blog/20230226stan/
---

This post prepares a toy example for estimating probalistic models using Rstan.

### Motivating Example
Suppose we observe data 
$$
Y \in\mathbb{R}^{n\times d}$$
and $$
X\in \mathbb{R}^{n\times p}
$$.


We believe the underlying model is
$$
\begin{align}
Y\sim N(X\beta, \Sigma)
\end{align}
$$,
where $$\beta\in\mathbb{R}^{p\times d}$$ and $$\Sigma$$ is a diagonal matrix.

How shall we estimate these parameters?

### Why Rstan?
As I quote from the [Stan documentation](https://mc-stan.org), Stan is a probabilistic programming language that provides high-performance computation (e.g. Bayesian inference) for problems like the above. It interfaces with many languages, and we will focus on `R` in this post. In specific, we use `CmdStanR`.

### Installation
Install `cmdstanr` following this [documentation](https://mc-stan.org/cmdstanr/).


### Example
Fitting a model using `cmdstanr` typically takes three steps:
* Prepare an input data list.
* Prepare a stan model.
* Fit the model with the input data.


Code shown below can also be found on [Github](https://github.com/XinranMiao/stan_example/tree/master/minimal).

#### Data Preparation

{% highlight r %}
# minimal.rmd
# library(cmdstanr); library(dplyr)

# Synthetic data generation
n <- 100
p <- 4 # dimension of x
d <- 2 # dimension of y

x <- matrix(rnorm(n * p), nrow = n)
beta0 <- matrix(c(1, -1, 1, 0, -3, 1, 0, 0), nrow = p)
y <- matrix(x %*% beta0, nrow = n) + matrix(rnorm(n * d, sd = .1), nrow = n)

# `data_list` will be your input data
data_list <- list(n = n, p = p, d = d,
                  x = x, y = y)
{% endhighlight %}





#### Stan code
{% highlight stan %}
// normal.stan
data {
  int<lower=1> n;
  int<lower=1> p;
  int<lower=1> d;
  matrix[n, d] y;
  matrix[n, p] x;
}

parameters {
  matrix[p, d] beta;
  vector<lower=0>[d] sigmas;
}

model {
  // likelihood
  for (i in 1:n) {
    y[i] ~ multi_normal(beta' * to_vector(x[i]), diag_matrix(sigmas));
  }
}

{% endhighlight %}

#### Model Compilation

{% highlight r%}
# minimal.rmd, continued
model <- cmdstan_model("normal.stan")
{% endhighlight %}


#### Model Fitting
The `$model` method provides MCMC sampling. After sampling, one may obtain draws by calling `$draws()` from the fitted object.
{% highlight r%}
# minimal.rmd, continued
## MCMC
mcmc_fit <- model$sample(data_list)
## Posterior draws
mcmc_fit$draws("beta[1,1]")

## A draws_array: 1000 iterations, 4 chains, and 1 variables
#, , variable = beta[1,1]

#         chain
#iteration    1    2    3    4
#        1 1.01 0.99 1.00 0.98
#        2 0.99 0.99 1.00 1.01
#        3 0.99 1.00 0.97 0.98
#        4 0.99 0.98 0.98 1.00
#        5 0.99 1.00 1.01 0.99

## ... with 995 more iterations
{% endhighlight %}


You can also obtain the sampling summary by
{% highlight r%}
# minimal.rmd, continued
mcmc_fit$summary()
{% endhighlight %}






Alternative to MCMC, one may consider a penalized optimization or variational inference.
{% highlight r%}
# minimal.rmd, continued
## Point estimate
opt_fit <- model$optimize(data_list)
opt_fit$summary()
{% endhighlight %}



{% highlight r%}
# minimal.rmd, continued
## Variational inference
var_fit <- model$variational(data_list)
var_fit$summary()
var_fit$draws("sigmas")
{% endhighlight %}


### Resources:

[How does CmdStanR work?](https://mc-stan.org/cmdstanr/articles/cmdstanr-internals.html)

[Stan User's Guide (version 2.31)](https://mc-stan.org/docs/stan-users-guide/index.html)
