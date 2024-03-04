---
layout: post
title:  "Optimize My Cube"
date:   2023-02-16 20:55:50 +0100
category: Work
tags: Optimization DesignOfExperiments AdaptiveExperimentalDesign MachineLearning
---
![An image](/images/optimize-my-cube/an-image.jpg)
*"Image Title" (this image was created with the assistance of DALL·E 2)*
  
...  
<!--more-->

## Guided experimentation
Let's say that you are a scientist in the lab and have a set of substances which you can combine for a reaction. Say we start from scratch, so we have no idea which substance or which combination of substances bring us closer to a desired reaction product. Maybe your first thought would be to create a systematic plan, a matrix of all possible mixtures. But already with 10 substances you realize that it is quite cumbersome to systematically conduct every possible experiment.  
  
As a data scientist, you would love such a systematic experimental study (aka Design of Experiments) since the resulting dataset is evenly sampled within the space of possible parameters. After the experiments are finished and all results collected, you would build a fine model which you could use to find the optimum between your sampled datpoints.<sup>1</sup> But even "small" designs are still time consuming for the scientist in the lab. Can't we train the machine learning model on-the-fly, incrementally with each completed experiment? Initially such a model would surely be bad, but it could already suggest a new datapoint to sample.  
  
Wait a second, actually there is something missing here: A model doesn't *suggest* anything, and a model doesn't know anything about the direction you want to take with your experiments. The missing component is an optimization algorithm that can use the model to *predict* new datapoints before suggesting them to the lab scientist.
  
## Scoping the field - what I want and what not
What I described in the last section sounds a lot like *Active Learning*. And there are many python libraries focused on active learning - but it's not what we *need*. The primary goal of active learning is to construct a good model. Period. It's nice that it aims to use as few examples as possible but the focus is on the accuracy of the model, not on optimizing a property that a lab scientist cares about. Similarly, you could say that grinding through a matrix of all possible combinations (aka design of experiments), is done purely to build a *good model* with as few experiments as possible.

In most industrially relevant applications, however, a lab scientist wants to optimize a product with as few examples as possible (without caring about the model behind). Enter adaptive experimental design ...
  
Adaptive experimental design is a subfield of active learning that combines active learning with optimization. The goal of adaptive experimental design is to select the most informative examples to label while also optimizing some property of the experiment, such as minimizing the variance of the estimates or maximizing the power of the test. Adaptive experimental design is particularly useful when the cost of performing experiments is high, and the goal is to minimize the number of experiments required to achieve a desired level of accuracy
  
## Nonlinear optimization
Augmented Lagrangian algorithm
There is one algorithm in NLopt that fits into all of the above categories, depending on what subsidiary optimization algorithm is specified, and that is the augmented Lagrangian method described in:

Andrew R. Conn, Nicholas I. M. Gould, and Philippe L. Toint, "A globally convergent augmented Lagrangian algorithm for optimization with general constraints and simple bounds," SIAM J. Numer. Anal. vol. 28, no. 2, p. 545-572 (1991).
E. G. Birgin and J. M. Martínez, "Improving ultimate convergence of an augmented Lagrangian method," Optimization Methods and Software vol. 23, no. 2, p. 177-195 (2008).
This method combines the objective function and the nonlinear inequality/equality constraints (if any) in to a single function: essentially, the objective plus a "penalty" for any violated constraints. This modified objective function is then passed to another optimization algorithm with no nonlinear constraints. If the constraints are violated by the solution of this sub-problem, then the size of the penalties is increased and the process is repeated; eventually, the process must converge to the desired solution (if it exists).

### Subsection
(Use of lists, use of links)
If you are interested in some articles on the topic, these are the ones that inspired me to do this project:  
- [The Unreasonable Effectiveness of Recurrent Neural Networks][karpathy-rnns]


<sup>1</sup> Let's be real here: Usually the data scientist is obsolete by that point because, by the time the lab scientist has finished conducting all experiments, her intuition is already *good enough* to find the optimum without any model.

[karpathy-rnns]: http://karpathy.github.io/2015/05/21/rnn-effectiveness/
