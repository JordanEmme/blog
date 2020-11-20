---
title: "Deep Neural Networks: a Basic Maths Introduction"

date: 2020-11-16T17:19:31Z
draft: false

description: "A quick maths introduction to Deep Neural Networks"
summary: "In this post I give a basic introduction of the main idea behind Artificial Neurons and Deep Neural Networks. In particular, I explain the classical techniques of linear and logistic regression, as well as the gradient descent algorithm"

tags: ["Maths", "Neural Networks", 'Deep Learning']
categories: ["Maths", "Deep Learning"]

featuredImage: ""


lightgallery: false
---

## Preliminary Rambling

First, let me explain what I am trying to achieve with this post, and why.  
If, like me, you have a background in pure mathematics, and have been following from afar the general public discussions about Deep Neural Networks, you have probably heard, amongst other things, that the reason this technique works and is so revolutionary is because it simulates the learning process of the human brain. Furthermore, if you only quickly glance over the technical terms, you may see things such as learning rate, perceptrons, neurons, hidden layers, forward propagation, backpropagation etc... My goal with this post is to demistify these claims and jargon with a bit of maths formalism. 

I want to make perfectly clear that nothing written here is either new, original, or even profound. I am not pretending to make any attempt at *formalising* the theory, because it already is, but rather presenting it in a way that makes sense to me and, hopefully, mathematicians, in a concise post. By experience, I've seen a number of, otherwise great, tutorials about Deep Neural Networks which focus a lot on their implementations --- as they should --- but give little to no formal definitions. When they do try to explain the maths behind, it is often by resorting to the aforementioned buzzwords and through handwaving. This can be a bit frustrating to the mathematician who wants a solid understanding rather than the usual 'marketing speech'.

If you are a seasoned mathematician interested in implementing deep learning models, you'll be able to read between the lines of the tutorials, or any other content for that matter, and infer everything that I'll say here. So, unless you specifically want an easy, purely theoretical, explanation of the main idea behind neural networks without, this post is not targeted at you.

If however, you are still in your early years of learning maths, or come from another background, know a little bit of calculus and linear algebra, and are interested in learning the basic maths behind Deep Learning, then this post is written especially for you.

## What Is Deep Learning?

Deep Learning is a subset of methods from Machine Learning. These methods involve the use of artificial neural networks for statistical learning. In essence, the goal is to train models on a data set in order to get a better understanding of the data and/or predict outcomes on new data points. The learning can be supervised, unsupervised or semi-supervised. In this post, we will limit ourselves to supervised learning. We'll dedicate ourselves to defining these terms and explain why these methods work, at least theoretically.


### Supervised Learning

It may not be the most exciting class of problems when it comes to A.I., but it is good to start here to in order to fix some ideas. Let us start with some notations and definitions.

* An *input variable* with $p$ *features* is a vector in $\mathbb{R}^p$, often denoted by $X$
* An *output variable* is a real number, often denoted by $y$, on which we wish to make predictions
* A $n$ points *data set* is a matrix $\mathbf{X}$ in $\mathcal{M}_{n,p}(\mathbb{R})$ such that

$$
\mathbf{X} = 
\begin{pmatrix} 
x_{1,1} & x_{1,2} & \cdots & x_{1,p} \\\ 
x_{2,1} & x_{2,2} & \cdots & x_{2,p} \\\ 
\vdots  & \vdots  & \ddots & \vdots  \\\ 
x_{n,1} & x_{n,2} & \cdots & x_{n,p} 
\end{pmatrix}
$$

* $x_i = \begin{pmatrix}x_{i,1} \\\ x_{i,2} \\\ \vdots \\\ x_{i,p} \end{pmatrix}$ is the $i^\text{th}$ <em>observation</em>
* $\mathbf{x}_j$ is the $j^\text{th}$ column of the matrix $\mathbf{X}$ 
* $\mathbf{y} = \begin{pmatrix} y_1 \\\ y_2 \\\ \vdots \\\ y_n \end{pmatrix}$ is the observation vector of the output variables

The fundamental assumption for statistical learning is the following. We assume the existence of a function $f: \mathbb{R}^p \rightarrow \mathbb{R}$ such that
$$
y = f(X) + \varepsilon.
$$
The function $f$ is the *systemic* information that $X$ carries about $y$ and $\varepsilon$ is a random *error term*, which is centered and independent from the input variable $X$. 

In the case of *supervised learning*, we have both a data set $\mathbf{X}$ and its observation vector $\mathbf{y}$, and we use both these informations to *learn* (i.e. compute) an estimate $\hat{f}$ of $f$. If this estimate is a good approximation of $f$, it should then allows to make prediction on new data points by inputting them in our function $\hat f$. We usually denote predictions $\hat y$.

It is to be noted that, for the sake of predictions, we are not necessarily interested in what $\hat f$ is. This estimate is to be computed using a range of techniques, one of which can be Deep Neural Networks, and is then used as a black box for future predictions.

Some classical, real-world examples where supervised learning methods can be used are:

|Input|Output|
|:---:|:---:|
|House properties|House price|
|User info|Click on ad ($0$ or $1$)|
|Image|Object detection (positional or boolean)|

There are, of course, many more applications, but this is not the subject at hand. 

### Baby Cases

Before we dive into the theory of artificial neural networks, let us introduce (or remind) two easy examples of techniques for supervised learning, namely linear regression and logistic regression. These examples are a good way to easily go into the more complex theory of deep neural networks later, as they already present most of the main ideas behind them.

#### Linear Regression

Remember that the fundamental assumption for statistical learning is that there exists a function $f$ such that:

$$
y = f(X) + \varepsilon.
$$

In the case of linear regression, we assume that $f$ is an affine function from $\mathbb{R}^p$ to $\mathbb{R}$. In other words, there exists a vector

$$
W = \begin{pmatrix}W_1 \\\ \vdots \\\ W_p   \end{pmatrix}
$$

and a real number $b$ such that

$$
\forall X \in \mathbb{R}^p, \quad f(X) = W \cdot X + b
$$

where $W \cdot X$ is the inner product of $W$ and $X$, i.e.

$$
W\cdot X = \sum_{i = 1}^p W_i X_i.
$$

Our goal is to determine, both $W$ and $b$. Geometrically, given a data set $\mathbf{X}$ and an observation vector $\mathbf{y}$, we are trying to determine the affine hyperplane of $\mathbb{R}^{p+1}$ which is the "closest" to the cloud of points $\\{(x_i, y_i), \quad i \in \\{1,...,n\\} \\}$. 

Here, we need to define what 'closest' means. Our goal is to minimize the error, so we want each prediction $\hat y_i = \hat{f}(x_i)$ to be as close as possible to $y_i$. One possibility is to minimize the *Residual Sum of Square*:

$$
RSS =\sum_{i = 1}^n (\hat y_i - y_i)^2.
$$

This can be written as a function of $W$ and $b$ as follows:

$$
RSS(W, b) =\sum_{i = 1}^n (W \cdot x_i + b - y_i)^2.
$$

Now, we want to find the values of $W$ and $b$ which minimize the Residual Sum of Square. This is done by finding the critical points of the RSS seen as a function of $W$ and $b$. A classical trick to make our lives easier is the following. Set $\overrightarrow{\beta}$ as the vector:

$$
\overrightarrow{\beta} = \begin{pmatrix} W_1 \\\ \vdots \\\ W_p \\\ b \end{pmatrix}
$$

and consider the matrix 

$$
\bar{\mathbf{X}} = 
\begin{pmatrix}
x_{1,1} & x_{1,2} & \cdots & x_{1,p} & 1\\\ 
x_{2,1} & x_{2,2} & \cdots & x_{2,p} & 1\\\ 
\vdots  & \vdots  & \ddots & \vdots  & 1\\\ 
x_{n,1} & x_{n,2} & \cdots & x_{n,p} & 1
\end{pmatrix}.
$$

Notice that 

$$
RSS(\overrightarrow\beta) = (\bar{\mathbf{X}} \overrightarrow{\beta} - \mathbf{y})^\intercal \\ (\bar{\mathbf{X}} \overrightarrow{\beta} - \mathbf{y}).
$$

Under this form, a simple computation, differentiating once, shows that the critical point is given by

$$
\overrightarrow{\beta} = (\bar{\mathbf{X}}^\intercal \\ \bar{\mathbf{X}})^{-1}\bar{\mathbf{X}}^\intercal \\ \mathbf{y}.
$$

{{<admonition type="note" title="Remark">}}
It is not enough to conclude that this point is a local minimum. It is needed to show that the Hessian matrix is definite positive, but we will admit it here.
{{</admonition>}}

This concludes the linear regression case. If the assumption that $X$ and $y$ are linked linearly is correct, then this model should yield good predictive results. As a bonus, here, the expression of $f$ is completely known, since it is given by $W$ and $b$. It is thus easy to infer which features in the input have the most significant influence on the output. But of course, things would be too simple if the world was linear...

#### Logistic Regression

So what if we are handling problem of different nature? Say, for instance, that we do not want to predict a numerical output but rather a binary choice ('dead' vs 'alive', 'clicks on ad' vs 'doesn't click on ad', 'true' vs 'false' *etc*...). 

First, for obvious reasons, we'll encode this choice by saying that $y$ can take values in $\\{0,1\\}$. Then, given a dataset $\mathbf{X}$ along with observations $\mathbf{y}$, we want to build an estimator in the form of a probability measure. In essence, we'd like a function $\hat{f}$ such that

$$
\hat f(X) = \mathbb{P}(y = 1 \\  | \\  X).
$$

To that end, we introduce the *logistic function*, usually denoted by $\sigma$:

$$ 
\begin{aligned}
\sigma \colon \mathbb{R} & \longrightarrow \mathbb{R}\cr
                  t      & \longmapsto \frac{e^t}{e^t + 1}
\end{aligned}
$$

It is clear that this function takes its values $]0,1[$ and is thus a valid  candidate. The fundamental assumption in the case of logistic regression is that the systemic information $f$ is of the form

$$
f(X) = \sigma(W \cdot X + b).
$$

In other words, the assumption is that $X$ and $\log(\frac{\mathbb{P}(y = 1 \\  | \\  X)}{1-\mathbb{P}(y = 1 \\  | \\  X)})$ are linearly correlated.

As previously, we want to determine the best $W$ and $b$ such that the error is minimal. But there are two differences with the previous case:
1. We will not be using the RSS so we need to define another function which describe the distance between the predictions $\hat\mathbf{y}$ and $\mathbf{y}$.
2. It is not be possible, as before, to simply determine the minimum through calculus: we have to use the *Gradient Descent* method.

I'll start addressing the first point. Let us denote 

$$
\begin{aligned}
g_{W, b}(X): \mathbb{R}^p & \longrightarrow \mathbb{R} \cr
                       X & \longmapsto \sigma(W \cdot X + b) 
\end{aligned}
$$

and remark that, defining a probability such that

$$
\mathbb{P}(y = 1 \\  | \\  X, W, b) = g_{W, b}(X),
$$

we have

$$
\mathbb{P}(y = t \\  | \\  X, W, b) = g_{W, b}(X)^t(1 - g_{W, b}(X))^{1-t}.
$$

We thus introduce the following *loss function*

$$
\mathcal{L}(\hat y, y) = - (y\log(\hat y) + (1-y)\log(1 - \hat y)).
$$

{{<admonition type="note" title="Remark">}}
Naively, we can understand this function as simply penalizing predictions $\hat y$ which are far from $y$ by giving them large values. This is not the sole reason we chose this function. Another, perhaps shallow, reason is that the loss function defined through taking the logarithm (and multiplying by $-1$) of the probability above makes it convex, which is going to be important for the gradient descent to work. A more fundamental reason is that the cost function, which I will define in a moment, using this loss function, is going to be, more or less, the [log likelihood function](https://en.wikipedia.org/wiki/Likelihood_function) of the random variable $y$.
{{</admonition>}}

Using this loss function, given a data set $\mathbf{X}$ of sample size $n$, with targets $\mathbf{y}$, we define a cost function as the average of the loss function over the predictions.

$$
J(W, b) = \frac{1}{n} \sum_{i=1}^n \mathcal{L}(g_{W,b}(x_i), y_i)
$$

The challenge is now to find $W$ and $b$ which minimize this cost function, since $\mathcal{L}$ goes to zero as $\hat y$ gets closer to $y$, which is what we want.

It is time to now address the second point and talk about the gradient descent algorithm.

Since we cannot find the parameters $W$ and $b$ which minimize $J$ algebraically, we have to find a way to approximate them. This is what we can achieve with the gradient descent. First,let us define what the gradient of a function is.

***<ins>Definition:</ins>***  
*Let $f$ be function from $\mathbb{R}^p$ to $\mathbb{R}$. If $f$ is differentiable in the neighbourhood of a point $a$, the gradient of $f$ at point $a$ is*
$$
\nabla f(a) = \begin{pmatrix} \frac{\partial f}{\partial x_1}(a) \\\ \vdots \\\ \frac{\partial f}{\partial x_p}(a) \end{pmatrix}
$$ 

Intuitively, the gradient is a vector tangent to the hypersurface $\\{(x,f(x)), \quad x \in \mathbb{R}^p\\}$ which points in the direction of fastest increase of the function. Consequently, the vector $-\nabla f(a)$ points in the direction of fastest decrease of the function $f$. This motivates the following algorithm:

    Inputs: function f, step s, threshold e

    Initialize random point a
    Compute gradient(f)(a)
    While norm(gradient(f)(a)) > e
        a = a - e * gradient(f)(a)
        Compute gradient(f)(a) 

    Output: a

{{<admonition type=note title="Remark">}}
The algorithm above is a simplified, fixed step, version of the gradient descent. A proper one should feature a variable step size, most commonly computed through linear research, in order to take smaller and smaller steps as we approach the minimum. There are several ways to do so and some are commonly used in machine learning. This post will not address this problem, in order to keep things as simple as possible. Note that in the case of deep learning, this step is usually called the *learning rate* of the ANN.

For the same reason, this post does not address the numerous convergence issues that the gradient descent can have, depending on the geometry of the graph $\\{(x,f(x)), \quad x \in \mathbb{R}^p\\}$ For the purposes ahead, we'll admit that the gradient descent algorithm performs what it is supposed to on the given examples.
{{</admonition>}}    

### Perceptrons

Let us now dive in the main subject. If you have read any kind of explanation for neural networks, you have probably seen a picture very similar to the one below.

{{<image src="neuron.png" caption="Source: [Neuron Vectors by Vecteezy](https://www.vecteezy.com/free-vector/neuron)" title="A neuron diagram">}}

It is usually presented as an illustration for how artificial neurons work. The idea is to say something along the lines of *"the dendrites carry information to the nucleus where computations happen and the result is then transfered to the axon, and artificial neurons, or perceptrons, model that process by getting inputs from multiple sources and make a computation and output it"*. Well, for anyone with a bit of maths knowledge, having an object that takes several inputs and produces an output is something called a function of several variables...  As for understanding what truly happens inside a real neuron nucleus and how it all works on a bigger scale, well, let's just say we are not quite there yet...

So, after taking these claims for what they are, i.e. analogies at best, and ending my rant in favour of writing something more productive, we can say what artificial neurons/perceptrons are: a subclass of functions of several variables. Let us make this subclass explicit.

***<ins>Definition:</ins>***  
*A perceptron, or artificial neuron, with $p$ inputs is a function $f$ of the form*
$$
\begin{aligned}
f: \mathbb{R}^p & \longrightarrow \mathbb{R} \cr
             X  & \longmapsto g(W\cdot X + b)
\end{aligned}
$$
*where $W$ is a vector in $\mathbb{R}^p$ whose entries are called **weights**, $b$ is a real number called the **bias** and $g$ is an* activation function.

An activation function can't be just about anything (else our definition would be empty, as any function could be a perceptron). The problem here is that we cannot really have a satisfying mathematical definition of what an activation function is. The best we can do is say that it is a function from a certain list of allowed activation functions (which may change, as techniques evolve).

At the time of writing this article, there are two activation functions which are used a lot, and thus we'll limit ourselves to considering activation function to be one of these two functions:

1. The aforementioned logistic function
2. The ReLU (Rectifier Linear Unit) function, given by

$$
\forall x \in \mathbb{R}, \quad g(x) = \max_{x}(0,x).
$$

In the literature, you may see an artificial neuron being represented with the following kind of diagram

{{<style "text-align: center">}}
{{<image src="perceptron.svg" title="Perceptron diagram" caption="Source: MartinThoma, CC0, via Wikimedia Commons" width="60%">}}
{{</style>}}

where $\Sigma$ denotes the affine function with weights $w_i$ and $\varphi$ denotes the chosen activation function. If, at first, it seems like an unnecessary way to describe the type of function of the previous definition, the point of such a diagram will become clear in the next section. 

### Artificial Neural Networks

So far, we have only defined what a perceptron (or artificial neuron) is and, up to the change of possible activation function, we have a somewhat limited amount of problems that can be solved through the use of a single perceptron. Indeed, given an activation function $g$, being able to use the former method of gradient descent (if it works) in order to approximate the systemic information requires the relation between $X$ and $g^{-1}(y)$ to be linear.

We thus want to address this issue while still keeping a limited amount of possible activation function (else we just transfer the problem to somehow finding a good function in a plethora of choices, which is just not reasonable). Without getting into too much detail for now, notice that the ReLU activation function introduces non linearity. In particular, chaining perceptrons together will introduce more and more points where the function is not linear.  

So now, instead of having a single perceptron which outputs a result, we'll chain *hidden layers* of perceptrons together before feeding them into an *output layer* which can contain one several perceptron (depending on if we want $y$ to be one or multi dimensional).

It is possible, though counter productive in this case, to fully formalise things here. I'll first show a diagram explaining the structure of an artificial neural network and then I'll explain why traditional mathematical notations are not useful here.

{{<style "text-align: center">}}
{{<image src="neural_network.svg" title="Example of Artificial Neural Network" caption="Source: :Cburnett, CC BY-SA 3.0 via Wikimedia Commons" width="60%">}}
{{</style>}}

Above is an example of neural network with one *hidden layer*. The input layer is of size $p$ if the data is $p$-dimensional, the output is of size $k$ if the output variable $y$ is $k$ dimensional, and we can have as many hidden layers of any given sizes as we want. Each disk on the diagram above represents a perceptron and the arrows show which data is fed to which perceptron.

{{<admonition type="note" title="Remark" >}}
In the example above, every layer parameter is being fed to every neuron of the subsequent layer. Such layers are called *Dense*. This would be easy to describe as multivariate maps which are compositions of perceptrons along coordinates, but the fact that neural networks' layers don't necessarily have to be dense, and that we can feed inputs to whichever neuron we chose to, makes the above diagram representation particularly useful to understand what is going on. 
{{</admonition>}}

Let us now address the main question of this post

***<ins>Definition:</ins>***  
*A Neural Network is called* Deep *if it has two or more hidden layers.*

On a side note, both linear regression and logistic regression are neural networks without any hidden layers. They are called *shallow* neural networks.

I will now be giving a few useful notations. In all that follows, for ease of writing, I'll only consider ANNs with $p$-dimensional data $(x_1,...,x_p)$ and $1$-dimensional output $y$, which only have dense layers.

Let us consider an ANN with $k$ hidden layers. We are endowed with a data set $\mathbf{X}$ in $\mathcal{M}_{n,p}(\mathbb{R})$.

Let us denote by $p_i$ the number of neurons on the $i^\text{th}$ layer. By convention, let us fix $p_0 = p$.  
Let $W^{[i]}$ in $\mathcal{M}\_{p_{i-1},p_i}(\mathbb{R})$ be the matrix whose columns are the weights of the $i^\text{th}$ layer neurons. Let $b^{[i]}$ be the $\mathcal{M}\_{p_{i-1},p_i}(\mathbb{R})$ matrix whose lines are $(b^{[i]}_1,...,b^{[i]}\_{p_i})$, namely the biases of the perceptrons on the $i^\text{th}$ layer, let $\varphi^{[i]}$ be the activation function of the perceptrons of the $i^\text{th}$ layer.  
Let $Z^{[1]}$ be the following matrix 

$$
Z^{[1]} = \mathbf{X}  W^{[1]} + b^{[1]}
$$

and 

$$
A^{[1]}\_{i,j} = \varphi(Z^{[1]}\_{i,j}).
$$

From there on we can define, in a similar way, for any $i$ in $\\{2,...,k+1\\}$:
$$
Z^{[i]} = {A^{[i-1]}}  W^{[i]} + b^{[i]}
$$
and 
$$
A^{[i]}\_{l,m} = \varphi(Z^{[i]}\_{l,m}).
$$

With our notations, the outputs $\hat\mathbf{y}$ are given by $A^{[k+1]}$. Given a fixed data set $\mathbf{X}$ and neural network, these outputs depend on the weights and biases of each perceptron. Similarly as for the [baby cases](#baby-cases), we want our predicted values to be 'near' the actual values. As before, in order to understand exactly what is meant by 'near', we need a *cost* function.

### Cost Function

Similarly as for the [linear regression](#linear-regression) and the [logistic regression](#logistic-regression), we define a loss function and derive from it cost function which depends on the weights and biases. We wish to define a loss function which penalises instances where $\hat y$ is very different from $y$. In practice, we also want the loss function to be so that the gradient descent is effective. Having a convex cost function (seen as a function of $W$ and $b$) would ensure the gradient descent converge to a global minimum and not a purely local one. Let us give some examples.

#### Regression problem

In the case of a regression problem, where we want to predict a real valued number, one of the most common choice for the loss function is the following

$$
\mathcal{L}(\hat y, y) = (\hat y - y)^2
$$

which, with our notations gives rise to the following cost function, the *Mean Square Error*

$$
J(W,b) = \frac{1}{n} \sum_{i = 1}^n (\hat y_i - y_i)^2.
$$

#### Binary classification problem

If we wish to predict a binary outcome, in the form of a probability measure, the single output neuron could have an activation function like the logistic function. In that case, the loss function in the *cross entropy*

$$
\mathcal{L}(\hat y, y) = - y\log(\hat y) - (1 - y)\log(1 - \hat y)
$$

which, of course, is averaged out to make the *average cross entropy* function

$$
J(W,b) = \frac{-1}{n} \sum_{i = 1}^n   y_i\log(\hat y_i) + (1 - y_i)\log(1 - \hat y_i).
$$

#### Multiclass classification

Finally, let us give one last example, in case where the problem we want to solve is a multiclass classification problem, meaning we want to classify instances in three or more classes (such that no instance can belong to more than one class).

Let us give an example. Let us imagine, for instance that we are trying to predict who will win a race amongst 3 athletes. The output must then take its values in a set of three elements. For reason that I'll explain right after, let this set be $\\{(1,0,0), (0,1,0), (0,0,1)\\}$. Let us denote $v_i = (\delta_{i1}, \delta_{i2}, \delta_{i3})$ where $\delta_{ij}$ is the [Kronecker Delta](https://en.wikipedia.org/wiki/Kronecker_delta).

In that case, the output layer of our ANN should consist of 3 artificial neurons with no activation function, which means at this stage the neural network outputs an element of $\mathbb{R}^3$. We wish to convert into a probability measure, through the use of a well-chosen activation function. We want the final output to be something like
$$
\hat y = \begin{pmatrix}
\mathbb{P}(y=v_1 | X) \cr 
\mathbb{P}(y=v_2 | X) \cr 
\mathbb{P}(y=v_3 | X) 
\end{pmatrix}.
$$

In the case of a classification problem with $c$ classes (which are mutually exclusive), the current way to do this is to use the *softmax activation function*, which is defined as follows:

$$
\begin{aligned}
\varphi : \mathbb{R}^c & \longrightarrow  \mathbb{R}^c \cr
          \begin{pmatrix}x_1\\\ \vdots \\\x_c\end{pmatrix}& \longmapsto \frac{1}{\sum_{i=1}^c e^{x_i}}\begin{pmatrix}e^{x_1}\\\ \vdots \\\ e^{x_c}\end{pmatrix}
\end{aligned}.
$$

If you are familiar with statistical mechanics of thermodynamics, you might recognize a Gibbs measure. Else you can just think of this as a natural generalisation of the logistic function in the case of non-binary classifications. As such, the same loss function is used, namely *cross entropy*

$$
\mathcal{L}(\hat y, y) = - \sum_{i = 1}^c y_i \log(\hat y_i)
$$

which we can, again average the loss over the points of the data set to get the cost function

$$
J(W,b) = \frac{-1}{n} \sum_{i = 1}^n \sum_{j=1}^c y_\{i,j\} \log(\hat y_\{i,j\}).
$$

Now that we have covered three usual cost functions, let us move on to a short section, briefly explaining the 'learning' process of the algorithm.

### Learning

I won't give too much detail about the actual "learning" process, as I feel the main idea has already been explained: we basically compute the cost function with a random set of parameters and then apply the gradient descent algorithm. I'll however give some details on the jargon that is usually employed.

#### Forward propagation

The forward propagation is, basically, the step where the outputs are computed. It is called forward propagation because, if you look at the artificial neural network, you compute the different matrices $Z^{[i]}$ and $A^{[i]}$, going through the hidden layers from left to right. On a purely theoretical standpoint, there is no mystery there, but on a technical, implementation point of view, there are other important operations that need to be performed during this phase (mostly caching important values).

#### Backpropagation

Now that all the outputs and, consequently, the cost function, have been computed, it is time to do some parameter optimisation using the gradient descent. 

In order to do that, we need to compute the gradient $\nabla J$, where $J$ is seen as a function of $W$ and $b$. To that end, we compute things 'backwards' in the ANN. Let me explain on a simple example.

Say we have two real valued functions $f$ and $g$ and we want to minimise the function $g \circ f$. Let us make a silly little diagram to understand:

$$
x \xmapsto{f} f(x) \xmapsto{g} g \circ f (x)
$$

So, if I want to first compute $g \circ f (x)$, knowing only $x$, I need to compute 'forward', as told in the previous section.

Now if I want to compute the derivative of $g \circ f$ in $x$, the chain rule tells me that

$$
(g \circ f)^\prime(x) = g^\prime(f(x)) f^\prime(x)
$$

and thus, having already computed the value of $f(x)$, we can now compute the different derivatives, going from right to left in through the layers.

If you are not convinced, a good calculus exercise is to compute the partial derivatives of a cost function on a simple example of DNN for yourself.

#### Epoch

An *epoch* is simply a forward propagation, followed by a backpropagation and finally, an update of the weights and biases using the computed gradient of the cost function. In other words, it is an iteration of the gradient descent algorithm. The more epochs we will run, the more the parameters will be optimised, so that the cost function will be as small as possible, at least on the training set...

### The Universal Approximation Theorem

All this is nice but, even assuming that we can minimise a well chosen cost function which somehow encodes well the distance between the systemic information and its approximation, how can we know that the approximation has any chance of being any good? It could very well be that the systemic information is a function that, despite the data scientist best efforts, *cannot* be approximated by neural networks with the aforementioned activation functions, regardless of their depths and number of neurons, right?

As it turns out, there are purely theoretical results which states that any function can be approximated by a neural network. Given the examples given in this post, we will cite the 2017 result by [Zhou Lu et al.](https://proceedings.neurips.cc/paper/2017/file/32cbf687880eb1674a07bf717761dd3a-Paper.pdf).

***<ins>Theorem:</ins>***  
*For any Lebesgue-integrable function $f:\mathbb {R}^{p}\rightarrow \mathbb {R}$ and any $\epsilon >0$, there exists a fully-connected ReLU network with width (i.e. max number of neurons on layers) less than $p+5$ such that the function $\hat f$ given by this network satisfies*

$$
\int_{x \in \mathbb{R}^p} |f(x) - \hat f(x) | dx < \epsilon
$$

In other words, functions given by arbitrary depth, ReLU activated, neural networks are dense in Lebesgue-integrable function for the $L^1$ distance, which justifies, at least theoretically, the use of deep neural network to approximated systemic information functions.

## Conclusion

There is obviously a lot more to know about the theory of artificial neural networks. Whether on the purely theoretical side, which was barely grazed here, or on the actual implementations methods and issues which I have not even mentioned.

In any, case I hope this blog post gave you an understanding of some of the  underlying maths principles which allow deep neural networks to work. Now you just have to start coding...
