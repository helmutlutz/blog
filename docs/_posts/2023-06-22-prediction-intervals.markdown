---
layout: post
title:  "A Matter of User Experience: Prediction Intervals"
date:   2023-06-22 20:55:50 +0100
category: Work
tags: MachineLearning
---
![Uncertainty](/images/prediction-intervals/article_header.jpg)
*"Uncertainty" (this image was created with the assistance of Stable Diffusion)*
  
When using machine learning (ML) applications, end-users often seek a reliable measure to assess the uncertainty of predictions. This is especially true when the stakes are high, such as medical diagnosis or accident prevention. One such measure are *prediction intervals*. When you're working in python, they are basically just one more wrapper for your model. From my experience, the value of an uncertainty measure is often underestimated by end users at the start of the project. But in the end, it can make all the difference when it comes to user experience and adoption.  
<!--more-->

## Prediction Intervals: A Primer
Prediction intervals play a vital role in estimating the uncertainty of ML predictions. These intervals establish a range within which the true value is likely to fall. Two popular methods for obtaining prediction intervals are conformal prediction and quantile regression. In conformal prediction you fit a model on a *training* data set, then quantify uncertainty based on the residuals of a held-out validation (or *calibration*) set. While the fraction of correct predictions matches well the chosen confidence interval, the downside is the (almost) constant interval width.  
  
Quantile regression is performed by fitting two models (e.g. random forests) on the 5% percentile ($q_{\alpha,low}$) and 95% percentile ($q_{\alpha,high}$) and form the corresponding intervals with these models. Compared to conformal prediction, quantile regression lacks precision in terms of the percentage of incorrect predictions on new data, but it's adaptive to heteroscedastic data and produces variable prediction interval width, which is reflective of the uncertainty of individual data points. 
  
Conformalized Quantile Regression (CQR) combines the best of both worlds. You can find the original paper [here][cqr-paper]. Like conformal prediction, CQR aims to provide a valid *coverage*, i.e. keeping the number of mistakes on new data close to a predefined tolerance level. It does so without making strong assumptions like a gaussian distribution of the data - as in quantile regression, it is designed to be adaptive to heteroscedasticity.  

### CQR under the hood
As in conformal prediction, we first split the data into a training set and a calibration set. We then fit two quantile models ($q_{\alpha,high}$ and $q_{\alpha,low}$) on the training samples. The next step is critical and differs from quantile regression: on the calibration data set, we compute *conformity scores* which quantify the errors made by $q_{\alpha,high}$ and $q_{\alpha,low}$. Finally, we calculate the (1-alpha) quantile from all conformity scores. For a new data point we can calculate the lower (upper) end of the prediction interval by subtracting (adding) the conformity score's quantile to the prediction of $q_{\alpha,low}$ ($q_{\alpha,high}$).

### Generating Prediction Intervals with the "MAPIE" Python module
One effective method to get your CQR prediction intervals is with the [MAPIE][mapie-docs] Python module. This module integrates with custom scikit-learn models and pipelines. I've also tested and used it for my custom scikit-learn pipelines on AWS Sagemaker, streamlining the process of generating prediction intervals. But make sure you "parse" the output of your model pipeline since you will not only get the prediction but also the corresponding prediction intervals as numpy arrays.

## Conclusion: Improving Prediction Confidence with Conformal Quantile Regression
Whether you are at the beginning of a project or in the beta testing phase, consider offering prediction intervals as additional features to your customers to increase trust and user adoption. Conformal Quantile Regression (CQR) offers an effective solution to address the limitations of conformal prediction and quantile regression when estimating prediction intervals. A quick and easy way to obtain CQR intervals with your existing scikit-learn pipelines is by using the MAPIE module. So far, stay curious and keep learning! 
  
[cqr-paper]: https://arxiv.org/abs/1905.03222
[mapie-docs]: https://mapie.readthedocs.io/en/latest/
