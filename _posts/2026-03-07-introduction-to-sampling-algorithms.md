---
layout: distill
title: Introduction to Sampling Algorithms
description: From a uniform random number generator to inverse transform sampling, rejection sampling, and importance sampling — building intuition for how computers draw samples from arbitrary distributions
tags: sampling, monte carlo, statistics, probability
giscus_comments: true
date: 2026-03-07
featured: true

authors:
  - name: Ricardo Huaman
    url: "https://rotvie.github.io/"
    affiliations:
      name: Independent

bibliography: introduction-to-sampling-algorithms.bib
---

## Introduction to Sampling Algorithms

If you're reading this blogpost, you're probably accessing it through a computer of some sort. Computers are fundamentally deterministic machines. Yet many algorithms rely on randomness. In practice, computers simulate randomness using pseudorandom number generators (PRNGs) — deterministic algorithms that produce sequences of numbers that behave like random samples <d-cite key="Devroye_1986"></d-cite>.

With this in mind, let's assume we already have a computer that can generate samples from a uniform distribution:

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/introduction-to-sampling-algorithms/sampling-uniform-blackbox.png" title="Uniform distribution primitive" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

This Probability Density Function (PDF) gives us numbers from 0 to 1 with equal probability — essentially a random number generator. This is our **primitive**, and the goal is to use it to generate samples from other, more complex PDFs like the Gaussian distribution.

To be precise: the problem is to take this primitive and produce samples whose empirical distribution approximates a target PDF. This process is called **sampling**.

## Why sampling is hard?

Sampling from a probability density function is not as easy as it may seem. Suppose you have the following Gaussian and your task is to click along the x-axis to recreate its shape:

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/introduction-to-sampling-algorithms/sampling-why-hard.png" title="Why sampling is hard" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

Intuition tells us to select samples from the middle more frequently than from the tails. With some effort, we could get something that roughly resembles a Gaussian. But the real question is: **how do we give this intuition to a computer?**

## Inverse Transform Sampling

This is the most straightforward sampling method. Let's start by defining a few terms.

### Cumulative Distribution Function (CDF)

The cumulative distribution function $$F_X(x)$$ of a real-valued random variable $$X$$, evaluated at $$x$$, is the probability that $$X$$ will take a value less than or equal to $$x$$:

$$
F_X(x) = \int_{-\infty}^{x} f_X(t) \, dt
$$

where $$f_X(t)$$ is the PDF of $$X$$, $$x$$ is the upper limit of integration, and $$t$$ is the variable of integration spanning from $$-\infty$$ to $$x$$.

As a concrete example, consider the exponential distribution:

$$
f_X(x) = \lambda e^{-\lambda x}, \quad x \geq 0, \quad \lambda > 0
$$

which looks like this:

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/introduction-to-sampling-algorithms/sampling-exponential-pdf.png" title="Exponential PDF" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

Applying the CDF definition:

Since the exponential PDF is zero for $$x < 0$$, the integral reduces to:

$$
F_X(x) = \int_{0}^{x} \lambda e^{-\lambda t} \, dt = \left[-e^{-\lambda t}\right]_{0}^{x} = 1 - e^{-\lambda x}, \quad x \geq 0
$$

Plotting this CDF:

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/introduction-to-sampling-algorithms/sampling-exponential-cdf.png" title="Exponential CDF" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

Now think of the y-axis of the CDF curve, which ranges from 0 to 1. If you pick a random point on this y-axis, draw a horizontal line until you hit the CDF curve, and then drop a vertical line down to the x-axis — you've just performed inverse transform sampling.

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/introduction-to-sampling-algorithms/sampling-geometric-construction.png" title="Geometric construction of inverse transform sampling" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

The key insight: a CDF is steeper where the probability density is higher. This means a larger chunk of the y-axis maps to the more likely x-values. Picking uniformly on the y-axis naturally produces the correct clustering on the x-axis.

### Inverse Cumulative Distribution Function

With this insight, we can derive a sampling formula by inverting the CDF. Starting from the exponential CDF:

$$
y = 1 - e^{-\lambda x}
$$

Solving for $$x$$:

$$
\begin{align}
e^{-\lambda x} &= 1 - y \\
-\lambda x &= \ln(1 - y) \\
x &= -\frac{1}{\lambda} \ln(1 - y)
\end{align}
$$

Since $$y \sim \text{Uniform}(0, 1)$$, we note that subtracting a uniform variable from 1 preserves the uniform distribution ($$1 - y \sim \text{Uniform}(0,1)$$), so we can simplify to:

$$
F_X^{-1}(u) = -\frac{1}{\lambda} \ln(u), \quad u \sim \text{Uniform}(0, 1)
$$

We can now plug uniform random values from our primitive into this formula and watch the exponential distribution emerge:

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/introduction-to-sampling-algorithms/sampling-its-animation.gif" title="Inverse transform sampling animation" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

## Rejection Sampling

So far so good — but obtaining the inverse CDF is not always tractable. Many distributions don't have a closed-form inverse, which makes inverse transform sampling impractical. Let's look at a method that sidesteps this limitation entirely.

Rejection sampling requires two ingredients: (1) a target distribution $p(x)$ that we want to sample from, and (2) a proposal distribution $q(x)$ that we already know how to sample from and that roughly captures the shape of the target <d-cite key="VonNeumann_1951"></d-cite>. We then define a scaled proposal $$cq(x)$$, where $$c$$ is a constant chosen such that

$$
cq(x) \geq p(x), \quad \forall x
$$

As a concrete example, let's define a target $$p(x) = e^{-x^2/2} \cdot (\sin^2(6+x) + 3 \cdot \cos^2(x) \cdot \sin^2(4x) + 1)$$ on the domain $$[-3, 3]$$. Note that this function does not need to be normalized — rejection sampling only requires evaluating the target up to a constant factor. For the proposal, we'll use a uniform distribution on the same domain with $$c = 25$$:

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/introduction-to-sampling-algorithms/sampling-proposal-envelopes.png" title="Proposal and target distributions" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

Now suppose we sample $$x = -1$$ from the proposal. We need to decide whether to accept or reject this sample. To build intuition, let's rotate the plot clockwise and look at the vertical line from the x-axis through the target value up to the proposal envelope:

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/introduction-to-sampling-algorithms/sampling-rotated-view.png" title="Rotated view of rejection sampling" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

Along this line, we sample a uniform number $$u$$. If $$u$$ lands beyond the target value, we reject the sample; if it lands within, we accept. This is the core mechanism — points in high-density regions of $$p(x)$$ are accepted more often, naturally reconstructing the target shape.

Formalizing this:

**Algorithm: Rejection sampling**

$$
\begin{array}{l}
\textbf{Input: } \text{target } p(x), \text{ proposal } q(x), \text{ constant } c \text{ such that } cq(x) \geq p(x) \; \forall x \\
\textbf{Output: } \text{a sample from } p(x) \\
\\
\textbf{repeat:} \\
\quad x \sim q(x) \\
\quad u \sim \text{Uniform}(0, 1) \\
\quad \textbf{if } u \leq \dfrac{p(x)}{c \cdot q(x)}: \\
\quad\quad \textbf{return } x \\
\end{array}
$$

Notice that the choice of proposal matters. If we use a distribution that more tightly envelops the target — say, a scaled Gaussian instead of a uniform — we increase the acceptance rate and converge faster:

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/introduction-to-sampling-algorithms/sampling-rejection-animation.gif" title="Rejection sampling animation" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

## Importance Sampling

This will be our last method for this blogpost. The name is somewhat misleading — importance sampling is not really a sampling algorithm in the same sense as the previous two. It's a **variance reduction technique** for estimating expectations. Still, it's foundational to many applications and connects directly to the methods we've covered, so it's worth understanding here.

### Expected Value

Before we get to importance sampling itself, we need to establish the tools it builds on. The first is the expected value.

The expected value of a discrete random variable $$X$$ is defined as:

$$
E[X] = \sum_{i} x_i \cdot P(X = x_i)
$$

For a continuous random variable with PDF $$f_X(x)$$:

$$
E[X] = \int_{-\infty}^{\infty} x \cdot f_X(x) \, dx
$$

Intuitively, the expected value is a weighted average: each possible value of $$X$$ is weighted by how likely it is to occur. Values in high-density regions of the PDF contribute more to the sum than values in the tails.

For example, if $$X \sim \text{Uniform}(0, 1)$$, the expected value is:

$$
E[X] = \int_{0}^{1} x \cdot 1 \, dx = \frac{1}{2}
$$

No surprises — the average of a uniform distribution over $$[0, 1]$$ is $$0.5$$.

### Law of the Unconscious Statistician (LOTUS)

Now suppose we don't care about $$X$$ itself, but about some function of it — say $$g(X)$$. What is $$E[g(X)]$$?

The naive approach would be: figure out the distribution of the new random variable $$Y = g(X)$$, then compute $$E[Y]$$ using that distribution. This can be painful — deriving the PDF of a transformed variable requires change-of-variables techniques and Jacobians.

LOTUS gives us a shortcut. It says we can compute the expected value of $$g(X)$$ directly using the original PDF of $$X$$:

$$
E[g(X)] = \int_{-\infty}^{\infty} g(x) \cdot f_X(x) \, dx
$$

No need to derive the distribution of $$g(X)$$ — we just integrate $$g(x)$$ weighted by the density of $$X$$. The name is somewhat tongue-in-cheek: it's the law that statisticians use "unconsciously" because it feels so natural.

This result is what connects expected values to sampling. If we can draw samples $$x_1, x_2, \dots, x_N$$ from $$f_X(x)$$, then we can approximate this integral by averaging $$g$$ over the samples — which is exactly what Monte Carlo methods do.

### Monte Carlo Methods

Monte Carlo estimation is the idea that we can approximate an expected value by drawing samples and averaging <d-cite key="Robert_Casella_2004"></d-cite>. Given $$N$$ samples $$x_1, x_2, \dots, x_N$$ drawn from $$f_X(x)$$, the Monte Carlo estimator is:

$$
E[g(X)] \approx \hat{\mu}_N = \frac{1}{N} \sum_{i=1}^{N} g(x_i), \quad x_i \sim f_X(x)
$$

Notice that $$f_X(x)$$ doesn't appear explicitly in the sum. This is because the sampling process already accounts for it — since each $$x_i$$ is drawn from $$f_X$$, values in high-density regions appear more frequently in the sample, and values in low-density regions appear rarely. The weighting by $$f_X(x)$$ is built into the distribution of the samples themselves.

By the **Law of Large Numbers**, this estimator converges to the true expected value as $$N \to \infty$$. And by the **Central Limit Theorem**, for large $$N$$ the estimator is approximately normally distributed:

$$
\hat{\mu}_N \approx \mathcal{N}\left(E[g(X)], \; \frac{\text{Var}(g(X))}{N}\right)
$$

The variance of the estimator shrinks as $$\frac{1}{N}$$ — so to halve the error, we need four times as many samples. This is where importance sampling enters the picture.

### The Importance Sampling Trick

Monte Carlo works great when we can sample from the target distribution $$p(x)$$. But what if sampling from $$p$$ is difficult, or what if naive sampling produces estimates with high variance? Importance sampling addresses both problems <d-cite key="Kahn_Marshall_1953"></d-cite>.

The idea behind importance sampling starts with a simple algebraic trick. Take the expectation we want to compute:

$$
E_p[g(X)] = \int g(x) \cdot p(x) \, dx
$$

Now multiply and divide by a proposal distribution $$q(x)$$ that we *can* sample from, provided that $$q(x) > 0$$ wherever $$p(x) \neq 0$$ (the proposal must have support everywhere the target does):

$$
E_p[g(X)] = \int g(x) \cdot \frac{p(x)}{q(x)} \cdot q(x) \, dx
$$

Look at what happened — this is now an expectation under $$q$$:

$$
E_p[g(X)] = E_q\left[g(X) \cdot \frac{p(X)}{q(X)}\right]
$$

We've rewritten an expectation under a distribution we can't sample from ($$p$$) as an expectation under one we can ($$q$$). The cost is that we need to correct each sample with the ratio $$\frac{p(x)}{q(x)}$$, called the **importance weight**.

The Monte Carlo estimator becomes:

$$
E_p[g(X)] \approx \frac{1}{N} \sum_{i=1}^{N} g(x_i) \cdot \frac{p(x_i)}{q(x_i)}, \quad x_i \sim q(x)
$$

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/introduction-to-sampling-algorithms/sampling-is-trick.png" title="Importance sampling trick" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

### Building Intuition: A Rigged Die

To see why this works, consider a concrete example. Imagine a rigged die where face 6 comes up half the time, while faces 1 through 5 each appear with probability $$\frac{1}{10}$$. We want to compute the expected value under this rigged die, but all we have available is a fair die.

If we roll the fair die $$N$$ times, each face shows up roughly $$\frac{1}{6}$$ of the time. But under the rigged distribution, face 6 should appear in $$\frac{1}{2}$$ of our rolls — the frequencies are wrong.

The importance weight corrects this mismatch. For face 6, the weight is $$\frac{P_{\text{rigged}}(6)}{P_{\text{fair}}(6)} = \frac{1/2}{1/6} = 3$$ — each observation of a 6 contributes its value multiplied by a weight of 3. For face 1, the weight is $$\frac{1/10}{1/6} = 0.6$$ — we downweight it because the fair die overrepresents it relative to the rigged distribution.

The importance weights compensate for the mismatch between the distribution we're sampling from and the distribution we actually care about.

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/introduction-to-sampling-algorithms/sampling-rigged-die.png" title="Rigged die importance weights" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

### Why Not Just Use Any $$q$$?

Mathematically, any $$q(x) > 0$$ wherever $$p(x) > 0$$ will give a correct estimator. But the choice of $$q$$ dramatically affects the **variance**.

If $$q$$ is very different from $$p$$, some samples will have enormous importance weights while most will have tiny ones. A few samples dominate the sum, and the estimate becomes noisy. In the extreme, you might need millions of samples to get a reasonable approximation.

The optimal proposal is:

$$
q^*(x) \propto |g(x)| \cdot p(x)
$$

This proposal minimizes the variance of the importance sampling estimator. It concentrates samples where the integrand $$g(x) \cdot p(x)$$ is large — the "important" regions. This is where the name comes from: we sample more from the parts of the space that matter most to the integral.

In practice, we can't usually compute $$q^*$$ exactly (if we could, we wouldn't need Monte Carlo in the first place). But even a rough approximation to it can reduce variance by orders of magnitude compared to naive sampling — and that's what makes importance sampling so powerful.

To see this in action, suppose we want to estimate $$E_p[X^2]$$ where $$p(x) = \mathcal{N}(2, 0.5^2)$$. We compare two proposals: a good one centered near the target ($$q_{good} = \mathcal{N}(2, 0.8^2)$$) and a bad one far away ($$q_{bad} = \mathcal{N}(-1, 2^2)$$). Watch how the good proposal converges smoothly to the true value, while the bad proposal is erratic — dominated by rare samples with extreme importance weights:

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/introduction-to-sampling-algorithms/sampling-convergence.png" title="Good vs bad proposal convergence" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/introduction-to-sampling-algorithms/sampling-is-animation.gif" title="Importance sampling animation" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

## Summary

We started with a single primitive — a uniform random number generator — and built three increasingly flexible methods to sample from arbitrary distributions. Each method makes a different tradeoff:

| Method | What it needs | Strength | Mode of failure |
|---|---|---|---|
| Inverse Transform Sampling | Closed-form $$F_X^{-1}$$ | Exact samples, no waste | Most distributions don't have an invertible CDF |
| Rejection Sampling | A proposal $$cq(x) \geq p(x)$$ | Works for any evaluable target | Poor proposal choice leads to massive rejection rates — most samples are thrown away |
| Importance Sampling | A proposal $$q(x)$$ with overlapping support | Doesn't discard samples; reweights instead | Bad $$q$$ causes a few samples to dominate via extreme weights, making the estimate unreliable |

A pattern emerges across all three: **the quality of your results depends on how well you understand the shape of the distribution you're targeting.** Inverse transform sampling requires you to know the CDF so well that you can invert it analytically. Rejection sampling requires a proposal that tightly envelops the target — too loose and you waste most of your computation on rejected points. Importance sampling requires a proposal that places mass where the integrand matters — too far off and a handful of samples with extreme weights will dominate your estimate.

This is not the end of the story. All three methods share a fundamental limitation: they struggle in high-dimensional spaces. As the number of dimensions grows, the volume of space increases exponentially, and most of the probability mass concentrates in thin regions of the space. The probability of generating a useful sample drops just as fast. Rejection sampling acceptance rates collapse, and importance weights become increasingly degenerate.

This is where **Markov Chain Monte Carlo (MCMC)** methods come in <d-cite key="Robert_Casella_2004"></d-cite>. Instead of generating independent samples and hoping they land in the right place, MCMC constructs a Markov chain whose stationary distribution is the target distribution, generating correlated samples that gradually explore the space by taking small, informed steps. But that's a topic for another post.
