---
layout: post
title: "Pruning: removing what a network doesn't need"
date: 2026-02-04 10:00:00+0900
description: From Optimal Brain Damage to the Lottery Ticket Hypothesis — what the pruning literature actually removes, and why unstructured sparsity doesn't automatically buy speed.
tags: model-compression pruning efficiency
categories: research-notes
related_posts: false
---

**Han, S., Pool, J., Tran, J., & Dally, W. J. (2015). [Learning both Weights and Connections for Efficient Neural Networks](https://arxiv.org/abs/1506.02626). NeurIPS 2015.**
**Frankle, J., & Carbin, M. (2019). [The Lottery Ticket Hypothesis: Finding Sparse, Trainable Neural Networks](https://arxiv.org/abs/1803.03635). ICLR 2019.**

## The Basic Method (Han et al., 2015)

1. Train a dense network to convergence.
2. Remove all weights below a magnitude threshold.
3. Retrain (fine-tune) the remaining sparse network to recover accuracy.
4. Optionally iterate the prune-then-retrain cycle.

| Network | Before | After | Reduction | Accuracy loss |
|---|---:|---:|---:|---|
| AlexNet | 61M params | 6.7M params | 9x | none reported |
| VGG-16 | 138M params | 10.3M params | 13x | none reported |

The criterion is magnitude, applied to individual weights — it says nothing about the *shape* of what's removed, which is why it produces **unstructured** sparsity: a scattered pattern of zeros with no consistent shape.

## Why Unstructured Sparsity Doesn't Automatically Mean Faster

A 90%-sparse unstructured weight matrix has 90% fewer nonzero entries, but a dense-matrix-multiply kernel on standard hardware doesn't know to skip the zeros — it multiplies through them like any other value. Realizing an actual speedup requires sparse-matrix kernels or hardware with native sparse support, neither of which is universal.

This is why **structured** pruning — removing entire channels, attention heads, or layers rather than individual weights — is more directly useful for latency:

- The result is a smaller *dense* matrix that ordinary kernels already multiply faster.
- The cost: less aggressive removal is tolerated before accuracy degrades, since removal happens in coarser units.

## The Lottery Ticket Hypothesis (Frankle & Carbin, 2019)

A follow-up question about pruning results like Han et al.'s: is the sparse subnetwork itself special, or does any sparse mask of the same size work equally well if retrained from scratch?

**The experiment.** Take a pruned subnetwork, but instead of keeping its trained weights, reset each remaining weight to its *original pretraining initialization value*, then train from there.

**The result.** These "winning tickets" — sparse subnetworks paired with their original initialization —

- train to match or exceed the accuracy of the full dense network,
- do so *faster* than the dense network,
- even at sparsity levels below 10–20% of the original parameter count.

Critically, the same sparse mask with freshly *random* re-initialization usually fails to match this performance — implying both the structure *and* the specific initial values found by pruning are doing real work, not the mask shape alone.

This reframed pruning from "a way to compress an already-trained network" into a question about which sparse subnetworks are trainable in the first place, and motivated later work on identifying such subnetworks earlier in (or even before) training.

## Beyond Plain Magnitude

Magnitude pruning is a strong, simple baseline, but it only looks at a weight's own value, not its effect on the loss. Criteria building on classic second-order pruning ideas (going back to LeCun, Denker, and Solla's *Optimal Brain Damage*, NeurIPS 1989) estimate each weight's contribution to the loss via gradient or Hessian information; structured variants apply the same idea to whole channels or heads using calibration-data activation statistics instead of magnitude.

## Takeaways

1. Magnitude pruning is simple and works well, but it produces unstructured sparsity that most hardware can't turn into real speed without special kernels.
2. Structured pruning trades some achievable sparsity for guaranteed latency gains on ordinary hardware.
3. The Lottery Ticket Hypothesis shows that a good sparse mask paired with its original initialization can match — and train faster than — the full network, which reframes pruning as a question about trainable subnetworks, not just post-hoc compression.
