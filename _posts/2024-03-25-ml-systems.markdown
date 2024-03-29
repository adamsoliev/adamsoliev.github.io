---
layout: post
title:  "Deep Learning"
date:   2024-03-25 22:08:32 -0600
categories: jekyll update
---

This is an introduction to deep learning (DL) without heavy jargon. 
Potentially useful for educational purposes.

DL is a subfield of machine learning (ML), a broad field of algorithms used to approximate functions
that take in some input and produce some useful output. In the context of DL, the said function is 
'deep' (or layered/complex). Key components of any ML problem are data, a model (or above function), a loss function 
and an algorithm (for adjusting model parameters), as depicted below.

![Key components]({{site.baseurl}}/assets/2024-01-27-ml-systems-1.png){: width="600"}

`data` can be anything (e.g., text, images, etc), given each of its examples consists of a set of features, 
including a label that we want to predict. `model` is a computational graph that
ingests data and outputs predictions. `loss function` is a tool used to measure how good the prediction
is (e.g., lower the loss, better the prediction). finally, `algorithm` is another tool used to adjust 
parameters of the model in a direction that lowers the loss. 

<!-- ![Key components]({{site.baseurl}}/assets/2024-01-27-ml-systems-2.png){: width="300"} -->

To make things a bit more concrete, say you are given coordinates of points in 2D plane and 
asked to predict new `y` given a value of `x`. You probably find the line that fits best 
to those points and calculate `y` based on the found function. Simple enough, right? Except what if your input is 
n-dimentional and output is m-dimentional (e.g., dataset of 28x28 grayscale images and 10 categories 
they belong to)?

For such cases, you need a much larger and more complex function (model) that can express complex relationships between
inputs and outputs. 

* Key components
  * Data
    * Manipulation
    * Preprocessing

  * Model
    * Linear algebra
    * Calculus
    * Automatic Differentiation
    * Probability and Statistics

  * Objective function

  * Algorithm to adjust the model parameters to optimize the objective function

* Linear Neural Networks for Regression/Classification
* Multilayer Perceptrons

Covariate shift: shift in the input distribution given constant conditional distribution of the output labels
  correct with importance weighting (assign higher weights to training samples
  that are more similar to the test data distribution) and domain adaptation
Label shift: shift in the output distribution given constant conditional distribution of the inputs
  correct with importance weighting and direct estimation of label distribution 
Concept shift: shift in the underlying relationship between inputs and outputs
  correct with online learning and ensemble methods


<hr><br>
