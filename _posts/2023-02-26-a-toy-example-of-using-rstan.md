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
## A tibble: 11 × 10
#    variable        mean    median      sd     mad       q5      q95  rhat ess_b…¹ ess_t…²
#    <chr>          <dbl>     <dbl>   <dbl>   <dbl>    <dbl>    <dbl> <dbl>   <dbl>   <dbl>
#  1 lp__      361.         3.61e+2 2.22    2.07     3.57e+2  3.64e+2 1.00    1906.   2371.
#  2 beta[1,1]   1.00       1.00e+0 0.00894 0.00896  9.88e-1  1.02e+0 1.00    7268.   2817.
#  3 beta[2,1]  -0.988     -9.88e-1 0.00927 0.00906 -1.00e+0 -9.73e-1 1.00    7103.   3187.
#  4 beta[3,1]   0.979      9.79e-1 0.0107  0.0106   9.61e-1  9.96e-1 1.00    6679.   2970.
#  5 beta[4,1]  -0.00940   -9.42e-3 0.0103  0.0105  -2.63e-2  7.29e-3 1.00    8235.   2650.
#  6 beta[1,2]  -2.99      -2.99e+0 0.00925 0.00937 -3.00e+0 -2.97e+0 1.00    6604.   2750.
#  7 beta[2,2]   1.00       1.00e+0 0.00992 0.0101   9.85e-1  1.02e+0 1.00    8557.   3242.
#  8 beta[3,2]   0.00171    1.91e-3 0.0109  0.0108  -1.61e-2  2.00e-2 0.999   6404.   3082.
#  9 beta[4,2]   0.000692   6.32e-4 0.0108  0.0109  -1.69e-2  1.84e-2 1.00    9002.   2984.
# 10 sigmas[1]   0.00895    8.80e-3 0.00133 0.00128  7.02e-3  1.13e-2 1.00    6215.   2802.
# 11 sigmas[2]   0.00973    9.59e-3 0.00147 0.00139  7.58e-3  1.24e-2 1.00    7145.   2823.
# # … with abbreviated variable names ¹​ess_bulk, ²​ess_tail
{% endhighlight %}






Alternative to MCMC, one may consider a penalized optimization or variational inference.
{% highlight r%}
# minimal.rmd, continued
## Point estimate
opt_fit <- model$optimize(data_list)
opt_fit$summary()
## A tibble: 11 × 2
#    variable    estimate
#    <chr>          <dbl>
#  1 lp__      376.      
#  2 beta[1,1]   1.00    
#  3 beta[2,1]  -0.988   
#  4 beta[3,1]   0.979   
#  5 beta[4,1]  -0.00943 
#  6 beta[1,2]  -2.99    
#  7 beta[2,2]   1.00    
#  8 beta[3,2]   0.00181 
#  9 beta[4,2]   0.000809
# 10 sigmas[1]   0.00824 
# 11 sigmas[2]   0.00895 
{% endhighlight %}



{% highlight r%}
# minimal.rmd, continued
## Variational inference
var_fit <- model$variational(data_list)
var_fit$summary()
# # A tibble: 12 × 7
#    variable         mean    median      sd     mad        q5        q95
#    <chr>           <dbl>     <dbl>   <dbl>   <dbl>     <dbl>      <dbl>
#  1 lp__        174.      174.      3.68    3.76    168.      179.      
#  2 lp_approx__  -4.94     -4.64    2.23    2.00     -9.17     -1.93    
#  3 beta[1,1]     1.01      1.01    0.00893 0.00901   0.999     1.03    
#  4 beta[2,1]    -0.993    -0.993   0.00923 0.00927  -1.01     -0.978   
#  5 beta[3,1]     0.981     0.981   0.0113  0.0109    0.962     0.999   
#  6 beta[4,1]    -0.0190   -0.0187  0.0116  0.0111   -0.0382    0.000335
#  7 beta[1,2]    -2.98     -2.98    0.0101  0.00981  -2.99     -2.96    
#  8 beta[2,2]     1.00      1.00    0.00980 0.00917   0.984     1.02    
#  9 beta[3,2]     0.00614   0.00611 0.0113  0.0105   -0.0125    0.0243  
# 10 beta[4,2]    -0.00542  -0.00565 0.0114  0.0116   -0.0244    0.0133  
# 11 sigmas[1]     0.00940   0.00934 0.00133 0.00131   0.00737   0.0118  
# 12 sigmas[2]     0.0101    0.00992 0.00178 0.00166   0.00748   0.0133  
var_fit$draws("sigmas")
## A draws_matrix: 1000 iterations, 1 chains, and 2 variables
#      variable
# draw sigmas[1] sigmas[2]
#   1     0.0098    0.0106
#   2     0.0071    0.0133
#   3     0.0098    0.0092
#   4     0.0103    0.0128
#   5     0.0106    0.0097
#   6     0.0099    0.0104
#   7     0.0093    0.0103
#   8     0.0090    0.0069
#   9     0.0091    0.0124
#   10    0.0102    0.0128
# # ... with 990 more draws
{% endhighlight %}


### Resources:

[How does CmdStanR work?](https://mc-stan.org/cmdstanr/articles/cmdstanr-internals.html)

[Stan User's Guide (version 2.31)](https://mc-stan.org/docs/stan-users-guide/index.html)
