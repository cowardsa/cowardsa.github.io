---
layout: distill
title: Are Datapath Circuit Transformations just Compiler Optimisations?
date: 2026-04-17 11:12:00-0400
description: An investigation into whether carry-save arithmetic can be viewed as loop fusion.
giscus_comments: true
tags: optimization


authors:
  - name: Nate Young
    url: "https://natetyoung.github.io/"
    affiliations:
      name: Cornell
  - name: Samuel Coward
    url: "https://samuelcoward.co.uk/"
    affiliations:
      name: University College London

# bibliography: parabit.bib

toc:
  - name: Carry-Save Addition
  - name: Loop Fusion
  - name: The Challenge
  - name: The Proof
  - name: Conclusion

---

During a visit to Cornell, I presented a bunch of hardware optimisations to fuse arithmetic operators. Being from a compiler background, Nate asked whether we could view these as more traditional compiler transformations.

TLDR: Yes, you can, but it gets pretty ugly…

In this blog, we’re going to show how we answered this question for a specific example, namely discovering carry-save addition (a hardware optimisation) via loop fusion (a compiler optimisation). 

## Carry-Save Addition
Consider the problem of adding up three n-bit bitvectors `(x+y+z)`. Assuming we know how to add two bitvectors, the simplest solution is to add `x` and `y`, then add `z` to the result. However, each time we add two bitvectors, we have to propagate the carries all the way from the least-significant to the most significant bit.


The time to perform this computation unfortunately scales at least logarithmically with the width of the bitvectors. In hardware we can go much faster and compute `(x+y+z)` in the same time as `(x+y)` plus a small constant overhead. To achieve this, we use a full-adder which takes three individual bits of equal weight (`x[i], y[i], z[i]`) and produces two bits `s` and `c` such that: `2*c + s = x[i] + y[i] + z[i]`. In terms of gates:
```
c = MAJ(x[i], y[i], z[i])
  = (x[i] AND y[i]) XOR (x[i] AND z[i]) XOR (y[i] AND z[i])
s = x[i] XOR y[i] XOR z[i]
```

Using n of these full-adders operating in parallel on bits of equal weight, we can take our three addends and reduce them down to two. Consider this example:
```
  1010 = 10 (x)
  1011 = 11 (y)
 +0011 =  3 (z)
 ----------
  0010 =  2
+10110 = 22
 ----------
 11000 = 24
```
Here we first reduced three values to two which takes constant time as we only look locally, so it is much faster than the complete addition. We of course do have to pay the price for the slow addition of two bitvectors once, but that’s much better than doing it twice sequentially. This technique is known as carry-save addition (and it generalises to an arbitrary number of addends).

## Loop Fusion
Now we’ll briefly describe a compiler concept, loop fusion, which we’ll later use to “rediscover” carry-save addition. Consider the following example containing two loops:
```python
# Loop 1: Square each element
for i in range(n):
    a[i] = a[i] * a[i]

# Loop 2: Add 1 to each element
for i in range(n):
    a[i] = a[i] + 1
```

A classic compiler optimisation is to fuse these two loops together, so that you only iterate over `a` once:
```python
for i in range(n):
    a[i] = a[i] * a[i] + 1
```

This is more efficient because we only execute the loop once in the program and have also been able to fuse operations together to eliminate some loads/stores. Nate’s question was, can we use loop fusion as a lens through which we can discover carry-save addition?

## The Challenge
Consider the following program fragment, which takes in three arrays of booleans `x`, `y`, and `z`, and produces an array of booleans `s` (`= x+y+z`):

```python
# Perform (x+y) - store result in w
a[0] = 0
for i in range(n):
    w[i],a[i+1] = FullAdd(x[i], y[i], a[i])
# Perform (w+z) - store result in s
b[0] = 0
for i in range(n):
    s[i],b[i+1] = FullAdd(w[i], z[i], b[i])
```

It should be easy to see that this program adds its three bitvector inputs together using two separate additions that are performed sequentially. We’ve implemented bitvector addition as a sequence of full-adders (`FullAdd`) each feeding their carry to the next full-adder. So the question is, what happens when we perform loop fusion?

```python
a[0] = 0
b[0] = 0
for i in range(n):
    w[i],a[i+1] = FullAdd(x[i], y[i], a[i])
    s[i],b[i+1] = FullAdd(w[i], z[i], b[i])
```

This isn't that different (although we might save some memory if we executed this sequentially), but now consider a fused version of the adders with the carry-save trick:

```python
c[0] = 0
d[0] = 0
for i in range(n):
    w*[i],c[i+1] = FullAdd( x[i], y[i], z[i])
    s*[i],d[i+1] = FullAdd(w*[i], c[i], d[i])
```

The two programs above at least *look* very similar; do they produce the same final output (does `s=s*`)? We know (from global analysis of the carry-save trick) that they do, but is there a **local** argument? That is, can we show that these two programs are equivalent without appealing to the actual *meaning* of `s` and `s*` as bitvectors, just by manipulating values? The answer is yes; we'll do the proof next.

## The Proof

Let’s do some bit-level analysis. We’ll do all our analysis in $\mathbb{F}_2$ (so XOR will be addition, and AND will be multiplication). Note that `MAJ(x, y, z) = xy + yz + xz` (this is usually defined where + is OR, but XOR works just as well).


From the first program:
```
s[i] = w[i] + z[i] + b[i]
     = x[i] + y[i] + a[i] + z[i] + b[i]
```
From the second program:

```
s*[i] = w*[i] + c[i] + d[i]
      = x[i] + y[i] + z[i] + c[i] + d[i]
```

So we can see that these `s` and `s*` values are equivalent if `a[i] + b[i] = c[i] + d[i]`. We're going to prove this by induction. The base case `a[0] + b[0] = c[0] + d[0]` is true since `a[0] = b[0] = c[0] = d[0] = 0`. For the inductive case we'll assume `a[i] + b[i] = c[i] + d[i]`.

### Step 1: a[i+1] + b[i+1] = c[i+1] + MAJ(x[i] + y[i] + z[i], a[i], b[i])

``` 
  a[i+1] + b[i+1]
  = MAJ(x[i], y[i], a[i]) xor MAJ(w[i], z[i], b[i])
  = MAJ(x[i], y[i], a[i]) xor MAJ(x[i] + y[i] + a[i], z[i], b[i])
```

For simplicity, let’s drop the `[i]` subscripts.
```
= MAJ(x, y, a) + MAJ(x + y + a, z, b)
= xy + ya + xa + (x + y + a)z + zb + (x + y + a)b
= xy + ya + xa + xz + yz + az + zb + xb + yb + ab
= xy + xz + yz + (x + y + z)a + (x + y + z)b + ab
= MAJ(x, y, z) + MAJ(x + y + z, a, b)
= c[i+1]       + MAJ(x + y + z, a, b)
```


It remains to show that `d[i+1]=MAJ(x[i] + y[i] + z[i], a[i], b[i])`, which will be easy to show if we also prove by induction `a[i]b[i] = c[i]d[i]` (strengthening the inductive hypothesis).


### Step 2: Assuming ab=cd show that d[i+1]=MAJ(x + y + z, a, b):
    
```
MAJ(x + y + z, a, b) = (x + y + z)a + ab + (x + y + z)b
                     = (x + y + z)(a + b) + ab
                     = (x + y + z)(c + d) + cd
                     = (x + y + z)c + (x + y + z)d + cd
                     = MAJ(x + y + z, c, d)
                     = MAJ(w*, c, d)
                     = d[i+1]
```

So lastly let’s prove `a[i]b[i] = c[i]d[i]` by induction as well. The base case again is trivial. The inductive case is where it gets annoying so strap in. 

### Step 3: Assuming ab=cd show that a[i+1]b[i+1]=c[i+1]d[i+1]
    
```
a[i+1]b[i+1] = MAJ(x, y, a)MAJ(w, z, b)
             = (xy + ya + xa)(wz + zb + wb)
             = xyzw + xyzb + xywb + 
               yzwa + yzab + ywab + 
               xzwa + xzab + xwab
```
Now we need to expand `w` everywhere, but that’ll be a lot of terms. To make it easier, note that in every term where `w` appears in the expression, it happens that exactly 2 of its components `{x, y, a}` appear as well, so you get something like `xyzw = xyzx + xyzy + xyza`. Two of these terms (where we used a component that already appears) are equivalent (since `xx=x` and `yy=y`) and cancel out. So basically the `w` just gets replaced by the one component that doesn't already appear in the term:
```
= xyza + xyzb + xyab + xyza + yzab + xyab + xyza + xzab + xyab
= xyzb + yzab + xyab + xyza + xzab
= xyz(a + b) + ab(xy + yz + xz)
= xyz(c + d) + cd(xy + yz + xz)
```

This looks pretty simple, so let’s turn our attention to `c[i+1]d[i+1]`:
```
c[i+1]d[i+1] = MAJ(x, y, z)MAJ(x + y + z, c, d)
             = (xy + yz + xz)(xc + yc + zc + cd + xd + yd + zd)
             = (xyzc + xycd + xyzd) + 
               (xyzc + yzcd + xyzd) + 
               (xyzc + xzcd + xyzd) 
```

Note that in that step, we neglected to write some things that immediately cancel out within individual sub-expressions, but there’s still more to cancel out between them:
```
= xyzc + xyzd + xycd + yzcd + xzcd
= xyz(c + d) + cd(xy + yz + xz) = a[i+1]b[i+1]
```
    
This is pretty gnarly, but it means at least in principle what we wanted to show: a compiler which can fuse loops and then rewrite the loop body under this kind of inductive argument can automatically discover carry-save adders.

## Conclusion

Obviously, we don't need to rediscover the carry-save trick, since we already know it.

Rather, the hope is that other similar tricks can be discovered as well. In hardware, it's easy to see how we can build a library of operator-level optimizations, and how we can build a library of bit-level optimizations for unrolled circuits, but compiler optimizations operate in a space in the middle, where the regularity inherent in the program must be taken advantage of to build global optimizations out of local transformations without unrolling. Hardware has regularity too, and we should perhaps be exploiting it so that we are not stuck with the other two choices of optimization.
