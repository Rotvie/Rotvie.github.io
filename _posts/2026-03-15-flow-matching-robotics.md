---
layout: distill
title: Introduction to Flow Matching with Robotics
description: Learning to generate collision-free robot configurations by transporting noise to data — from the math behind flow matching to a working implementation on a 2-link arm
tags: flow matching, generative models, robotics, configuration space, sampling
giscus_comments: false
date: 2026-03-15
featured: false

authors:
  - name: Ricardo Huaman
    url: "https://rotvie.github.io/"
    affiliations:
      name: Independent

bibliography: flow-matching-robotics.bib
---

## Introduction

Most tutorials on flow matching demonstrate it on toy 2D distributions — moons, circles, spirals. This post takes a different route. We'll use flow matching to solve a problem from robotics: **generating collision-free configurations for a robot arm**.

The idea is simple. A 2-link planar arm operates in a workspace cluttered with obstacles. Some joint angle combinations cause the arm to collide; others are safe. The set of all safe configurations forms a complex, non-convex region in **configuration space** (C-space). We want a generative model that produces samples exclusively from this free region — without ever having to solve the collision geometry at test time.

Flow matching gives us exactly the tool for this. It learns a velocity field that transports Gaussian noise into the target distribution, and at inference we just integrate an ODE. No rejection sampling, no iterative denoising with hundreds of steps — just a clean forward pass through a learned vector field.

This post covers the theory behind flow matching, builds intuition for why the objective simplifies the way it does, and walks through a complete implementation — from data generation to evaluation.

## The Problem: Sampling from Configuration Space

A planar robot arm has two revolute joints with angles $$(\theta_1, \theta_2) \in [-\pi, \pi]^2$$ and link lengths $$L_1 = 1.5$$, $$L_2 = 1.0$$. The workspace contains three circular obstacles.

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/flow-matching-robotics/fm-workspace-cspace.png" title="Workspace and C-space" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Left: the 2-link arm in its workspace with three circular obstacles. Right: the corresponding configuration space. Green dots are collision-free configurations; gray regions are obstacle zones. The free C-space has a complex, non-convex structure that makes naive sampling inefficient.
</div>

The left panel shows the physical arm and obstacles. The right panel shows what happens when we map the obstacle boundaries into joint angle space: the free region fragments into irregular channels separated by curved obstacle zones. If we sampled $$(\theta_1, \theta_2)$$ uniformly at random, roughly 35% of configurations would collide — a significant waste if we need valid configurations for motion planning or policy learning.

The goal is to learn a generative model whose samples concentrate in the free (white) regions of C-space.

## Flow Matching: The Core Idea

Flow matching constructs a time-dependent velocity field $$v_\theta(x_t, t)$$ that defines an ordinary differential equation (ODE):

$$\frac{dx_t}{dt} = v_\theta(x_t, t), \qquad t \in [0, 1]$$

At $$t = 0$$, we start from a simple distribution we can sample from — a standard Gaussian. At $$t = 1$$, the ODE should have transported those samples into the target distribution — our collision-free C-space. The velocity field $$v_\theta$$ is a neural network trained to make this transport happen.

The question is: what should the network learn to predict?

### The Interpolation Trick

The key idea in flow matching is to define straight-line paths between noise and data <d-cite key="Lipman_2023"></d-cite>. Given a noise sample $$x_0 \sim \mathcal{N}(0, I)$$ and a data sample $$x_1$$ from the target, the interpolation is:

$$x_t = (1 - t)\, x_0 + t\, x_1$$

Taking the time derivative:

$$\frac{dx_t}{dt} = x_1 - x_0$$

This is the **conditional velocity** — the constant-speed straight line from noise to data. At every point in time, the "true" velocity that moves a specific noise sample toward a specific data point is just $$x_1 - x_0$$.

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/flow-matching-robotics/fm-conditional-paths.png" title="Conditional interpolation paths" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Conditional interpolation paths at three timesteps. At \(t = 0\), samples are Gaussian noise (left). At \(t = 1\), they match the target data distribution in C-space (right). The middle panel shows \(t = 0.5\) with pink lines connecting paired noise-data samples, illustrating the straight-line transport.
</div>

### From Flow Matching to Conditional Flow Matching

If we wanted to learn the velocity field directly, we'd need to minimize the **Flow Matching objective**:

$$\mathcal{L}_{\text{FM}} = \mathbb{E}_{t,\, p_t(x_t)} \left[ \| v_\theta(x_t, t) - u_t(x_t) \|^2 \right]$$

where $$u_t(x_t)$$ is the marginal velocity field — the aggregate velocity at position $$x_t$$ and time $$t$$, averaged over all possible noise-data pairings that could pass through that point. The problem is that we don't have access to either $$p_t$$ (the marginal distribution at time $$t$$) or $$u_t$$ (the marginal velocity). Computing them would require integrating over all possible pairings — intractable in practice.

The breakthrough in <d-cite key="Lipman_2023"></d-cite> is showing that we can instead optimize the **Conditional Flow Matching (CFM) objective**:

$$\mathcal{L}_{\text{CFM}} = \mathbb{E}_{t,\, q(z),\, p_t(x_t | z)} \left[ \| v_\theta(x_t, t) - u_t(x_t | z) \|^2 \right]$$

where $$z = (x_0, x_1)$$ is a specific noise-data pair, and $$u_t (x_t \| z) = x_1 - x_0$$ is the conditional velocity for that pair.

The key result: the gradients of both objectives are identical.

$$\nabla_\theta \mathcal{L}_{\text{FM}}(\theta) = \nabla_\theta \mathcal{L}_{\text{CFM}}(\theta)$$

This means we never need to compute the intractable marginal quantities. We just sample pairs, compute straight-line velocities, and regress. The derivation uses the Fubini-Tonelli theorem to swap the order of integration and a marginalization argument — the details are in the original paper, but the practical upshot is that training reduces to a simple regression problem.

### Comparison with Other Generative Approaches

Flow matching is part of a broader family of methods that learn to transform noise into data. It helps to see where it sits relative to two other approaches:

**Score Matching** learns the gradient of the log-density, $$\nabla_x \log p_t(x)$$. Sampling then requires running Langevin dynamics — an iterative process that mixes gradient steps with noise injections. The stochastic sampling can be slow, and the score function can be difficult to estimate accurately in low-density regions.

**Denoising Diffusion (DDPM)** defines a forward process that gradually corrupts data with Gaussian noise over many timesteps, then trains a network to reverse each step <d-cite key="Ho_2020"></d-cite>. Sampling involves hundreds or thousands of sequential denoising steps, each involving a forward pass through the network.

**Flow Matching** sidesteps both issues. Instead of learning a score and running stochastic dynamics, or learning to reverse a multi-step noising process, it directly learns a velocity field that defines a deterministic ODE from noise to data. The interpolation paths are straight lines, the training target is a simple difference $$x_1 - x_0$$, and sampling requires integrating a single ODE — which can be done in as few as 10–100 Euler steps. The result is a method that is simpler to implement, faster to sample from, and often produces higher-quality results <d-cite key="Lipman_2023"></d-cite>.

### The Final Training Objective

Putting it all together, the training procedure samples a noise point $$x_0 \sim \mathcal{N}(0, I)$$, a data point $$x_1$$ from the training set, and a time $$t \sim \mathcal{U}[0, 1]$$. It constructs the interpolant $$x_t = (1 - t)\, x_0 + t\, x_1$$ and regresses the network output against the target velocity:

$$\mathcal{L} = \| v_\theta(x_t, t) - (x_1 - x_0) \|^2$$

That's it. No ELBO, no adversarial loss, no score estimation. Just an MSE regression on straight-line velocities.

## Code Implementation

The full implementation is in the companion notebook. Here I'll walk through the key components.

### The Model

The velocity field $$v_\theta(x_t, t)$$ is parameterized by an MLP with sinusoidal time embeddings — the same idea as positional encodings in transformers. Time $$t$$ is mapped to a high-dimensional feature vector via sinusoidal functions, projected, and added to the data projection before passing through a stack of feedforward blocks:

{% highlight python %}
class FlowMLP(nn.Module):
def **init**(self, dim*data=2, layers=5, channels=512, dim_t=512):
super().**init**()
self.dim_t = dim_t
self.in_proj = nn.Linear(dim_data, channels)
self.t_proj = nn.Linear(dim_t, channels)
self.blocks = nn.Sequential(\*[Block(channels) for * in range(layers)])
self.out_proj = nn.Linear(channels, dim_data)

    def sinusoidal_embedding(self, t, max_positions=10000):
        t = t * max_positions
        half = self.dim_t // 2
        freqs = torch.arange(half, device=t.device).float()
        freqs = freqs.mul(-math.log(max_positions) / (half - 1)).exp()
        args = t[:, None] * freqs[None, :]
        emb = torch.cat([args.sin(), args.cos()], dim=1)
        return emb

    def forward(self, x, t):
        x = self.in_proj(x)
        t = self.sinusoidal_embedding(t)
        t = self.t_proj(t)
        x = x + t  # additive conditioning
        x = self.blocks(x)
        return self.out_proj(x)

{% endhighlight %}

The additive conditioning — projecting $$x$$ and $$t$$ separately then summing — is simple and works well for low-dimensional problems. For higher-dimensional data you'd typically use adaptive normalization (FiLM layers), but for 2D configuration space this is more than sufficient.

### Training

Each training step implements the CFM objective directly:

{% highlight python %}
for step in range(100_000):
x1 = data[torch.randint(len(data), (batch_size,))] # sample data
x0 = torch.randn_like(x1) # sample noise
t = torch.rand(batch_size, device=device) # sample time

    xt = (1 - t[:, None]) * x0 + t[:, None] * x1          # interpolate
    target = x1 - x0                                       # target velocity

    pred = model(xt, t)
    loss = ((pred - target) ** 2).mean()

    optim.zero_grad()
    loss.backward()
    optim.step()

{% endhighlight %}

Five lines of logic. Sample data, sample noise, sample time, build interpolant, regress on the velocity. The training data consists of 5,000 collision-free configurations obtained by rejection sampling — the only part of the pipeline that requires explicit collision checking.

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/flow-matching-robotics/fm-training-loss.png" title="Training loss" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Training loss (smoothed MSE) over 100k steps with AdamW at learning rate \(10^{-4}\). The loss converges quickly and stabilizes, indicating the network has learned the velocity field.
</div>

### Sampling

At inference, we draw Gaussian noise and integrate the learned ODE forward using Euler steps:

{% highlight python %}
xt = torch.randn(n_samples, 2, device=device) # start from noise
dt = 1.0 / euler_steps

for t_val in torch.linspace(0, 1, euler_steps):
vel = model(xt, t_val.expand(n_samples))
xt = xt + dt \* vel # Euler step
{% endhighlight %}

No denoising schedule, no noise injection, no classifier guidance. Just a forward ODE integration — 100 Euler steps, each a single forward pass through the MLP.

## Results

### ODE Transport: From Noise to Free C-Space

The snapshots below show 2,000 particles being transported from Gaussian noise ($$t = 0$$) to the target distribution ($$t = 1$$). By $$t = 0.25$$, the particles have already begun organizing into the channel structure of the free C-space. By $$t = 1$$, they closely match the target with only a few particles landing in obstacle regions.

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/flow-matching-robotics/fm-ode-snapshots.png" title="ODE integration snapshots" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    ODE integration snapshots at \(t = 0.00, 0.10, 0.25, 0.50, 0.75, 1.00\). Particles (black) start as Gaussian noise and are transported into the free C-space. Red dots at \(t = 1\) indicate the few samples that land in collision regions.
</div>

The full animation shows this transport with individual particle traces:

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/flow-matching-robotics/fm-sampling-animation.gif" title="Sampling animation" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Animated ODE sampling: 2,000 particles transported from Gaussian noise to the collision-free C-space over 100 Euler steps.
</div>

### The Learned Velocity Field

We can visualize what the network actually learned by evaluating $$v_\theta(x, t)$$ on a grid at different timesteps. This reveals the structure of the transport:

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/flow-matching-robotics/fm-velocity-field.png" title="Learned velocity field" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Learned velocity field \(v_\theta(x, t)\) at \(t = 0.00, 0.25, 0.50, 0.75\). Early timesteps show broad sweeping vectors pushing noise toward data modes. Later timesteps show finer corrections that steer particles into the narrow free-space channels.
</div>

At $$t = 0$$, the field shows large, directional vectors — the network is making coarse decisions about which mode each particle should head toward. By $$t = 0.25$$, the vectors in obstacle regions have nearly vanished (particles have already left those areas), while vectors in the free channels point inward to keep particles on track. The later timesteps show increasingly local, corrective behavior.

### Evaluation: Do Generated Configurations Actually Work?

The real test: forward-kinematics the generated joint angles and check whether the resulting arm poses collide with obstacles.

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/flow-matching-robotics/fm-generated-evaluation.png" title="Generated sample evaluation" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Left: generated samples in C-space — red dots are collision-free, green dots collide. Center: valid arm configurations rendered in the workspace. Right: the 77 colliding configurations. The model achieves a 96.2% collision-free rate.
</div>

Out of 2,000 generated samples, 1,923 are collision-free — a **96.2% success rate**. The few failures cluster near the boundaries of obstacle regions in C-space, which is expected: the boundary is where the density drops sharply, and the model has the least data to learn from.

### Baseline: Why Not Just Sample Randomly?

If we sampled $$(\theta_1, \theta_2)$$ uniformly at random across $$[-\pi, \pi]^2$$, the collision-free rate would be about 65.6% — determined by the fraction of C-space occupied by obstacles. Flow matching improves this to 96.2%, a substantial gain that would compound in downstream applications where every invalid sample requires an expensive re-planning step.

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/flow-matching-robotics/fm-baseline-comparison.png" title="Baseline comparison" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Collision-free rate: random uniform sampling (65.6%) versus flow matching (96.2%).
</div>

## Summary

Flow matching provides an elegant approach to generative modeling that is both theoretically clean and practical to implement. The core insight — that we can train on straight-line conditional velocities while recovering the correct marginal flow — turns a seemingly intractable transport problem into a simple regression.

Applied to robotics, this translates into a generator that produces valid arm configurations without explicit collision checking at test time. The model learns the shape of the free configuration space from data alone, achieving 96.2% collision-free accuracy compared to 65.6% for random sampling.

The implementation is minimal: an MLP with sinusoidal time conditioning, trained for 100k steps on an MSE loss. Sampling is a 100-step Euler integration — deterministic, fast, and parallelizable.

For higher-dimensional configuration spaces or more complex obstacle geometries, the same framework applies — you'd just need a larger network, more training data, and potentially a more expressive architecture (U-Nets or transformers instead of an MLP). The underlying math stays the same.

## Glossary

**Configuration Space (C-space):** The space of all possible joint angle combinations for a robot. For a 2-link arm, C-space is $$(\theta_1, \theta_2) \in [-\pi, \pi]^2$$. Obstacles in the workspace carve out forbidden regions in C-space.

**Flow Matching:** A generative modeling framework that learns a velocity field defining an ODE from a simple source distribution (Gaussian noise) to a target data distribution. Training uses straight-line conditional interpolation paths.

**Conditional Flow Matching (CFM):** The tractable version of the flow matching objective. Instead of computing intractable marginal quantities, it conditions on specific noise-data pairs and regresses on their straight-line velocity $$x_1 - x_0$$.

**Euler Integration:** A first-order numerical method for solving ODEs. Given $$\frac{dx}{dt} = v(x, t)$$, the update is $$x_{t+\Delta t} = x_t + \Delta t \cdot v(x_t, t)$$. Simple but effective for smooth velocity fields.

**Sinusoidal Time Embedding:** A technique borrowed from transformer positional encodings. Maps a scalar $$t \in [0, 1]$$ to a high-dimensional vector using sine and cosine functions at different frequencies, giving the network a rich representation of time.
