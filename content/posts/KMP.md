+++
title = "KMP String Matching Algorithm"
date = 2023-03-01T16:04:18+08:00
draft = false
tags = ["algorithm"]
keywords = ["KMP", "string matching"]
description = "Notes for KMP String Matching Algorithm"

+++

# KMP String Matching Algorithm

## Rational

Brute-force string matching requires $O(MN)$ time, where $N$ is the length of the **source string** $S$, and $M$ is the length of the **target string** $P$.

However, utilizing the following fact, time consumption can be further reduced:

For example,
```
Index: 012345
S    = abxabx
P    = abxaby       <- Attempt 1
          abx       <- Attempt 2
```

When Attempt 1 failed because $S[5] \ne P[5]$, $S[:4]$ has already been matched. Noticing that $P$ starts with "ab" while $S[3:5]$, which equals $P[3:5]$, is also "ab", we can continue matching by aligning "x" at $P[2]$ with $S[5]$ as shown in Attempt 2.

## The "next" Array

Therefore, the key to accelerating matching is by locating **longest identical prefix and suffix**(LIPS) in $P[:j]$ for every $j$. We determine these prefixes and suffices with an "next" array $W$, defined as

$$
W[i] = \max \left \\{k | P[:k] = P[i-k:i] \right \\}
$$

For example,
```
Index: 012345
P    = abxaby
W    = 010012
```

To calculate $W$ from $P$, we use **recursion**.

Firstly, $W[0]$ is certainly $0$.

Then, assuming that we have already obtained $W[i] = k$, and there exists $P[k] = P[i]$, we can simply set $W[i + 1]$ to $k + 1$.

However, when $P[k] \ne P[i]$, we will have to look for a shorter LIPS.

```
Index: 0123456789ABCD
P    = abyabxabyabyz
       ----  ----|
       ----- -----|      <- i = B, W[B] = W[A] + 1
       ---      ---|     <- i + 1 = C
```

For example, when $i = B$, there exists LIPS "abyab", however when $i = C$, there only exists shorter "aby". Noticing that the shorter prefix, which is "aby", can be splitted into two part "ab" and "y". "ab" is actually the LIPS of "abyab", which is again the LIPS of $P[:B]$!

Therefore, all we have to do when $P[k] \ne P[i]$ is:

- Find the LIPS of $P[:k]$, length of which has been calculated and stored in $W[k] = l$.
- See if $P[l] = P[i]$. If so, $W[i + 1]$ would be $l + 1$. Otherwise, $W[i + 1]$ would be 0.

The Rust code for solving $W$ is listed below. (To simplify stuff, char codes stored in `Vec<char>` is used instead of UTF-8 encoded `String`)


```Rust
/// Wrapper for [`Vec<char>`]
struct Chars(Vec<char>);
impl From<&str> for Chars {
    fn from(s: &str) -> Self {
        Self(s.chars().collect())
    }
}
impl From<&Chars> for String {
    fn from(s: &Chars) -> Self {
        let mut st = String::new();
        for &c in s.0.iter() {
            st.push(c);
        }
        st
    }
}

struct Pattern {
    P: Chars,
    W: Vec<usize>
}

impl Pattern {
    fn from(P: &str) -> Self {
        let P = Chars::from(P);
        let mut W = Vec::with_capacity(P.0.len());
        W.resize(P.0.len(), 0);
        
        for i in 1..P.0.len() - 1 {
            let k = W[i];
            
            W[i + 1] = if P.0[i] == P.0[k] {
                k + 1
            } else {
                let l = W[k];
                if P.0[l] == P.0[i] { l + 1 } else { 0 }
            };
        }
        
        Self { P, W }
    }
}

```


```Rust
let pat = Pattern::from("abyabxabyabyz");
println!("P = {}", String::from(&pat.P));
println!("W = {:?}", pat.W);
```

    P = abyabxabyabyz
    W = [0, 0, 0, 0, 1, 2, 0, 1, 2, 3, 4, 5, 3]


## Matching

Now we have the "next" array $W$, we can do the matching.

For example,

```
Index: 0123456
S    = abyac     <- j is used to index the char to be compared in S
           -     <- Mismatch found when i = j = 4
P    = abyab     <- i is used to index the char ... in P
W    = 00001
          abyab  <- i set to W[i] = 1, and compare again
```

Now mismatch occurred when $i = j = 4$, and all we have to do is to align $P$ again.




```Rust
impl Pattern {
    fn match_str(&self, s: &str) -> Option<usize> {
        let mut i = 0;
        let mut j = 0;
        let mut chars = s.chars();
        let mut c = chars.next();
        
        loop {
            if i >= self.P.0.len() {
                return Some(j - self.P.0.len());
            }
            
            match c {
                None => return None,
                Some(chr) => {
                    if self.P.0[i] == chr {
                        i += 1;
                        c = chars.next();
                        j += 1;
                    } else {
                        if i == 0 {
                            c = chars.next();
                            j += 1;
                        }  // if P[0] mismatched, try j + 1
                        else { i = self.W[i]; }
                    }
                }
            }
        }
    }
}
```


```Rust
let idx = pat.match_str("ababyyabyabxaabyabxabyabyzab");
//                                    -------------
println!("Match found at {}", idx.unwrap());
```

    Match found at 13

