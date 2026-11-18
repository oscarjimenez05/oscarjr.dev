---
layout: '../../layouts/Project.astro'
title: Lehmer PRNG
description: "Pseudo-Random Number Generator using sliding window permutations. Accompanying research paper to be posted soon."
badge: "Research"
tags: ["C/Cython", "Cryptography"]
priority: 10
date: 2026-10-19
---

## The Core Problem
Pseudo-Random Number Generators (PRNGs) generally fall into one of two categories: blazingly fast algorithms that are statistically terrible (like Linear Congruential Generators or basic Xorshifts), and cryptographically secure generators (CSPRNGs) that rely on hardware entropy and system calls, making them far too slow for massive simulations.

The goal of this research was to see if we could take a mathematically awful, cheap generator and "whiten" its output by ignoring the actual numbers it produces. Instead, we only look at the *relative order* of small sliding windows of numbers.

---

## How "Lehmerizing" Works
Take a fast, flawed generator like a 64-bit Xorshift and generate a raw stream of numbers. Instead of using those numbers directly, we grab a sliding window of size $w$ (say, $w=3$) and map their relative magnitudes to a **Lehmer Code**.

For each element in the window, you simply count how many elements to its right are smaller than it. For example:
1. Grab a window of 3 numbers: `[1.08e9, 4.28e9, 3.83e9]`
2. Count smaller elements to the right: `(0, 1, 0)`
3. Convert from the factorial number system to base-10: $0(2!) + 1(1!) + 0(0!) = 1$

Because we only care about pairwise comparisons ($W[j] < W[i]$) and not the raw values, bit-level parity issues, low-order periodicity, and hyperplane lattice structures inherent to simple generators vanish.

### Generalized Ranges & Rejection Sampling
A window of size $w$ naturally outputs integers in the range $[0, w! - 1]$. If you want a random calendar date in $[0, 364]$, you pick the smallest factorial that fits ($6! = 720$). 

Naively applying `output % 365` ruins the uniformity because $720 \pmod{365} = 355$, meaning the first 355 days of the year appear twice as often as the last 10. To fix this, we discard incomplete cycles where the output $v \ge w! - (w! \bmod R)$.

To avoid wasting CPU cycles throwing away half our numbers, we modeled the hardware execution cost:

$$T_{(w,R)} = \frac{c_1 w^2 + c_2 w + c_3}{1 - \frac{w! \bmod R}{w!}}$$

At startup, the generator runs a quick quadratic regression to find the hardware constants ($c_1, c_2, c_3$) and automatically selects the optimal window size $w_s$ that balances the $O(w^2)$ comparison cost against the rejection rate.

---

## Benchmarks & The "Seesaw" Bug
Writing this in pure Python was roughly $7\times$ slower than Python's built-in Mersenne Twister. Porting the core loop to **Cython** (and replacing a C array with a raw pointer reference) yielded a $60\times$ speedup, making the Lehmerized Xorshift (`XOR_LH`) roughly **$7.8\times$ faster** than a standard Mersenne Twister loop.

To verify randomness, we threw the generator at **TestU01's BigCrush**—a notoriously brutal battery of 160 statistical tests that chews through over a terabyte of generated numbers.

### The Overlapping Window Trap
Initially, we slid the window by $\delta = 1$ element at a time to save computations. BigCrush immediately flagged four failures (*Knuth's Run*, *Sum Collector*, *String Run*, and *String Autocorrelation*). 

The culprit was a "seesaw" effect. If a tiny number sits at the end of window $n$, it inflates the count of every number before it, producing a massive output. In window $n+1$, that exact same tiny number shifts to the front, forcing a leading zero and guaranteeing a tiny output next. The generator was oscillating high-low-high-low.

Setting the stride to $\delta = w$ (non-overlapping windows) completely eliminated the seesaw effect.

| Generator | Stride Config | BigCrush Stats | Failures | Verdict |
| :--- | :--- | :--- | :--- | :--- |
| **PCG-64** (Industry Standard) | 64-bit state | 160 | 2 | Failed (*CollisionOver*, *BirthdaySpacings*) |
| **XOR_LH** (Overlapping) | $w = 14, \delta = 13$ | 160 | 4 | Failed (Serial correlation seesaw) |
| **XOR_LH** (Non-Overlapping) | $w = 14, \delta = 14$ | 160 | **0** | **Complete Pass** |
| **LCG_LH** (Non-Overlapping) | $w = 14, \delta = 14$ | 160 | **0** | **Complete Pass** |

---

## Whitening Non-Uniform Chaos
Because the Lehmer transformation only evaluates ordinal rankings ($u_i < u_j \iff F^{-1}(u_i) < F^{-1}(u_j)$), it is mathematically invariant under strictly monotonic transformations. 

If you feed a heavily skewed exponential decay, a power slope ($X^5$), or a Gaussian bell curve into the Lehmer whitener, it outputs a flat, statistically uniform distribution. The only time it breaks is when consecutive inputs have extreme deterministic interdependence—such as the Chaotic Logistic Map ($x_{n+1} = 4x_n(1-x_n)$), where it is mathematically impossible to ever generate two consecutive descending steps.

---

## Cryptographic Failure (And a Happy Accident)
Naturally, the next question was: *Can we use this for cryptography?*

Short answer: **No.**

Even after building a hardened variant (`CryptoLehmer`) with a 320-bit state, non-linear ChaCha20 ARX mixing rounds, and non-deterministic bit-discarding, it failed Claude Shannon's diffusion criteria. Because inequality comparisons ($x < y$) halt at the first differing Most Significant Bit, an attacker can model the comparisons as linear equations in $\mathbb{F}_2$ and recover the seed in polynomial time ($O(k^{2.807})$) via Gaussian elimination and branch-and-bound.

### Pivoting to Computer Vision (Locality Sensitive Hashing)
In cryptography, lacking the "avalanche effect" and ignoring monotonic transformations is a fatal flaw. In **Locality Sensitive Hashing (LSH)**, those exact flaws are the holy grail.

We retooled the 1D sliding window into a vectorized 2D NumPy image pipeline to generate perceptual image hashes:
- **Multiscale Pyramid & Thresholding:** Downsamples grayscale images ($1.0\times, 0.85\times, 0.70\times$) and ignores low-variance flat backgrounds ($\max(W) - \min(W) < 15$) so empty skies don't cause false positives.
- **Spatial Shingling:** Groups $w=5$ window permutations into 4-grams ($2.07 \times 10^8$ possible states) pinned to an $8 \times 8$ spatial grid.
- **Containment Scoring:** Compares multiset histograms via $O(A,B) = \frac{|A \cap B|}{\min(|A|, |B|)}$.

When benchmarked across 100 ImageNet samples against standard spatial **MinHash**, the ordinal approach proved wildly resilient to photometric edits:

| Perturbation Condition | Lehmer LSH | Spatial MinHash |
| :--- | :--- | :--- |
| **Monotonic Tone / Brightness Distortion** | **89.94%** | 0.15% |
| **Gaussian Noise & Sensor Blur** | **70.52%** | 0.01% |
| **Foreground Occlusion (15%–30% blocked)** | **97.87%** | 88.27% |
| **Random Cropping (75%–90% retained)** | **56.13%** | 4.04% |

Because extreme brightness, contrast, and gamma shifts are just monotonic transformations of pixel intensity, the relative orderings of the pixels stayed intact—resulting in a **99.26% similarity match** on severely altered images where traditional pixel hashing completely collapsed.