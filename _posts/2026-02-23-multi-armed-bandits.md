---
layout: distill
title: The Multi-Armed Bandit Problem
description: A visual walkthrough of exploration vs exploitation — from YouTube thumbnails to the algorithms behind smart experimentation
tags: multi-armed bandits, reinforcement learning, exploration-exploitation, bayesian methods
giscus_comments: true
date: 2026-02-23
featured: true

authors:
  - name: Ricardo Huaman
    url: "https://rotvie.github.io/"
    affiliations:
      name: Independent

bibliography: multi-armed-bandits.bib
---

## The Multi-Armed Bandit Problem

There are many resources that explain multi-armed bandits as "a smarter A/B test." While that's one application, it undersells what's going on. The bandit problem captures something more fundamental: **how do you make good decisions when every choice teaches you something about the world?** <d-cite key="Slivkins_2019"></d-cite>

This post builds the full picture — starting from a problem every content creator faces, formalizing it into math, and then walking through three algorithms that solve it in fundamentally different ways.

## You Just Uploaded a Video

You've spent a week editing a video and it's ready to publish. But which thumbnail should you use? You've prepared **5 candidates** — different compositions, text overlays, color palettes. Each thumbnail will attract a different fraction of viewers who see it in their feed. You want to find the best one as quickly as possible.

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mab-five-thumbnails.png" title="Five thumbnail candidates" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Five thumbnail candidates for the same video. Each has an unknown click-through rate (CTR) — the fraction of viewers who click when they see it in their feed.
</div>

YouTube actually does something like this internally. When you upload a video, it can show different thumbnails to different viewers and track which one gets the most clicks. But how should it allocate impressions across the candidates?

### The naive approach: equal split

The standard approach is a fixed A/B test: split impressions equally across all 5 thumbnails for a few days, then keep the winner. Simple, statistically valid — but during that test period, you're sending **equal impressions to every thumbnail, including the bad ones**.

Say T3 has an 8% CTR and T1 has a 2% CTR. An equal-split test keeps showing T1 to 20% of your audience for the entire test. Every impression spent on T1 instead of T3 is a potential click lost.

### The smarter approach: adapt as you learn

A bandit algorithm starts by exploring all thumbnails, but as it gathers evidence, it **shifts impressions toward the winners** while still occasionally checking the others. Fewer wasted impressions, faster convergence to the best thumbnail.

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mab-ab-vs-bandit-allocation.png" title="A/B test vs bandit allocation" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    An A/B test splits impressions equally (left). A bandit algorithm shifts traffic toward the best-performing thumbnail as evidence accumulates (right).
</div>

This is the multi-armed bandit problem: balancing **exploiting** what's working with **exploring** what might work better.

## Formal Definition

Let's translate this into math.

You have $K$ thumbnail candidates. Each thumbnail $i$ has a true CTR $\theta_i$ that you don't know. Every time an impression is served (time step $t$), you choose which thumbnail to show (action $a_t$). The viewer either clicks ($r_t = 1$) or scrolls past ($r_t = 0$):

$$
P(r_t = 1) = \theta_{a_t}
$$

This is called a **Bernoulli bandit** — each action gives a binary outcome, like a coin flip with unknown bias <d-cite key="Lattimore_Szepesvari_2020"></d-cite>.

### Why "Multi-Armed Bandit"?

The name comes from slot machines. A "one-armed bandit" is a single slot machine — one lever, steals your money. A "multi-armed bandit" is the problem of facing $K$ machines at once, each with a different hidden payout. Pull an arm, observe the reward, decide which arm to pull next. Your 5 thumbnails are your 5 arms.

### Regret

We measure performance by **regret** — the total clicks lost by not always showing the best thumbnail:

$$
\mathcal{L}_T = \sum_{t=1}^{T} (\theta^* - \theta_{a_t}), \quad \theta^* = \max_i \theta_i
$$

Every impression spent on a suboptimal thumbnail adds some regret. A good algorithm minimizes this.

### The Estimation Problem

You estimate each thumbnail's CTR from observed data:

$$
\hat{Q}_t(a) = \frac{\text{clicks on thumbnail } a}{\text{impressions of thumbnail } a }
$$

The catch: **early estimates are noisy**. After showing T3 to just 50 viewers, your estimated CTR could be anywhere from 0% to 16%, even if the true CTR is 8%. This noise is what makes the problem hard — and it's why the three algorithms below take fundamentally different approaches to dealing with it.

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mab-estimation-noise.png" title="Noisy estimates" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    After 50 impressions per thumbnail, error bars are wide and overlap — you can't confidently pick the winner. After 2,000 each, estimates are tight — but that's 10,000 impressions, most wasted on bad thumbnails.
</div>

## Algorithm 1: $\varepsilon$-Greedy

The simplest bandit strategy. Most of the time, show the thumbnail with the highest estimated CTR. Occasionally, try a random one:

$$
a_t = \begin{cases} \displaystyle\operatorname*{argmax}_a \hat{Q}_t(a) & \text{with probability } 1 - \varepsilon \quad (\text{show the current best}) \\ \text{random thumbnail} & \text{with probability } \varepsilon \quad (\text{try something random}) \end{cases}
$$

Set $\varepsilon = 0.1$ and 10% of impressions go to a random thumbnail.

**Why it works (sort of):** the random explorations help you discover if a different thumbnail is actually better. Over time, estimates converge and you mostly exploit the true winner.

**Why it's limited:** that random 10% is _blind_. It doesn't care whether you've already confirmed a thumbnail is terrible. After 10,000 impressions, you might have rock-solid evidence that T1 has a 2% CTR, and $\varepsilon$-Greedy will still randomly show it. Those are wasted impressions — viewers who could have been shown T3.

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mab-epsilon-greedy-results.png" title="Epsilon-greedy results" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    $\varepsilon$-Greedy with different exploration rates. Too little exploration ($\varepsilon = 0$) risks getting stuck on a suboptimal thumbnail. Too much ($\varepsilon = 0.3$) wastes impressions on thumbnails already known to be bad. The regret keeps growing linearly — the waste never stops.
</div>

## Algorithm 2: Upper Confidence Bounds (UCB1)

### Why Not Explore Smarter?

The problem with $\varepsilon$-Greedy is that it explores _blindly_. What if, instead of exploring randomly, we sent impressions specifically to **thumbnails we're still uncertain about?**

### Building the Intuition

Imagine you've been running the test and here's what you know:

| Thumbnail | Impressions | Clicks | Observed CTR | How confident are you? |
|-----------|-------------|--------|-------------|----------------------|
| T1 | 500 | 10 | 2.0% | Very confident — it's bad |
| T2 | 500 | 26 | 5.2% | Pretty confident |
| T3 | 12 | 1 | 8.3% | **Not confident at all!** |
| T4 | 500 | 14 | 2.8% | Very confident — it's bad |
| T5 | 500 | 21 | 4.2% | Pretty confident |

T3 _looks_ great at 8.3%, but you've only shown it 12 times. That could easily be noise — maybe nobody happened to be in the mood to click. Or maybe it really is the best. **You should show it more to find out.**

Meanwhile, T1 and T4 have been shown 500 times each. You're very confident they're bad. There's no point sending more impressions their way.

UCB1 formalizes exactly this intuition. For each thumbnail, it computes a score that combines two things:

$$
\text{UCB score}(a) = \underbrace{\hat{Q}_t(a)}_{\text{estimated CTR}} + \underbrace{\sqrt{\frac{2 \ln t}{N_t(a)}}}_{\text{how uncertain you are}}
$$

Then it shows the thumbnail with the highest score.

The second term captures your uncertainty about the estimate. It's large when $N_t(a)$ is small (few impressions → uncertain → explore), and shrinks as you gather data (many impressions → confident → trust the estimate).

### Where Does the Uncertainty Term Come From?

It comes from **Hoeffding's Inequality**, a result from probability theory that bounds how far a sample mean can deviate from the true mean <d-cite key="Auer_2002"></d-cite>.

Hoeffding tells you: given $N$ observations of a random variable bounded in $[0,1]$, the probability that the true mean exceeds the sample mean by more than $u$ is at most $e^{-2Nu^2}$.

If we want to be very sure (probability $\leq t^{-4}$, which is vanishingly small) that our upper bound covers the true CTR, we set $e^{-2Nu^2} = t^{-4}$ and solve:

$$
e^{-2Nu^2} = t^{-4} \implies u = \sqrt{\frac{2\ln t}{N}}
$$

That's the uncertainty term. It's a statistically rigorous confidence bound — hence the name "Upper Confidence Bound."

The philosophy is sometimes called **"optimism in the face of uncertainty"**: act as if each thumbnail is as good as it _plausibly could be_ given the data. Under-explored thumbnails get the benefit of the doubt.

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mab-ucb-scores.png" title="UCB score decomposition" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    UCB1 scores for each thumbnail. The uncertainty term (lighter bar) is large for T3 (only 12 impressions) and tiny for well-explored thumbnails. UCB1 sends the next impression to T3 — the thumbnail it knows the least about.
</div>

## Algorithm 3: Thompson Sampling

### A Different Way to Handle Uncertainty

UCB1 uses a formula to put a number on uncertainty. Thompson Sampling takes a different route: it maintains a **full probability distribution** representing your belief about each thumbnail's true CTR, and uses randomness to decide what to explore <d-cite key="Chapelle_Li_2011"></d-cite>.

### Beliefs as Beta Distributions

For each thumbnail, you keep a belief: _"given everything I've observed, what could this thumbnail's true CTR be?"_ This belief is a **Beta distribution** $\text{Beta}(\alpha, \beta)$, where:

- $\alpha$ = clicks + 1
- $\beta$ = non-clicks + 1

If T2 has received 100 impressions, 5 clicks, 95 skips: the belief is $\text{Beta}(6, 96)$ — peaked near 5%, fairly narrow. You're reasonably confident.

If T3 has received 3 impressions, 1 click: the belief is $\text{Beta}(2, 3)$ — a very wide distribution spanning nearly the whole range from 0% to 70%. You have almost no idea what T3's true CTR is.

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mab-beta-distributions.png" title="Beta distributions at different stages" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    The Beta distribution sharpens with data. With few impressions, the belief is wide — the true CTR could be almost anything. With many impressions, the belief narrows to a tight peak near the true value.
</div>

### The Algorithm

Each time a new viewer arrives:

1. For each thumbnail, **draw a random value** from its belief distribution.
2. Show the viewer **whichever thumbnail drew the highest value**.
3. Observe click or skip. **Update the belief**: click → $\alpha + 1$, skip → $\beta + 1$.

**Why this works:** Thumbnails you're uncertain about have wide beliefs, so their draws occasionally land very high — they get impressions. Thumbnails you're confident about have narrow beliefs — draws cluster near the true CTR. If it's genuinely the best, it consistently "wins the draw." Over time, all beliefs narrow and the best thumbnail dominates.

This property is called **probability matching**: each thumbnail is selected with probability proportional to the probability that it _is_ the best, given what you've observed <d-cite key="Russo_2017"></d-cite>.

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mab-thompson-one-round.png" title="One round of Thompson Sampling" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    One round of Thompson Sampling. Each thumbnail's belief produces a random draw. T3, with its wide belief (only 12 impressions), draws a high value and gets explored — not because of a formula, but because uncertainty naturally produces high samples.
</div>

### Beliefs Converge Over Time

As the algorithm runs, beliefs sharpen. Early on, all distributions are wide and the algorithm explores broadly. Eventually, the best thumbnail's distribution dominates, and almost all impressions flow there.

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mab-thompson-convergence.png" title="Posterior convergence" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Thompson Sampling posteriors at four time steps. At $t = 0$, all beliefs are flat — "any CTR is equally plausible." By convergence, the best thumbnail (T3, 8% true CTR) has a sharp, dominant peak and receives nearly all impressions.
</div>

## Head-to-Head Comparison

Now the main event. All three algorithms on the same 5-thumbnail problem (true CTRs: 2%, 3%, 4%, 5%, 8%), over 10,000 impressions, averaged across 30 runs.

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mab-comparison.png" title="Algorithm comparison" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    (a) Cumulative regret: Thompson and UCB1 flatten, while $\varepsilon$-Greedy grows linearly. (b) Log-log plot reveals the growth rate: $O(t)$ for $\varepsilon$-Greedy, $O(\log t)$ for the others. (c) Impression distribution: Thompson concentrates nearly all impressions on the best thumbnail.
</div>

The key result: $\varepsilon$-Greedy regret grows **linearly** — it never stops wasting impressions. UCB1 and Thompson Sampling grow **logarithmically** — the regret curve flattens as they lock onto the best thumbnail. Thompson typically achieves the lowest total regret.

## Beyond Thumbnails

The explore-exploit tradeoff isn't unique to thumbnails. It appears wherever decisions must be made under uncertainty with limited feedback:

- **Ad placement**: Which banner to show on a webpage. This is where bandits are most widely deployed in industry.
- **Clinical trials**: Choosing between a proven treatment and a promising experimental one. Bandit algorithms have been proposed for adaptive designs that shift patients toward the more effective treatment as evidence accumulates <d-cite key="Villar_2015"></d-cite>.
- **Recommendation systems**: Should a platform show you content it's confident you'll enjoy, or something new that _might_ be a hit?

Even training deep learning models involves this tradeoff. Stochastic Gradient Descent uses mini-batches that introduce noise into gradient estimates — acting as implicit exploration that helps escape local minima. It's the same "productive randomness" that makes Thompson Sampling work.

## Summary

| Algorithm | How it explores | Regret | Key property |
|---|---|---|---|
| $\varepsilon$-Greedy | Randomly — wastes impressions on known-bad thumbnails | Linear — $O(T)$ | Simple but inefficient |
| UCB1 | Targets uncertain thumbnails via a Hoeffding-derived confidence bound | Logarithmic — $O(\log T)$ | Deterministic, theoretically grounded |
| Thompson Sampling | Draws from belief distributions; uncertain thumbnails can "win" by chance | Logarithmic — $O(\log T)$ | Automatic, typically lowest regret |

The core insight: **information has value**. Showing a viewer an under-explored thumbnail isn't wasting that impression — it's investing in learning which thumbnail is best, which pays off by routing all future viewers better. The question is how to make those investments wisely. $\varepsilon$-Greedy invests randomly. UCB1 and Thompson Sampling invest where it matters.

## Glossary

**Click-Through Rate (CTR):** The fraction of impressions that result in a click. If 1,000 viewers see a thumbnail and 50 click, the CTR is 5%.

**Bernoulli Bandit:** A multi-armed bandit where each arm gives a binary reward (0 or 1) with some unknown probability $\theta$.

**Regret:** The cumulative difference between the reward of the best arm and the arm actually chosen, summed over all time steps. Lower is better.

**Upper Confidence Bound (UCB):** A score combining the estimated reward with an uncertainty term derived from Hoeffding's Inequality. Arms with fewer observations get a larger uncertainty term, encouraging exploration.

**Beta Distribution:** A probability distribution on the interval $[0, 1]$, parameterized by $\alpha$ and $\beta$. In Thompson Sampling, it represents the posterior belief about an arm's true reward probability.

**Probability Matching:** The property that each arm is selected with probability equal to the probability it is optimal, given the observed data.
