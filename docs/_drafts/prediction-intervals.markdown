---
layout: post
title:  "Prediction Intervals and the User Experience"
date:   2023-02-16 20:55:50 +0100
category: Work
tags: MachineLearning
---
![Kitchentable Magic](/images/experimenting-with-text-generators/kitchentable-magic.jpg)
*"Multiverse Kitchen Table" (this image was created with the assistance of DALL·E 2)*
  
Motivation: Users of ML Apps need some form of quality measure for the prediction. Whether they can trust it or should explore other parameter spaces for the model. The problem starts when you want to deploy a model which produces prediction (or confidence intervals) in the cloud (e.g. in AWS) 
<!--more-->

## Prediction intervals
Here a description of how to obtain what prediction intervals are, and how you can obtain them from trained models  

## What worked
Here a description how you would obtain predition intervals with the "mapie" python library. Was not working in a cloud environment.

## What didn't work in my setup
Here a description how you would obtain predition intervals with the "doubt" python library. Was not working in a cloud environment.

## Some differences
What are the differences between how mapie gets prediction intervals compared to the doubt library. What could be the advantage of using doubt?

### Subsection
(Use of lists, use of links)
If you are interested in some articles on the topic, these are the ones that inspired me to do this project:  
- [The Unreasonable Effectiveness of Recurrent Neural Networks][karpathy-rnns]

[karpathy-rnns]: http://karpathy.github.io/2015/05/21/rnn-effectiveness/
