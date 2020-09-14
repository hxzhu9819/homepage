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

## Autograd
Before we dive in to Autograd, we need to understand the goal of **back propogation**.
* backprop: measure how much each param affects the loss

Autograd accomplishes the exact same thing as *forward* and *backprop*, but it adds code to forward propagation in order to automate backprop.

### Forward Pass
During forward propagation, autograd automatically constructs a **computational graph**. The computational graph tracks how elementary operations modify data throughout the function. Autograd does this in the background, while the function is being run.

\[PLACEHOLDER: a computational graph\]

#### Computational graph
There are three types of nodes,
* Constant Node
* Accumulated Node
* BackwardFunction

Nodes are added whenever an operation occurs. In practice, we call `.apply()` method of the operation. Calling `apply()` on the subclass implicitly calls `Function.apply()`, which does the following:

1. Create a node object for the operation’s output
2. Run the operation on the input(s) to get an output tensor
3. Store information on the node, which links it to the comp graph
  1. link the output tensor with parent tensors. For each parent tensor
    1. If the parent tensor has nodes, then add parent nodes to current node's *parent list*, and go to 4
    2. If the parent tensor does not have nodes (`grad_fn` is None)
      1. If the parent tensor `is_leaf=True `, `requires_grad=False`, then create a **Constant Node** for it, and append `None` to current node's *parent list*, and go to 4
      2. If the parent tensor `is_leaf=True`, `requires_grad=True`, then create a **Accumulated Node** for it, and append the parent tensor to the current node's *parent list*, and go to 4
4. Store the node on the output tensor `grad_fn=BackwardFunction`
5. Return the output tensor


*Credits: CMU 11785 - Introduction to Deep Learning Fall 2020 HW1*

### Userful external links
1. [Autograd Explained](https://www.youtube.com/watch?v=MswxJw-8PvE&t=605s)