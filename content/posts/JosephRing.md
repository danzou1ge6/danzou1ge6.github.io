+++
title = "Extended Joseph Ring"
date = 2023-03-01T16:04:18+08:00
draft = false
tags = ["algorithm"]
keywords = ["Joseph Ring"]
description = "Notes for Extended Joseph Ring"

+++

# Extended Joseph Ring

## The Extension

In traditional Joseph Ring, candidates numbered $1, \cdots, n$ are eliminated $m$ by $m$.

An extension to this is choosing different $m$ at each elimination. For example, with $n = 5$ and $k = [1, 2, 3, 4]$, the process is illustrated below:

```
Original  1 2 3 4 5
k = 1     x 1 2 3 4
k = 2       3 x 1 2
k = 3       x   1 2
k = 4           1 x
```

Eventually, only $4$ is left.

## Problem

Usually, the problem is to determine which canditate will remain after $n - 1$ rounds of elimination.

A simple way is to utilize circular linked list. However, there exists a faster method, which can be extended as well. This method is based on the following result.

## A Useful Result

We define that before an elimination, remaining candidates are numbered
$$1, \cdots n$$

Then, the $k^{\mathrm{th}}$ candidate is eliminated, leaving
$$X = [1, \cdots, k - 1, k + 1, \cdots, n]$$
which are re-indexed as
$$Y = [n - k + 1, \cdots, n - 1, 1, \cdots, n - k]$$

Actually, it can be verified that there exists a mapping from $X_i$ to $Y_i$:
$$Y_i = p(X_i) = (X_i + n - k) \;\mathrm{mod}\; n$$

from which we can derive the inverse mapping
$$X_i = p^{-1}(Y_i) = (Y_i + k) \;\mathrm{mod}\; n$$

Note that for the $\mathrm{mod}$ operation here,
$$0 \;\mathrm{mod}\; n = n$$
to make sure $p^{-1}(Y_i)$ is within range of $X_i$.

## Solution

The problem can be restated as: Given a set of candidates indexed $1, \cdots, n$, and a series of $k_1, \cdots, k_{n-1}$, we need to determine the index for the eventually remaining candidate.

For each $k_t$, we define $p_t$ according to the above result. Note that the number of remaining candidates $n$ in $p$'s defination needs to be substituted by $n - t + 1$. Then, we annotate the origin indices with
$$X^0 = [1, \cdots, n]$$

After eliminating $k_1$ from $X_0$, the remainings are re-indexed
$$X^1_i = p(X^0_i), X^0_i \ne k_1$$
for next round of elimination.

The process is repeated for $n - 1$ times, and the final $X^{n - 1}$ only consists of single candidate indexed $1$. The goal is to find out the original index for this candidate.

This can be achieved by traversing back the elimination process. Using the previous result, if an item is indexed $x$ in the $t^{\mathrm{th}}$ round, its index in the ${t-1}^{\mathrm{th}}$ round must be $p_t^{-1}(x)$. Therfore, the origin index of the final candidate is
$$p_1^{-1} \circ p_2^{-1} \circ \cdots \circ p_{n - 1}^{-1} (1)$$

The Rust code for solving the probel is listed below.


```Rust
fn last(n: i32, k_vec: Vec<i32>) -> Result<i32, ()> {
    if k_vec.len() != (n - 1).try_into().unwrap() { Err(()) }
    else {
        
        let mut x = 1;
        for t in (1..n).rev() {
            x = (x + k_vec[usize::try_from(t).unwrap() - 1]) % (n - t + 1);
        }
        
        Ok(x)
    }
}
```


```Rust
let n = 5;
let k_vec = vec![1, 2, 3, 4];
let x = last(n, k_vec).unwrap();
println!("The remaining candidate is {}", x);
```

    The remaining candidate is 4



```Rust

```
