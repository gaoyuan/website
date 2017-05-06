---
title: Cramér-Rao bound and the Gauss-Markov theorem
date: 2017-05-05
published: true
author: gaoyuan
image: img/fig.png
mathjax: on
tags: machine learning, least squares, linear regression 
---
The Gauss-Markov theorem states that in a linear model $y = X\beta + \epsilon$, where $E[\epsilon] = 0$, and $Var[\epsilon] = \sigma^2I$, the ordinary least square estimator $\hat{\beta} = (X^TX)^{-1} X^T y$ has the smallest variance among all the **linear** unbiased estimators. 

You can find the proof of the Gauss-Markov theorem on [Wikipedia](https://en.wikipedia.org/wiki/Gauss%E2%80%93Markov_theorem). Today when reading about the Cramér-Rao bound, I find that a stronger argument can be made about $\hat{\beta}$ if we further assume $\epsilon \sim \mathcal{N}(0, \sigma^2I)$, in which case $\hat{\beta}$ is the minimum variance estimator among **all** unbiased estimators of $\beta$.  

The Cramér-Rao bound states that the variance of any unbiased estimator of a parameter $\theta$ is bounded below by $I(\theta)^{-1}$, the inverse of its Fisher information matrix. Under the Gaussian noise assmuption, $y \sim \mathcal{N}(X\beta, \sigma^2I)$, and Fisher information matrix of $\beta$ for the linear model is
\begin{equation}
I(\beta) = - E\left[\frac{\partial^2 \log f(y;\beta)}{\partial\beta^2}\right] = \frac{X^TX}{\sigma^2},
\end{equation}
where $f(y;\beta)$ is the pdf of $y$. Note that $Var(\hat{\beta}) = (X^TX)^{-1}\sigma^2 = I(\beta)^{-1}$, therefore the ordinary least square estimator $\hat{\beta}$ attains this lower bound. Since an minimum variance unbiased estimator is essentially unique, $\hat{\beta}$ is also the only one with this variance.
