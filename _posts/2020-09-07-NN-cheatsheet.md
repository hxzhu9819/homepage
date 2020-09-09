---
title: "NN - A Cheat Sheet"
lang: en
toc: true
permalink: /posts/nncheatsheet
tags:
  - NN
  - study
---
## Before we start
Neural Networks, nowadays, are like vehicles -- Anyone can operate them without knowing how they work. This post, for my best interest, and hopefully, will collect necessary informations and the concepts behind the scene if you truly want to build them from wheels.

## Mathematical Background
### Gradient, Directional Derivative, Backprob and Audograd
#### Gradient
The gradient of a scalar-valued multivariable function $f(x,y, ...)$ is a vector of its partial derivative
$ \bigtriangledown f = [\frac{\partial f}{\partial x}, \frac{\partial f}{\partial y}, ...]^T $.

To help understand the underlying meaning. Imagine that you are standing at a point $P(x_0, y_0, ...)$, then the gradient $\bigtriangledown f(x_0, y_0, ...)$ tells you the direction to which you should travel along that can increase the value of $f$ most quickly.

#### Directional Derivative
The directional derivative is a scalar, showing the rate at which $f$ will change while the inputs move along a given vector $\textbf{v}$.<br/>
$\bigtriangledown_{\textbf{v}} f = \bigtriangledown f \cdot \textbf{v}$

#### Userful external links
1. [Automatic differentiation](https://en.wikipedia.org/wiki/Automatic_differentiation#/Reverse_accumulation)