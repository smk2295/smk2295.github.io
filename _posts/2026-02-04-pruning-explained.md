---
layout: post
title: "Pruning: removing what a network doesn't need"
date: 2026-02-04 10:00:00+0900
description: From Optimal Brain Damage to the Lottery Ticket Hypothesis — what the pruning literature actually removes, and why unstructured sparsity doesn't automatically buy speed.
tags: model-compression pruning efficiency
categories: research-notes
related_posts: false
---

> Han, S., Pool, J., Tran, J., & Dally, W. J. (2015). [Learning both Weights and Connections for Efficient Neural Networks](https://arxiv.org/abs/1506.02626). NeurIPS 2015.
>
> Frankle, J., & Carbin, M. (2019). [The Lottery Ticket Hypothesis: Finding Sparse, Trainable Neural Networks](https://arxiv.org/abs/1803.03635). ICLR 2019.

Most trained networks are carrying weight they don't need. Pruning is the oldest compression idea there is: figure out which weights, channels, or heads contribute the least, and cut them.

## The recipe, and what it actually buys you

Han et al.'s version is about as simple as it gets: train to convergence, drop every weight below a magnitude threshold, fine-tune what's left to recover accuracy, and optionally repeat.

| Network | Before | After | Reduction |
|---|---:|---:|---:|
| AlexNet | 61M params | 6.7M params | 9x |
| VGG-16 | 138M params | 10.3M params | 13x |

Both with no reported accuracy loss — which sounds almost too easy, and the catch is in what kind of sparsity this produces. Magnitude is a per-weight criterion, so it doesn't care about shape: the result is **unstructured** sparsity, a scattered pattern of zeros with no consistent structure. A 90%-sparse matrix like this has 90% fewer nonzero entries, but a standard dense-matmul kernel doesn't know to skip them — it multiplies through the zeros exactly like everything else. Without sparse-aware kernels or hardware, the smaller checkpoint doesn't translate into a faster one.

**Structured** pruning sidesteps this by removing whole channels, heads, or layers instead of scattered individual weights. The payoff is a genuinely smaller *dense* matrix that ordinary hardware already multiplies faster. The price is that you can't push sparsity nearly as far before accuracy starts to break, since removal now happens in much coarser units.

## The subnetwork that was there all along

Frankle and Carbin asked a sharper question about results like Han et al.'s: is the sparse subnetwork itself special, or would any sparse mask of the same size do just as well if you retrained it from scratch?

Their test: take a pruned subnetwork, but instead of keeping its trained weights, reset every remaining weight back to its *original* pretraining initialization, then train from there. These "winning tickets" train to match or beat the full dense network's accuracy, and do it *faster* — even at under 10–20% of the original parameter count. The same mask with a fresh *random* initialization usually can't pull this off, which is the interesting part: it means both the sparse structure and the specific initial values pruning happened to land on are doing real work, not the mask shape alone.

That result quietly changed what pruning is for. It stopped being just a way to shrink an already-trained network, and became a question about which sparse subnetworks are trainable in the first place — which is the question a whole line of later work on finding such subnetworks earlier in training is really trying to answer.

Magnitude pruning, for all its simplicity, only ever looks at a weight's own size, never its actual effect on the loss. The more careful criteria — tracing back to LeCun, Denker, and Solla's *Optimal Brain Damage* from 1989 — use gradient or Hessian information instead, and the structured versions of that idea apply it to whole channels using calibration-data statistics rather than raw magnitude.
