---
layout: distill
title: Decrypting Ciphers Using Markov Chain Monte Carlo
description: Combining Markov chains and Monte Carlo estimation to crack a substitution cipher — from bigram statistics to the Metropolis algorithm
tags: MCMC, statistics, probability, cryptography
giscus_comments: false
date: 2026-04-05
featured: false

authors:
  - name: Ricardo Huaman
    url: "https://rotvie.github.io/"
    affiliations:
      name: Independent

bibliography: decrypting-ciphers-mcmc.bib
---

## Introduction

In the [previous post](https://rotvie.github.io/blog/2026/introduction-to-sampling-algorithms/), we saw how sampling methods like rejection sampling and importance sampling break down in high dimensions — the probability of generating a useful sample drops exponentially as the space grows. Markov Chain Monte Carlo (MCMC) sidesteps this by constructing a sequence of correlated samples that gradually explore the target distribution, taking small, informed steps instead of hoping random guesses land in the right place.

MCMC has applications across computational physics, biology, statistics, and machine learning <d-cite key="Robert_Casella_2004"></d-cite>. But here we'll explore a particularly fun one: **using MCMC to crack a substitution cipher** — a coded message where each letter has been swapped for another <d-cite key="Rohde_2022"></d-cite>.

To get there, we need two ingredients: Markov chains and Monte Carlo estimation. Let's build them up.

## Markov Chains

A Markov chain is a stochastic process that hops between states, where the probability of moving to the next state depends _only_ on the current state — not on how you got there. This is the **Markov property**: the future is conditionally independent of the past given the present.

Let's make this concrete. Suppose we have three states — $$A$$, $$B$$, $$C$$ — with transition probabilities that we can write compactly as a **transition matrix** $$P$$, where entry $$P_{ij}$$ is the probability of moving from state $$i$$ to state $$j$$:

$$
P = \begin{pmatrix}
0.2 & 0.5 & 0.3 \\
0.1 & 0.6 & 0.3 \\
0.4 & 0.2 & 0.4
\end{pmatrix}
$$

Each row sums to 1 — from any state, the probabilities of going somewhere (including staying put) must account for all possibilities.

If we start in state $$A$$ and simulate the chain by following these probabilities, we might get a sequence like: $$A \to B \to B \to C \to A \to B \to C \to C \to A \to B \to B \to \cdots$$

### Convergence to a Stationary Distribution

Here's the key property that makes Markov chains useful for sampling. If we run the chain long enough and count how often we visit each state, the proportions converge to a fixed distribution — the **stationary distribution** $$\pi$$ — regardless of where we started.

We can see this algebraically. Let $$\mathbf{s}_t$$ be a row vector representing the probability of being in each state at time $$t$$. Starting from state $$A$$, we have $$\mathbf{s}_0 = (1, 0, 0)$$. After one step:

$$\mathbf{s}_1 = \mathbf{s}_0 \cdot P = (0.2, 0.5, 0.3)$$

After two steps:

$$\mathbf{s}_2 = \mathbf{s}_1 \cdot P = (0.19, 0.46, 0.35)$$

If we keep multiplying, the distribution converges:

| Step | $$P(A)$$ | $$P(B)$$ | $$P(C)$$ |
| ---- | -------- | -------- | -------- |
| 0    | 1.000    | 0.000    | 0.000    |
| 1    | 0.200    | 0.500    | 0.300    |
| 2    | 0.190    | 0.460    | 0.350    |
| 5    | 0.196    | 0.390    | 0.414    |
| 10   | 0.196    | 0.392    | 0.412    |
| 50   | 0.196    | 0.392    | 0.412    |

The distribution stops changing — it has converged to the stationary distribution $$\pi \approx (0.196, 0.392, 0.412)$$. This $$\pi$$ satisfies:

$$\pi = \pi P$$

Meaning: if you're distributed according to $$\pi$$ and take one step, you stay distributed according to $$\pi$$. The chain has reached equilibrium.

What makes this remarkable is that **the stationary distribution doesn't depend on the starting state**. Start from $$B$$ or $$C$$ instead of $$A$$, and you'll converge to the same $$\pi$$. This is guaranteed (under mild conditions: the chain must be **irreducible** — you can get from any state to any other — and **aperiodic** — you don't get trapped in deterministic cycles).

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/decrypting-ciphers-mcmc/mcmc-markov-chain-simulation.png" title="Markov chain simulation" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

This is the property MCMC exploits: if we can construct a Markov chain whose stationary distribution is the target distribution we want to sample from, then running the chain long enough gives us samples from that target — even if we can't sample from it directly.

## Monte Carlo Methods

We covered Monte Carlo estimation in the [sampling algorithms post](https://rotvie.github.io/blog/2026/introduction-to-sampling-algorithms/), but let's recap the essential idea. Monte Carlo methods approximate expectations by averaging over samples:

$$
E_p[g(X)] \approx \frac{1}{N} \sum_{i=1}^{N} g(x_i), \quad x_i \sim p(x)
$$

The Law of Large Numbers guarantees this converges to the true expectation as $$N \to \infty$$. The key requirement is that the samples $$x_i$$ come from the distribution $$p(x)$$.

But what if we can't sample from $$p$$ directly? That's exactly the scenario where MCMC shines. Instead of drawing independent samples from $$p$$, we construct a Markov chain whose stationary distribution _is_ $$p$$, run it until convergence (the "burn-in" period), and then treat the subsequent states as (correlated) samples from $$p$$. The combination of **M**arkov **C**hains (to generate samples) and **M**onte **C**arlo (to estimate quantities from those samples) gives MCMC its name.

## The Substitution Cipher

The substitution cipher is one of the simplest cryptographic methods. It works by swapping each letter in the alphabet with another according to a fixed mapping.

A simple example is a **shift cipher**, where we shift the alphabet by a fixed number of positions. A shift of 3 would give:

{% highlight text %}
Alfabeto original: abcdefghijklmnopqrstuvwxyz
Alfabeto cifrado: defghijklmnopqrstuvwxyzabc
{% endhighlight %}

With this cipher, the word `hola` would map to:

- `h -> k`
- `o -> r`
- `l -> o`
- `a -> d`

resulting in the ciphertext `krod`.

> **Terminology.** **Plaintext** refers to the text you input to the cipher (`hola`) and **ciphertext** refers to the output (`krod`).

More generally, we can create an arbitrary substitution cipher by permuting the letters of the alphabet — not just shifting them. Each cipher is a permutation $$\sigma$$ of the character set. Encryption applies $$\sigma$$ letter by letter; decryption applies the inverse $$\sigma^{-1}$$.

For our problem, we'll work with **normalized Spanish text**: 26 lowercase letters plus space (no `ñ`, no accents, no punctuation) — 27 characters total. Here's the plaintext we'll use, from Borges' _Ficciones_ <d-cite key="Borges_Ficciones"></d-cite>:

{% highlight text %}
he dicho que los hombres de ese planeta conciben el universo como una serie
de procesos mentales que no se desenvuelven en el espacio sino de modo
sucesivo en el tiempo
{% endhighlight %}

Now suppose someone intercepts the ciphertext but doesn't know the cipher. How would they decode it?

### Why Not Brute Force?

A first idea: try all possible ciphers until we find the right one. How many are there? Each cipher is a permutation of our 27-character set, so there are $$27!$$ possibilities:

$$27! \approx 1.09 \times 10^{28}$$

That's an astronomically large number. As a thought experiment, let's assume we have access to the world's fastest supercomputer, reported to compute $$\sim 1.1 \times 10^{18}$$ operations per second. Although checking a cipher would take more than a single operation, let's assume we can check one per operation to see if this approach is even feasible:

$$\frac{1.09 \times 10^{28} \text{ ciphers}}{1.1 \times 10^{18} \text{ ciphers/sec}} \approx 9.9 \times 10^{9} \text{ seconds} \approx 314 \text{ years}$$

Even with extremely generous assumptions, brute force is hopeless. We need a smarter approach — one that seeks out the likely ciphers and ignores the rest. This is what the Metropolis algorithm does <d-cite key="Metropolis_1953"></d-cite>.

## Defining a "Spanish-Similarity Score"

Before we can run the Metropolis algorithm, we need a way to measure how similar an arbitrary text looks to Spanish. Even without speaking the language, you could probably tell that `krod` doesn't look like Spanish — most real words don't have that letter pattern.

The idea: real Spanish has strong statistical regularities in which characters tend to appear next to each other. The pair `de` is extremely common; `qz` almost never occurs. We can exploit this by looking at **bigram frequencies** — how often each pair of consecutive characters appears in a large Spanish corpus.

### Building a Bigram Matrix with PyTorch

Let's build a bigram frequency matrix from a reference Spanish text. We'll use a corpus of Borges' collected works <d-cite key="Borges_Corpus"></d-cite>, cleaned to contain only our 27 characters.

The approach follows the character-level language model pattern: we create a count matrix $$N$$ (27 characters + a special boundary token `.`) where $$N_{ij}$$ counts how many times character $$j$$ follows character $$i$$ in the corpus. We count bigrams over the continuous text (including spaces), with the boundary token only at the start and end of the corpus.

<!-- prettier-ignore-start -->
{% highlight python %}
import torch

# Load and clean the reference corpus
with open('corpus.txt', 'r', encoding='utf-8') as f:
    text = f.read()

# Normalize: lowercase, remove accents, ñ -> n, keep only a-z and space
import unicodedata
def normalize_spanish(s):
    s = s.lower()
    s = s.replace('\n', ' ')  # treat newlines as word boundaries
    s = unicodedata.normalize('NFD', s)
    s = ''.join(c for c in s if unicodedata.category(c) != 'Mn')  # strip accents
    s = s.replace('ñ', 'n')
    s = ''.join(c if c in 'abcdefghijklmnopqrstuvwxyz ' else '' for c in s)
    s = ' '.join(s.split())  # collapse multiple spaces
    return s

text = normalize_spanish(text)

# Character-to-index mapping: '.' = 0 (boundary), ' ' = 1, 'a' = 2, ..., 'z' = 27
words = text.split()
chars = sorted(list(set(' '.join(words))))
stoi = {s: i+1 for i, s in enumerate(chars)}
stoi['.'] = 0
itos = {i: s for s, i in stoi.items()}

# Build bigram count matrix over continuous text
n_tokens = len(stoi)
N = torch.zeros((n_tokens, n_tokens), dtype=torch.int32)

chs = ['.'] + list(text) + ['.']
for ch1, ch2 in zip(chs, chs[1:]):
    ix1 = stoi[ch1]
    ix2 = stoi[ch2]
    N[ix1, ix2] += 1
{% endhighlight %}
<!-- prettier-ignore-end -->

We can visualize this matrix as a heatmap — brighter cells indicate more frequent bigrams:

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/decrypting-ciphers-mcmc/mcmc-bigram-heatmap.png" title="Bigram frequency heatmap" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

Now we convert counts to probabilities. To avoid zero probabilities for unseen bigrams (which would make the log-likelihood $$-\infty$$), we add 1 to every count before normalizing — this is **Laplace smoothing**:

<!-- prettier-ignore-start -->
{% highlight python %}
# Convert to probability matrix with Laplace smoothing
P = (N + 1).float()
P /= P.sum(1, keepdims=True)
{% endhighlight %}
<!-- prettier-ignore-end -->

Each row of $$P$$ now sums to 1 and represents a probability distribution: $$P_{ij}$$ is the estimated probability that character $$j$$ follows character $$i$$ in Spanish.

### Scoring a Text

Given a candidate decryption $$d = d_1 d_2 \dots d_n$$, we can measure how "Spanish-like" it looks by computing the product of all consecutive bigram probabilities:

$$\text{sim}(d) = P(d_1, d_2) \times P(d_2, d_3) \times \cdots \times P(d_{n-1}, d_n)$$

This product will be large if the text follows typical Spanish letter patterns (lots of common bigrams) and tiny if it doesn't (full of rare bigrams).

However, multiplying hundreds of small probabilities together quickly underflows to zero. We work on the **log scale** instead, turning products into sums:

$$\text{Score}(d) = \sum_{i=1}^{n-1} \log P(d_i, d_{i+1})$$

Higher scores mean more Spanish-like text. This is our **log-likelihood**.

<!-- prettier-ignore-start -->
{% highlight python %}
def score_text(text, P, stoi):
    """Compute log-likelihood of a text under the bigram model."""
    log_prob = 0.0
    for ch1, ch2 in zip(text, text[1:]):
        ix1 = stoi.get(ch1, 0)
        ix2 = stoi.get(ch2, 0)
        log_prob += torch.log(P[ix1, ix2]).item()
    return log_prob
{% endhighlight %}
<!-- prettier-ignore-end -->

Let's verify that the score can differentiate Spanish from gibberish:

<!-- prettier-ignore-start -->
{% highlight python %}
>>> score_text("esto es texto en espanol", P, stoi)
-78.43

>>> score_text("fghr gh wghdfrf etfs xqzk", P, stoi)
-112.67
{% endhighlight %}
<!-- prettier-ignore-end -->

The Spanish text gets a much higher (less negative) score — exactly what we want.

## The Metropolis Algorithm

Now we have everything we need. The Metropolis algorithm constructs a Markov chain over the space of all $$27!$$ possible ciphers, biased to visit ciphers that produce Spanish-like decryptions more often <d-cite key="Metropolis_1953"></d-cite>.

The algorithm works like this. Start with a random cipher. Then repeat:

1. **Propose** a new cipher by swapping two characters in the current cipher at random. This small change is our proposal — it produces a cipher that's "close" to the current one.

2. **Score** both: decode the ciphertext with the proposed cipher and the current cipher, and compute the log-likelihood of each decoded text.

3. **Accept or reject**: compute the acceptance probability
   $$\alpha = \min\left(1, \; e^{\text{Score}(\text{proposed}) - \text{Score}(\text{current})}\right)$$
   If the proposed decryption looks more like Spanish ($$\alpha \geq 1$$), always accept it. If it looks less like Spanish, accept it with probability $$\alpha$$. This occasional acceptance of worse states prevents the chain from getting stuck in local optima.

Notice that we only need the _ratio_ of scores (equivalently, the _difference_ of log-scores), so any normalizing constants cancel. This is the key property that makes MCMC practical — we never need to compute the intractable normalization over $$27!$$ permutations.

Since our proposal is symmetric (swapping letters $$i \leftrightarrow j$$ is the same move as swapping $$j \leftrightarrow i$$), this is the original **Metropolis** algorithm — the special case of Metropolis-Hastings where the proposal cancels from the acceptance ratio <d-cite key="Hastings_1970"></d-cite>.

### Implementation

<!-- prettier-ignore-start -->
{% highlight python %}
import random

# The plaintext (we pretend we don't know this)
plaintext = ("he dicho que los hombres de ese planeta conciben el universo "
             "como una serie de procesos mentales que no se desenvuelven "
             "en el espacio sino de modo sucesivo en el tiempo")

# Create the alphabet: a-z + space
alphabet = list('abcdefghijklmnopqrstuvwxyz ')

def generate_cipher():
    """Generate a random permutation of the alphabet."""
    cipher = alphabet.copy()
    random.shuffle(cipher)
    return cipher

def encode(text, cipher):
    """Encode text using a cipher (permutation)."""
    mapping = {original: coded for original, coded in zip(alphabet, cipher)}
    return ''.join(mapping.get(c, c) for c in text)

def decode(ciphertext, cipher):
    """Decode ciphertext using a cipher (inverse permutation)."""
    mapping = {coded: original for original, coded in zip(alphabet, cipher)}
    return ''.join(mapping.get(c, c) for c in ciphertext)

def swap_two(cipher):
    """Propose a new cipher by swapping two random positions."""
    new_cipher = cipher.copy()
    i, j = random.sample(range(len(cipher)), 2)
    new_cipher[i], new_cipher[j] = new_cipher[j], new_cipher[i]
    return new_cipher

# Generate the true cipher and produce the ciphertext
true_cipher = generate_cipher()
ciphertext = encode(plaintext, true_cipher)

# Start with a random cipher
current_cipher = generate_cipher()
current_score = score_text(decode(ciphertext, current_cipher), P, stoi)

# Run the Metropolis algorithm
for iteration in range(50000):
    # Propose a new cipher
    proposed_cipher = swap_two(current_cipher)

    # Score the proposed decryption
    proposed_text = decode(ciphertext, proposed_cipher)
    proposed_score = score_text(proposed_text, P, stoi)

    # Acceptance probability (on log scale: difference of scores)
    log_alpha = proposed_score - current_score

    # Accept or reject
    if log_alpha >= 0 or random.random() < math.exp(log_alpha):
        current_cipher = proposed_cipher
        current_score = proposed_score

        if iteration % 500 == 0:
            print(f"Iteration {iteration}: {proposed_text}")
{% endhighlight %}
<!-- prettier-ignore-end -->

Here's what the output looks like as the chain converges:

{% highlight text %}
Iteration 0: dxptfsmihqxtiehdienbxehtxexexokglxfghsiglfxxtixqlfaxuehxshenxqgn...
Iteration 500: je disno wue cos nombtes de ele praneta sonsiben er univelso...
Iteration 2000: he dicho que los hombres de ese planeta conciben el universo...
{% endhighlight %}

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/decrypting-ciphers-mcmc/mcmc-cipher-convergence.png" title="Cipher convergence" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

The chain starts with gibberish and within a few thousand iterations, the Borges quote emerges. Each accepted swap nudges the decryption toward more Spanish-like text. The occasional acceptance of worse proposals allows the chain to escape local optima — configurations where a few letters are correct but swapping any single pair makes things worse before they get better.

### The Burn-In Period

The chain starts in a random state that is almost certainly far from the target distribution. The initial samples are not representative — the chain needs time to "burn in" and reach the high-probability region. We can see this by plotting the log-likelihood score over iterations: it rises sharply at first, then levels off once the chain has found the neighborhood of the true cipher.

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/decrypting-ciphers-mcmc/mcmc-burnin-score.png" title="Burn-in score trajectory" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

In practice, we discard the first several thousand samples (the burn-in period) and only use subsequent ones for inference. For our cipher problem, we're using MCMC for optimization rather than sampling — we just want the highest-scoring cipher found — so burn-in is less of a concern. But it's worth noting that this same machinery, when used for Bayesian inference, requires careful attention to convergence diagnostics.

### Practical Notes

A few things that affect convergence:

- **Text length matters.** Longer texts contain more bigram information, making the score function sharper — the algorithm converges faster. Our Borges quote (~180 characters) is on the short side; a full paragraph would converge in fewer iterations.
- **Random restarts help.** Since the cipher space has many local optima, some initial ciphers lead to chains that get stuck. If the chain hasn't converged after 20,000 iterations, restarting with a new random cipher is a reasonable strategy for optimization (though not for sampling applications of MCMC).
- **The proposal matters.** Swapping two letters is a good proposal because it makes a small change — most of the decoded text stays the same. If we instead proposed completely random permutations, the acceptance rate would be near zero and the chain would never move.

## Summary

We took two ideas — Markov chains converging to a stationary distribution, and Monte Carlo estimation via sampling — and combined them into MCMC, a method that can sample from distributions over spaces far too large to enumerate.

The cipher decryption example illustrates the core mechanism:

1. We defined a **score function** (bigram log-likelihood) that measures how Spanish-like a decoded text looks
2. This score induces a **probability distribution** over the $$27!$$ possible ciphers — higher scores mean more probable
3. The **Metropolis algorithm** explores this space efficiently by proposing small changes (letter swaps), accepting improvements, and occasionally accepting worse states to avoid getting trapped

| Concept                 | Role in cipher decryption                                                         |
| ----------------------- | --------------------------------------------------------------------------------- |
| Markov chain            | Sequence of ciphers, each obtained by swapping two letters in the previous one    |
| Stationary distribution | The distribution over ciphers proportional to Spanish-likeness score              |
| Monte Carlo             | Using the chain's samples to find the most likely decryption                      |
| Metropolis algorithm    | The accept/reject rule that ensures the chain converges to the right distribution |

This is the same machinery behind Bayesian inference in high-dimensional models, protein folding simulations, and probabilistic programming — just with different state spaces and score functions.
