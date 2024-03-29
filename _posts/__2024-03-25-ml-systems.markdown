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
is (e.g., lower the loss, better the prediction). finally, `algorithm` is usually a gradient descent based
approach used to adjust parameters of the model in a direction that lowers the loss. 

<h3>Data</h3>
In DL, data is usually stored in n-dimentional arrays, which also called tensors (e.g., ndarray in NumPy, 
Tensor in PyTorch). For example, we can store a 3-channel 32x32 image as a **`3 x 32 x 32`** tensor.
`vector` and `matrix` are simple forms of tensor with one and two axes respectively. These are useful 
for representing datasets, where rows correspond to individual records and columns correspond 
to distinct attributes.

<!-- | ![Three channel image]({{site.baseurl}}/assets/2024-01-27-ml-systems-2.png){: width="200"} 
| :--: |
| *RGB image* | -->

Tensor operations 
  * indexing and slicing - selecting certain parts of a tensor
  * elementwise: (scalar or vector or matrix or tensor) + tensorB
    * unary (e.g., exp)
    * binary (+, -, *, /, **)
      * elementwise multiplication of two matrices is their `Hadamard product`
  * linear algebraic operations
    * dot product  - sum over products of vector vector multiplication
    * matrix multiplication - matrix matrix multiplication
  * concatenating - where you stack up two tensors either along row (top and bottom) or along column (side by side)
  * broadcasting
    * expand one or both tensors by copying elements along axes with length 1 so that after this transformation, 
    two tensors have the same shape, which is necessary to perform some of the above operations
  * reduction - by default, invoking the sum function reduces a tensor along all of its axes, producing a scalar. 
  We could also reduce the tensor along a particular axis. 



* Key components
  * Data
    * Manipulation
    * Preprocessing

  * Model
    * Linear algebra
    * Calculus
    * Automatic Differentiation
    * Probability and Statistics

  * Algorithm to adjust the model parameters to optimize the objective function

* Linear Neural Networks for Regression/Classification
* Multilayer Perceptrons

* Covariate shift: shift in the input distribution given constant conditional distribution of the output labels
  * correct with importance weighting (assign higher weights to training samples
  * that are more similar to the test data distribution) and domain adaptation
* Label shift: shift in the output distribution given constant conditional distribution of the inputs
  * correct with importance weighting and direct estimation of label distribution 
* Concept shift: shift in the underlying relationship between inputs and outputs
  * correct with online learning and ensemble methods


<hr><br>
