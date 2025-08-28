---
layout: post
title: Welford's Online Algorithm
date: 2025-08-26 11:12:00-0400
description: Learning the math behind single-pass mean std calculation
tags: math code learning
categories: tech
related_posts: false
---

# Motivation

Up to this point, I have always lazily coded the mean std computation using the standard definition. The reason being that I like to stick to simple solutions until I encounter a case where it no longer works (my personal variant of Keep It Simple Stupid). But I had a rude awakening when I realized that being a simpleton was not going to cut it anymore for my flood mapping ML project. I needed to normalize my dataset for my ML experiments, but reading the enormous amounts of satellite data into memory and calculating the mean and std across it in two passes was a horrific approach to say the least. Luckily I had heard about the Welford's Online Algorithm.

# Math Background

To understand Welford's Algorithm, we first start with a derived identity for single pass variance computation from the standard definition:

$$
\sigma^2 = \frac{\sum_{i=1}^{N} (x_i - \bar{x})^2}{N - 1}
$$

where $$ \bar{x} = \frac{1}{N} \sum_{i=1}^{N} x_i $$.

Expanding the square, we get

$$
\begin{align}
\sigma^2 &= \frac{\sum_{i=1}^{N} (x_i^2 - 2x_i \bar{x} + \bar{x}^2)}{N - 1} \\
&= \frac{(\sum_{i=1}^{N} x_i^2) - 2 (\sum_{i=1}^{N} x_i) (\frac{1}{N} \sum_{i=1}^{N} x_i) + N\bar{x}^2}{N - 1} \\
&= \frac{(\sum_{i=1}^{N} x_i^2) - 2N \bar{x}^2 + N\bar{x}^2}{N - 1} \\
&= \frac{(\sum_{i=1}^{N} x_i^2) - N\bar{x}^2}{N - 1}
\end{align}
$$

Now we can calculate the mean and variance in one pass by keeping track of the sums $$ \sum_{i=1}^{N} x_i^2 $$ and $$ \sum_{i=1}^{N} x_i $$.

However, this approach for calculating variance is **not numerically stable** due to the nature of floating point precision. In the case that variance is small and the data is tightly clustered together, $$ \sum x_i^2 $$ and the square of the mean $$ N \bar{x}^2 $$ can differ very little from each other (usually they happen to be two large numbers with a difference orders of magnitude smaller). Subtracting the two results in catastrophic cancellation due to the loss of the significant digits.

# Interlude: IEEE floating points

Before continuing, I want to elaborate a bit on floating point precision and catastrophic cancellation. This requires a bit of knowledge about floating points. A 32 bit floating point number has 1 sign bit, 8 exponent bits, and 23 mantissa bits. There are also two important subdivisions of floating point values based on the exponent bits: denormalized and normalized. Subnormals or denormalized floating points are those with exponent bits all set to zero and approximate dense values close to zero on the real number line. On the other hand normalized floating points are those with exponent bits that are neither all nonzero or all ones and approximate values further out from zero.

When calculating the value of a **normalized** floating point representation, the exponent has a bias of `127` that is subtracted from the binary value in order to get the final value in the range of `[-126, 127]`. The ultimate value is:

$$
(-1)^{s} \times 2^{(e_7e_6e_5e_4e_3e_2e_1e_0)_2 - 127} \times (1 + \sum_{i=1}^{23} b_{23-i} \times 2^{-i})
$$

where $$s$$ is the sign bit, $$(e_7e_6e_5e_4e_3e_2e_1e_0)_2$$ represents the 8 exponent bits in binary, and $b_i$ are the 23 mantissa bits. Here the exponent bits specify which powers of two you are between, so the interval is $$[2^{e-127}, 2^{e-126})$$ where $$e$$ is the value of the 8 exponent bits.

When calculating the value of **denormalized** floating point representation, the exponent is treated as if it were 1 (before subtracting the bias of 127), and there is no implicit leading 1 in the mantissa. The ultimate value is:

$$
(-1)^{s} \times 2^{-126} \times (\sum_{i=1}^{23} b_{23-i} \times 2^{-i})
$$

where the exponent is fixed at -126 and the mantissa directly represents the fractional part without the implicit leading 1.

There are also more nuances to floating points when the exponent bits are **all 1**. If exponent bits are all one it represents inf or nan with the mantissa becoming the payload. You can read more about it in Goldberg's paper [What Every Computer Scientist Should Know About Floating Point Arithmetic](https://dl.acm.org/doi/10.1145/103162.103163).

So... how does floating point numbers cause catastrophic cancellation? As seen above, the precision of the floating point number is limited to what is representable by the exponent and mantissa bits. Between the interval `[2^2, 2^3)` with 23 mantissa bits, we have $$\frac{2^2}{2^{23}} \approx 4.76 \times 10^{-7}$$ as the spacing between consecutive representable values (this exact difference to get to the next nearest representable floating point value is called the unit in last place or `ulp`). This relative precision of roughly **7 digits** is constant for float 32, and it is **16 digits** for float 64 (from 52 mantissa bits). Catastrophic cancellation occurs when we subtract two nearly equal large numbers which causes the significant digits to cancel out, leaving only the trailing digits which contain rounding errors to remain. The relative error of the resulting difference from the true difference can thus be large. To illustrate, if both numbers are around $$10^4$$ but differ by only $$10^{-2}$$, the subtraction will eliminate 6 significant digits, leaving us with a result that may have little to no accuracy due to the limited precision of floating point representation.

# Welford's Algorithm

Now back to Welford's. What is the workaround here?

Welford's looks at the difference between the sums of squared differences to the mean for $$N$$ and $$N-1$$ samples:

$$
\begin{align}
\sum_{i=1}^{N} (x_i - \bar{x}_N)^2 - \sum_{i=1}^{N-1} (x_i - \bar{x}_{N-1})^2 &= (x_N - \bar{x}_N)^2 + \sum_{i=1}^{N-1} (x_i - \bar{x}_N)^2 - \sum_{i=1}^{N-1} (x_i - \bar{x}_{N-1})^2 \\
&= (x_N - \bar{x}_N)^2 + \sum_{i=1}^{N-1} [(x_i - \bar{x}_N)^2 - (x_i - \bar{x}_{N-1})^2] \\
&= (x_N - \bar{x}_N)^2 + \sum_{i=1}^{N-1} (x_i - \bar{x}_N + x_i - \bar{x}_{N-1})(\bar{x}_{N-1} - \bar{x}_N) \\
&= (x_N - \bar{x}_N)^2 + \sum_{i=1}^{N-1} (2x_i - \bar{x}_N - \bar{x}_{N-1})(\bar{x}_{N-1} - \bar{x}_N) \\
&= (x_N - \bar{x}_N)^2 + (\bar{x}_{N-1} - \bar{x}_N) \sum_{i=1}^{N-1} (2x_i - \bar{x}_N - \bar{x}_{N-1})
\end{align}
$$

We have that the term,

$$
\sum_{i=1}^{N-1} (2x_i - \bar{x}_N - \bar{x}_{N-1}) = 2(N-1)\bar{x}_{N-1} - (N-1)(\bar{x}_N + \bar{x}_{N-1}) = (N-1)(\bar{x}_{N-1} - \bar{x}_N)
$$

The right term in Equation 9 becomes,

$$
\begin{equation}
(\bar{x}_{N-1} - \bar{x}_N) \sum_{i=1}^{N-1} (2x_i - \bar{x}_N - \bar{x}_{N-1}) = (\bar{x}_{N-1} - \bar{x}_N)(N-1)(\bar{x}_{N-1} - \bar{x}_N) = (N-1)(\bar{x}_{N-1} - \bar{x}_N)^2
\end{equation}
$$

Using the recursive mean update formula that $$\bar{x}_N = \frac{(N-1)\bar{x}_{N-1} + x_N}{N}$$, we can rearrange it and get the fact that:

$$
\begin{equation}
\bar{x}_{N-1} - \bar{x}_N = \frac{\bar{x}_{N-1} - \bar{x}_N}{N}
\end{equation}
$$

and

$$
\begin{equation}
\bar{x}_N - x_N = \frac{(N-1)\bar{x}_{N-1}+x_N}{N} - x_N = \frac{N-1}{N}(\bar{x}_{N-1} - x_N)
\end{equation}
$$

Now using equation 10-12, the term becomes:

$$
\begin{equation}
(\bar{x}_{N-1} - \bar{x}_N)(N-1)(\bar{x}_{N-1} - \bar{x}_N) = (\bar{x}_{N-1} - \bar{x}_N) \frac{(N-1)}{N}(\bar{x}_{N-1} - x_N) = (\bar{x}_{N-1} - \bar{x}_N)(\bar{x}_N - x_N)
\end{equation}
$$

Finally,

$$
\begin{align}
\sum_{i=1}^{N} (x_i - \bar{x}_N)^2 - \sum_{i=1}^{N-1} (x_i - \bar{x}_{N-1})^2 &= (x_N - \bar{x}_N)^2 + (\bar{x}_{N-1} - \bar{x}_N) \sum_{i=1}^{N-1} (2x_i - \bar{x}_N - \bar{x}_{N-1}) \\
&= (x_N - \bar{x}_N)^2 + (\bar{x}_{N-1} - \bar{x}_N)(\bar{x}_N - x_N) \\
&= (x_N - \bar{x}_N)(x_N - \bar{x}_N - \bar{x}_{N-1} + \bar{x}_N) \\
&= (x_N - \bar{x}_N)(x_N - \bar{x}_{N-1})
\end{align}
$$

To calculate variance in a single pass, for each new data point say $$ x_k $$, you just need to iteratively update the mean $$ \bar{x}_k = \bar{x}_{k-1} + \frac{x_k - \bar{x}_{k-1}}{k} $$ and keep track of the difference of $$ x_k $$ to the current and previous mean. The product $$ (x_k - \bar{x}_k)(x_k - \bar{x}_{k-1}) $$ can then be added to a growing sum of squared differences term. At the end, simply divide the sum of squared differences by number of data points seen and you get the variance. From the iterative updates you will also have the full mean at the end.

<div class="row mt-3">
    <div class="col-sm-6 mt-3 mt-md-0 mx-auto">
        {% include figure.liquid loading="eager" path="assets/img/welford.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

<div class="caption">
    Welford's algorithm for computing variance in a single pass.
</div>

# Parallelized Welford

If a parallel processor is available, it is [recommended](https://engineering.yale.edu/application/files/3817/3714/7395/tr222.pdf) to split the data into smaller samples and then compute the sum of squares for each sample individually. Then the global sum of squares can be computed by combining these smaller sums in a merging step.

Say you split the dataset into arbitrary groups $$ A, B$$ with $$n_A, n_B$$ samples. Using Welford's on both sets you have the means and sum of squared differences $$ \bar{x}_A, \bar{x}_B, M_{2,A}, M_{2,B}$$, where $$M_2 = \sum (x_i - \bar{x})^2$$. Then,

$$
\begin{align}
n_{AB} &= n_A + n_B \\
\bar{x}_{AB} &= \frac{n_A \bar{x}_A + n_B \bar{x}_B}{n_{AB}} \\
\delta &= \bar{x}_B - \bar{x}_A \\
M_{2,AB} &= M_{2,A}+M_{2,B}+\delta^2 \cdot \frac{n_A n_B}{n_{AB}}
\end{align}
$$

The reason for the extra term when summing up the squared differences is to adjust for the difference between the means of the two groups. We can derive the merge formula, starting with the definition $$ M_{2,AB} = \sum (x_i - \bar{x}_{AB})^2$$:

We can split this into two sums,

$$
M_{2, AB} = \sum_{i \in A} (x_i - \bar{x}_{AB})^2 + \sum_{i \in B} (x_i - \bar{x}_{AB})^2
$$

For set $$A$$, we can substitute $$ (x_i - \bar{x}_{AB}) = (x_i - \bar{x}_A) + (\bar{x}_A - \bar{x}_{AB})$$. Squaring it we have,

$$
(x_i - \bar{x}_{AB})^2 = (x_i - \bar{x}_A)^2 + 2(x_i - \bar{x}_A)(\bar{x}_A - \bar{x}_{AB}) + (\bar{x}_A - \bar{x}_{AB})^2 \\
$$

Summing this over all points in $$A$$, we get,

$$
\begin{align}
\sum_{i \in A} (x_i - \bar{x}_{AB})^2 &= \sum_{i \in A} (x_i - \bar{x}_A)^2 + 2(\bar{x}_A - \bar{x}_{AB}) \sum_{i \in A} (x_i - \bar{x}_A) + n_A (\bar{x}_A - \bar{x}_{AB})^2 \\
&= M_{2, A} + 2(\bar{x}_A - \bar{x}_{AB}) \cdot 0 + n_A (\bar{x}_A - \bar{x}_{AB})^2 \\
&= M_{2, A} + n_A (\bar{x}_A - \bar{x}_{AB})^2
\end{align}
$$

The same result applies to set $$B$$. Thus we have,

$$
\begin{align}
M_{2, AB} &= M_{2, A} + n_A (\bar{x}_A - \bar{x}_{AB})^2 + M_{2, B} + n_B (\bar{x}_B - \bar{x}_{AB})^2 \\
&= M_{2, A} + M_{2, B} + n_A (\bar{x}_A - \bar{x}_{AB})^2 + n_B (\bar{x}_B - \bar{x}_{AB})^2
\end{align}
$$

We can simplify the right two terms. We know that $$ \bar{x}_{AB} = \frac{n_A \bar{x}_A + n_B \bar{x}_B}{n_A + n_B}$$, which allows us to rewrite:

$$
\begin{align}
(\bar{x}_A - \bar{x}_{AB}) &= (\bar{x}_A - \frac{n_A \bar{x}_A + n_B \bar{x}_B}{n_A + n_B}) = \frac{n_B}{n_A + n_B} (\bar{x}_A - \bar{x}_B)
&= \frac{n_B}{n_{AB}} (\bar{x}_A - \bar{x}_B)
\end{align}
$$

Similarly,

$$
(\bar{x}_B - \bar{x}_{AB}) = \frac{n_A}{n_{AB}} (\bar{x}_B - \bar{x}_A)
$$

Now we have,

$$
\begin{align}
n_A (\bar{x}_A - \bar{x}_{AB})^2 + n_B (\bar{x}_B - \bar{x}_{AB})^2 &= n_A (\frac{n_B}{n_{AB}})^2 (\bar{x}_A - \bar{x}_B)^2 + n_B (\frac{n_A}{n_{AB}})^2 (\bar{x}_B - \bar{x}_A)^2 \\
&= (\frac{n_A n_B^2}{n_{AB}^2} + \frac{n_B n_A^2}{n_{AB}^2}) (\bar{x}_A - \bar{x}_B)^2 \\
&= (\frac{n_A n_B^2 + n_B n_A^2}{n_{AB}^2}) (\bar{x}_A - \bar{x}_B)^2 \\
&= (\frac{n_A n_B (n_B + n_A)}{n_{AB}^2}) (\bar{x}_A - \bar{x}_B)^2 \\
&= (\frac{n_A n_B (n_{AB})}{n_{AB}^2}) (\bar{x}_A - \bar{x}_B)^2 \\
&= (\frac{n_A n_B}{n_{AB}}) (\bar{x}_A - \bar{x}_B)^2 \\
&= \delta^2 \frac{n_A n_B}{n_{AB}} \\
\end{align}
$$

Which is the between groups term in equation 21 that we must account for.