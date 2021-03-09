# The sequential.causal package

#### `sequential.causal` is an R package for sequential inference of the average treatment effect (ATE).

To download the package, run the following

``` r
devtools::install_github('wannabesmith/sequential.causal')
```

``` r
library(sequential.causal)
library(parallel)
library(pracma)
```

Let’s jump into a simple example based on the paper “Doubly-robust
confidence sequences for sequential causal inference” by Waudby-Smith et
al. (2021). Suppose that we wish to estimate the ATE in an observational
setting with no unmeasured confounding.

First, we will generate *n* = 10<sup>4</sup> observations each with 3
real-valued covariates from a trivariate Gaussian. That is,

``` r
n = 10000
d = 3
X <- cbind(1, matrix(rnorm(n*d), nrow = n))
```

Randomly assign subjects to treatment or control groups with equal
probability: Define the regression function, and the target parameter
(which we will ensure is the average treatment effect by design),

*ψ* := 1.

Finally, generate outcomes *Y*<sub>1</sub>, …, *Y*<sub>*n*</sub> as

*Y*<sub>*i*</sub> := *f*<sup>⋆</sup>(*x*<sub>*i*, 1</sub>, *x*<sub>*i*, 2</sub>, *x*<sub>*i*, 3</sub>) + *ψ* ⋅ *A*<sub>*i*</sub> + *ϵ*<sub>*i*</sub>,

where *ϵ*<sub>*i*</sub> ∼ *t*<sub>5</sub> are drawn from a
*t*-distribution with 5 degrees of freedom.

``` r
ATE <- 1

beta_mu <- c(1, -1, -2, 3)

reg_true <- function(x)
{
  beta_mu %*% c(1, x[1]^2, sin(x[2]), abs(x[3]))
}

prop_score_true <- function(x)
{
  logodds <- reg_true(x)
  pi = exp(logodds)/(1 + exp(logodds))
  # Ensure bounded away from 0 and 1
  pi = pi*0.6 + 0.2
}

reg_observed <- apply(X, MARGIN=1, FUN=reg_true)
p <- apply(X, MARGIN=1, FUN=prop_score_true)
treatment <- rbinom(n, 1, p)
y <- reg_observed + treatment*ATE + rt(n, df=5)
```

We will build three estimators for the ATE with increasing degrees of
complexity.

1.  **Unadjusted**: uses the constant function 0 for the outcome
    regression and the fraction of treated subjects for the propensity
    score. This is equivalent to an inverse-probability-weighted
    estimator whose propensity scores are estimated ignoring covariates.
2.  **Parametric**: estimates the outcome regression with a linear model
    and the propensity score with logistic regression.
3.  **Super Learner**: estimates the outcome regression and propensity
    score with stacked regressors/classifiers.

We expect the Super Learner to be the only consistent and most efficient
estimator of the three, since the regression function and propensity
scores are misspecified for the Unadjusted and Parametric models. In
this case, the Super Learner we use is implicitly made up of regression
splines, GAMs, *k*-NN, GLMs with interactions and regularization, and
random forests (see `?get_SL_fn` for more details and to customize this
list). Under the hood, these flexible machine learning methods are
combined using the fantastic `SuperLearner` package.

``` r
# Get SuperLearner prediction function for $\mu^1$.
# Using default ML algorithm choices
sl_reg_1 <- get_SL_fn()

# Do the same for $\mu^0$.
sl_reg_0 <- get_SL_fn()

# Get SuperLearner prediction function for $\pi$
pi_fn <- get_SL_fn(family = binomial())

# Get GLM regression superlearner
glm_reg_1 = get_SL_fn(SL.library = "SL.glm")

# Get GLM propensity score superlearner
glm_prop <- get_SL_fn(SL.library="SL.glm", family=binomial())

# Compute the confidence sequence at logarithmically-spaced time points
times <- unique(round(logseq(250, 1000, n = 10)))
alpha <- 0.05
# Set n_cores to 1 if you do not want to do parallel processing
n_cores <- detectCores()

# Split the sample (if we don't do this explicitly,
# confseq_ate can automatically)
train_idx <- rbinom(n, p = 0.5, size = 1) == 1

confseq_SL <- confseq_ate(y, X, treatment, regression_fn_1 = sl_reg_1,
                          regression_fn_0 = sl_reg_1,
                          propensity_score_fn = pi_fn,
                          train_idx = train_idx, t_opt = 500, alpha=alpha,
                          times=times, n_cores = n_cores, cross_fit = TRUE)
confseq_glm <- confseq_ate(y, X, treatment, regression_fn_1 = glm_reg_1,
                           regression_fn_0 = glm_reg_1,
                           propensity_score_fn = glm_prop,
                           train_idx = train_idx, t_opt = 500, alpha=alpha,
                           times=times, n_cores = n_cores, cross_fit = TRUE)

confseq_unadj <- confseq_ate_unadjusted(y = y, treatment = treatment,
                                        propensity_score = mean(treatment),
                                        t_opt = 500, alpha = alpha,
                                        times = times)
```

![](examples/Guide_to_sequential_causal_files/figure-markdown_github/unnamed-chunk-6-1.png)
