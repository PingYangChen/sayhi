---
title: "Generalized Linear Model with Elastic Net Penalty"
categories:
  - R
---

<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default"></script>


**elnglm** is a self-developed package that fits the generalized linear model (GLM) with elastic net penalty and predicts the response based on the fitted GLM.  This package is developed by R codes, and, accelerated using C++ codes through **Rcpp** and **RcppArmadillo** packages.  After the manual section, a report showing comparison of computer costs of R and Armadillo engines of this package is attached.

## How to Use **elnglm**

### Installation

The **elnglm** package is available on the Github repository \url{https://github.com/PingYangChen/elnglm}. The user can install **elnglm** in R by the command of the **devtools** package.
```r
install.packages("devtools")
devtools::install_github("PingYangChen/elnglm")
```

Then, activate **elnglm** by the standard command in R.
```r
library(elnglm)
```


### Simulation Data Generator

Use `glmDataGen` to generate simulation data with continuous response (`family = "gaussian"`), binary response (`family = "binomial"`) and multi-categorical response (`family = "multinomial"`).


For continuous and binary types of response variable, input the real-valued intercept `trueb0` and real-valued coefficient vector `trueb`. The length of `trueb` should be equal to the dimension of the predictor matrix, `d`.  The continuous response variable are generated based on Normal error assumption, and hence, the user needs to specify the stanard deviation of the error term, `s`.

```r
set.seed(99)
trueb0 <- 1
trueact <- c(1, 1, 1, 0, 0, 0, 0, 0, 0, 0)
trueb <- runif(10, -1, 1)*10
trueb[which(trueact == 0)] <- 0 
# Continuous response
df <- glmDataGen(n = 500, d = 10, family = "gaussian", trueb0, trueb, s = 1, seed = 100)
# Binary response
dfb <- glmDataGen(n = 500, d = 10, family = "binomial", trueb0, trueb, seed = 100)
```

For multi-categorical response variable, the funciton simulates the data based on  multi-logistic model. Suppose there are $K$ categories, the user needs to input $K$ sets of regression coefficients. The input of intercept term is a real-valued vector of length $K$ and the input of the regression coefficients is a $d\times K$ matrix. See details in the "Theoretical Development" section.

```r
set.seed(99)
trueb0m <- c(1, 1, 1)
trueactm <- cbind(
  c(1, 1, 1, 0, 0, 0, 0, 0, 0, 0),
  c(0, 0, 0, 1, 1, 1, 1, 0, 0, 0),
  c(0, 0, 0, 0, 0, 1, 1, 1, 1, 0)
)
truebm <- matrix(runif(10*3, -1, 1)*10, 10, 3)
for (m in 1:3) { truebm[which(trueactm[,m] == 0),m] <- 0 }
#
dfm <- glmDataGen(500, 10, family = "multinomial", trueb0m, truebm, seed = 100)

```


### Modelling and Prediction


Use `glmPenaltyFit` to fit the generalized linear model (GLM) with elastic net penalty.  The function supports the GLM of continuous response (`family = "gaussian"`), binary response (`family = "binomial"`) and multi-categorical response (`family = "multinomial"`).  The function fits GLMs for different penalty parameters as the same time. The vector of penalty parameters can be automatically generated, from the theoretically obtained maximal value, $\lambda_{max}$, to its `minLambdaRatio`-times multiple, of length `lambdaLength`. Alternatively, the function accepts user-defined input lambda vector via `lambdaVec`.

```r
mdl <- glmPenaltyFit(y = df$y, x = df$x, family = "gaussian", lambdaLength = 100, 
                     maxit = 1e5, tol = 1e-7, alpha = 0.5, minLambdaRatio = 1e-3, 
                     ver = "arma")
```

```r
colorlist <- sapply(1:nrow(mdl$b), function(k) {
  val <- runif(3); rgb(val[1], val[2], val[3])
})
plot(mdl$lambda, mdl$b[1,], type = "l", col = colorlist[1], ylim = range(mdl$b),
     xlab = "lambda", ylab = "Coefficient Value")
for (i in 2:nrow(mdl$b)) {
  points(mdl$lambda, mdl$b[i,], type = "l", col = colorlist[i])
}
legend("topright", legend = sprintf("b%d", 1:nrow(mdl$b)), ncol = 3,
       col = colorlist, lty = 1, cex = 0.6)
```
![Coefficient Path]({{ site.url }}{{ site.baseurl }}/assets/images/2022-01-05/coefpath.png)
{: .image-right}

The function `glmPenaltyCV` supports the tuning for the penalty parameter via k-fold cross validation. 

```r
mdlcv <- glmPenaltyCV(y = df$y, x = df$x, family = "gaussian", lambdaLength = 100, 
                      maxit = 1e5, tol = 1e-7, alpha = 0.5, minLambdaRatio = 1e-3,
                      nfolds = 10, ver = "arma")
```

```r
plot(mdlcv$lambda, mdlcv$cvscore, type = "l", xlab = "lambda", ylab = "RMSE Value")
```
![RMSE Values]({{ site.url }}{{ site.baseurl }}/assets/images/2022-01-05/lincv.png)
{: .image-right}

```r
# Intercept of the linear model
mdlcv$b0[mdlcv$lambdaBestId]
# Regression coefficients of the linear model
mdlcv$b[,mdlcv$lambdaBestId]
```


The function `glmPenaltyPred` predicts the response for the new predictor.

```r
# Predict for new data
xnew <- matrix(rnorm(10), 1, 10)
yp <- glmPenaltyPred(mdlcv, xnew)
dim(yp) # 1 100
print(yp[,mdlcv$lambdaBestId])
```


Below is the GLM with elastic net penalty fitting for the binary response.

```r
mdlcv_b <- glmPenaltyCV(y = dfb$y, x = dfb$x, family = "binomial", lambdaLength = 100,
                        maxit = 1e5, tol = 1e-7, alpha = 0.5, minLambdaRatio = 1e-3, 
                        nfolds = 10, ver = "arma")
```


```r
plot(mdlcv_b$lambda, exp(-mdlcv_b$cvscore), type = "l", xlab = "lambda", ylab = "Accuracy Value")
```
![ACC Values]({{ site.url }}{{ site.baseurl }}/assets/images/2022-01-05/logiscv.png)
{: .image-right}

```r
# Intercept of the logistic model
mdlcv_b$b0[mdlcv_b$lambdaBestId]
# Regression coefficients of the logistic model
mdlcv_b$b[,mdlcv_b$lambdaBestId]
```


```r
# Predict for new data
xnew <- matrix(rnorm(10), 1, 10)
#
yp <- glmPenaltyPred(mdlcv_b, xnew, type = "response")
print(yp[,mdlcv_b$lambdaBestId])
#
yp <- glmPenaltyPred(mdlcv_b, xnew, type = "probability")
print(yp[,mdlcv_b$lambdaBestId])
#
yp <- glmPenaltyPred(mdlcv_b, xnew, type = "link")
print(yp[,mdlcv_b$lambdaBestId])
```


Below is the GLM with elastic net penalty fitting for the multi-categorical response.
```r
mdlcv_m <- glmPenaltyCV(y = dfm$y, x = dfm$x, family = "multinomial", lambdaLength = 100, 
                        maxit = 1e5, tol = 1e-7, alpha = 0.5, minLambdaRatio = 1e-3, 
                        nfolds = 10, ver = "arma")
```

```r
# Intercept of the multinomial model
mdlcv_m$b0[,mdlcv_m$lambdaBestId]
# Regression coefficients of the multinomial model
mdlcv_m$b[,,mdlcv_m$lambdaBestId]
```


```r
# Predict for new data
xnew <- matrix(rnorm(10), 1, 10)
#
yp <- glmPenaltyPred(mdlcv_m, xnew, type = "response")
print(yp[,,mdlcv_m$lambdaBestId])
#
yp <- glmPenaltyPred(mdlcv_m, xnew, type = "probability")
print(yp[,,mdlcv_m$lambdaBestId])
#
yp <- glmPenaltyPred(mdlcv_m, xnew, type = "link")
print(yp[,,mdlcv_m$lambdaBestId])
```


## Performace Test

A simple test of this self-developed **elnglm** package is conducted. 

```r
library(glmnet)
# Set for true values of the regression coefficients
trueb0 <- 1
trueb <- c(6, -6, 2, -0.9, 0, 0, 0, 0, 0, 0)
nRep <- 50
testResult <- list(
  "elnglm_r" = list(
    "estb" = matrix(0, nRep, length(trueb)), 
    "cputime" = rep(0, length(trueb))
  ),
  "elnglm_arma" = list(
    "estb" = matrix(0, nRep, length(trueb)), 
    "cputime" = rep(0, length(trueb))
  ),
  "glmnet" = list(
    "estb" = matrix(0, nRep, length(trueb)), 
    "cputime" = rep(0, length(trueb))
  )
)

for (iRep in 1:nRep) {
  # Generate simulation data
  df <- glmDataGen(500, 10, family = "gaussian", trueb0, trueb, s = 1, 
                   seed = iRep)
  # Run cv for self-developed elnglm using R engine
  cpu0 <- system.time({
    mdlSelf_r <- glmPenaltyCV(y = df$y, x = df$x, family = "gaussian", 
                              lambdaLength = 100, maxit = 1e5, tol = 1e-7,
                              alpha = 0.5, minLambdaRatio = 1e-3, nfold = 5, 
                              ver = "r")
  })
  testResult$elnglm_r$estb[iRep,] <- mdlSelf_r$b[,mdlSelf_r$lambdaBestId]
  testResult$elnglm_r$cputime[iRep] <- cpu0
  
  # Run cv for self-developed elnglm using armadillo engine
  cpu1 <- system.time({
    mdlSelf_arma <- glmPenaltyCV(y = df$y, x = df$x, family = "gaussian", 
                                 lambdaLength = 100, maxit = 1e5, tol = 1e-7,
                                 alpha = 0.5, minLambdaRatio = 1e-3, nfold = 5, 
                                 ver = "arma")
  })
  testResult$elnglm_arma$estb[iRep,] <- mdlSelf_arma$b[,mdlSelf_arma$lambdaBestId]
  testResult$elnglm_arma$cputime[iRep] <- cpu1
  
  # Run cv for the glmnet package
  cpu2 <- system.time({
    mdlPkg <- cv.glmnet(y = df$y, x = df$x, family = "gaussian", 
                        lambda = mdlSelf_arma$lambda, maxit = 1e5, thresh = 1e-7,
                        alpha = 0.5, type.measure = "mse", nfolds = 5)
  })
  testResult$glmnet$estb[iRep,] <- mdlPkg$glmnet.fit$beta[,which.min(mdlPkg$cvm)]
  testResult$glmnet$cputime[iRep] <- cpu2
}
```

```r
trueact <- abs(trueb) > 0
#
elnglm_r_act <- abs(testResult$elnglm_r$estb) > 0
elnglm_arma_act <- abs(testResult$elnglm_arma$estb) > 0
glmnet_act <- abs(testResult$glmnet$estb) > 0
#
sumTable <- rbind(
  c(
    sapply(1:length(trueact), function(j) 100*mean(elnglm_r_act[,j] == trueact[j])),
    mean(testResult$elnglm_r$cputime)
  ),
  c(
    sapply(1:length(trueact), function(j) 100*mean(elnglm_arma_act[,j] == trueact[j])),
    mean(testResult$elnglm_arma$cputime)
  ),
  c(
    sapply(1:length(trueact), function(j) 100*mean(glmnet_act[,j] == trueact[j])),
    mean(testResult$glmnet$cputime)
  )
)
dimnames(sumTable) <- list(
  c("elnglm_r", "elnglm_arma", "glmnet"), 
  c(sprintf("x%02d", 1:length(trueact)), "CPU Time (s)")
)
```

```r
# Accuracy values, %, of identifying active/inactive status of the covariates
# and computing time, in seconds, of the functions.
print(sumTable)
#>             x01 x02 x03 x04 x05 x06 x07 x08 x09 x10 CPU Time (s)
#> elnglm_r    100 100 100 100  32  28  28  28  28  32       0.3148
#> elnglm_arma 100 100 100 100  34  32  32  28  28  34       0.0222
#> glmnet      100 100 100 100  14  36  32  28  34  38       0.0420
```

## Theoretical Development

Let $(\mathbf{y}, \mathbf{X})$ be the data of $n$ observations, where $\mathbf{y}\in\mathbb{R}^n$ is the vector of the response variable and $\mathbf{X}\in\mathbb{R}^{n\times p}$ is the matrix of $p$ covariates.
The linear model is the common approach to describe the linear associations between the covariates and the response variable. For a given $\mathbf{x}$ and proper link function $g(\cdot)$, the expected response has the form of the linear combination of the covariates: 
$$
g\left[E(Y\mid\mathbf{x})\right] = \beta_0 + \sum_{j=1}^p x_j\beta_j
$$
where $\beta_0$ is the intercept term and $\boldsymbol\beta=(\beta_1, \ldots, \beta_p)$ are the regression coefficients.  

(TBW)

Suppose $Y$ is continuous response variable. The elastic net estimates of $\beta_0$ and $\boldsymbol\beta$ minimize the following objective function
$$
\underset{\beta_0, \boldsymbol\beta}{\min} \frac{1}{2n}\sum_{i=1}^n \left( y_i - \beta_0 - x_i^\top\boldsymbol\beta \right)^2 + \lambda \sum_{j=1}^p\left[ \frac{1-\alpha}{2} \beta_j^2 + \alpha  |\beta_j| \right]
$$
where $\alpha$ is the elastic net mixing parameter between the ridge regression penalty ($\alpha = 0$) and lasso ($\alpha = 1$) penalty; $\lambda$ is the shrinkage parameter.



For binary responses, $Y\in\{0,1\}$, the linear logistic regression is considered. That is, 
$$
p(x)=\mathcal{Pr}(Y = 1\mid x) = \frac{1}{1 + e^{-(\beta_0 + x^\top\boldsymbol\beta)}},
$$
or equivalently,
$$
\log{\frac{p(x)}{1-p(x)}}=\beta_0 + x^\top\boldsymbol\beta
$$

The elastic-net logistic regression estimates of $\beta_0$ and $\boldsymbol\beta$ maximize the penalized likelihood function
$$
\underset{\beta_0, \boldsymbol\beta}{\max} \frac{1}{n}\sum_{i=1}^n 
\Big[ I\{y_i=0\}\log{\{1-p(x_i)\}} + I\{y_i=1\}\log{p(x_i)} \Big] + 
\lambda \sum_{j=1}^p\left[ \frac{1-\alpha}{2} \beta_j^2 + \alpha  |\beta_j| \right]
$$
When the response variable has multiple categories, the multinomial logistic model is used. Suppose there are $K$ categories, $\{0, 1, \ldots, K-1\}$, the probability that $Y$ belongs to the $k$-th category is
$$
p_k(x)=\mathcal{Pr}(Y = k\mid x) = \frac{e^{\beta_{0k} + x^\top\boldsymbol\beta_k}}{\sum_{l=0}^{K-1} e^{\beta_{0\mathcal{l}} + x^\top\boldsymbol\beta_\mathcal{l}}},
$$
where $\beta_{0k}$ and $\boldsymbol\beta_k=(\beta_{1k}, \beta_{2k}, \ldots, \beta_{pk})$ are the coefficients of the $k$-th logistic model.

The elastic-net estimates of the multinomial regression coefficients maximize the penalized likelihood function
$$
\underset{\beta_{0k},\boldsymbol\beta_k,k=0,\ldots,K-1}{\max}\frac{1}{n}\sum_{i=1}^n\Big[\sum_{k=0}^{K-1}\log{p_k(x_i)}\Big]+\lambda\sum_{j=1}^p\sum_{k=0}^{K-1}\left[ \frac{1-\alpha}{2}\beta_{jk}^2+\alpha|\beta_{jk}|\right].
$$
