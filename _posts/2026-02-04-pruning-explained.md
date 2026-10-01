---
layout: post
title: "Pruning: cutting what a network doesn't need"
date: 2026-02-04 10:00:00+0900
description: Most trained networks carry weight they don't need. From magnitude pruning to the Lottery Ticket Hypothesis — what gets removed, and why fewer weights don't automatically mean faster inference.
tags: model-compression pruning efficiency
categories: research-notes
series: Efficient Inference
related_posts: false
toc:
  sidebar: right
---

<div class="papers">
  <div class="paper">
    <span class="paper-venue">NeurIPS<br>2015</span>
    <p class="paper-title"><a href="https://arxiv.org/abs/1506.02626">Learning both Weights and Connections for Efficient Neural Networks</a></p>
    <p class="paper-authors">S. Han, J. Pool, J. Tran, W. J. Dally</p>
  </div>
  <div class="paper">
    <span class="paper-venue">ICLR<br>2019</span>
    <p class="paper-title"><a href="https://arxiv.org/abs/1803.03635">The Lottery Ticket Hypothesis: Finding Sparse, Trainable Neural Networks</a></p>
    <p class="paper-authors">J. Frankle, M. Carbin</p>
  </div>
</div>

<div class="tldr" markdown="1">
<span class="label">TL;DR</span>

- Han et al.'s **train → prune → fine-tune** loop shrinks AlexNet **9×** and VGG-16 **13×** with no reported accuracy loss.
- But magnitude pruning yields **unstructured** sparsity — a smaller checkpoint, not a faster one, unless kernels or hardware know to skip the zeros.
- The **Lottery Ticket Hypothesis** shows sparse subnetworks, reset to their original initialization, can train to match the dense network — turning pruning into a question about trainability.
</div>

## The simplest recipe that works

Most trained networks are carrying weight they don't need. Pruning is the oldest compression idea there is: find the weights, channels, or heads that contribute the least, and cut them.

Han et al.'s version is about as simple as it gets. Train to convergence, drop every weight whose magnitude falls below a threshold, fine-tune what's left to recover accuracy — and optionally go around again.

<div class="flow">
  <div class="flow-node"><strong>Train</strong>to convergence</div>
  <div class="flow-arrow">→</div>
  <div class="flow-node hot"><strong>Prune</strong>drop small-magnitude weights</div>
  <div class="flow-arrow">→</div>
  <div class="flow-node"><strong>Fine-tune</strong>recover accuracy</div>
  <div class="flow-arrow">↻</div>
  <div class="flow-node"><strong>Repeat</strong>optional</div>
</div>
<p class="flow-caption">Magnitude pruning: the entire method fits in one loop.</p>

The numbers are striking for something this plain:

<div class="stats">
  <div class="stat"><p class="stat-value">9×</p><p class="stat-label">AlexNet: 61M → 6.7M parameters</p></div>
  <div class="stat"><p class="stat-value">13×</p><p class="stat-label">VGG-16: 138M → 10.3M parameters</p></div>
</div>

Both come with no reported accuracy loss. That sounds almost too easy — and the catch is hiding in *what kind* of sparsity this produces.

## Smaller is not the same as faster

Magnitude is a per-weight criterion, so it has no opinion about shape. The result is **unstructured** sparsity: zeros scattered across the matrix with no consistent pattern.

<div class="callout warn" markdown="1">
**A 90%-sparse matrix is not 90% cheaper to multiply.** It has 90% fewer nonzero entries, but a standard dense-matmul kernel doesn't know to skip them — it multiplies through the zeros exactly like everything else. Without sparse-aware kernels or hardware, a smaller checkpoint doesn't translate into a faster one.
</div>

**Structured** pruning sidesteps the problem by removing whole channels, heads, or layers instead of individual weights. What's left is a genuinely smaller *dense* matrix, which ordinary hardware already multiplies faster.

<div class="compare" markdown="1">
<div class="compare-card" markdown="1">
<span class="label">Unstructured</span>
#### Remove individual weights

- Fine-grained: can push sparsity very far
- Scattered zeros with no consistent structure
- Speedup needs sparse-aware kernels or hardware
</div>
<div class="compare-card" markdown="1">
<span class="label">Structured</span>
#### Remove channels, heads, layers

- Leaves a smaller *dense* matrix
- Faster on ordinary hardware out of the box
- Coarser units, so accuracy breaks at lower sparsity
</div>
</div>

## The subnetwork that was there all along

Frankle and Carbin asked a sharper question about results like Han et al.'s. Is the sparse subnetwork itself special — or would *any* sparse mask of the same size do just as well if retrained from scratch?

Their experiment separates the mask from the weights:

1. **Train and prune.** Obtain a sparse subnetwork the usual way.
2. **Rewind the weights.** Instead of keeping the trained values, reset every surviving weight to its *original* initialization.
3. **Retrain the subnetwork.** These "winning tickets" match or beat the full dense network's accuracy — and train *faster* — even at under 10–20% of the original parameter count.
4. **Control: re-initialize randomly.** The same mask with a fresh random initialization usually can't pull this off.
{: .steps}

<div class="callout" markdown="1">
**The mask alone isn't the secret.** Because random re-initialization fails, both the sparse structure *and* the specific initial values pruning happened to land on are doing real work.
</div>

<div class="pullquote" markdown="1">
Pruning stopped being just a way to shrink a trained network, and became a question about which sparse subnetworks are *trainable* in the first place.
</div>

That reframing is what a whole line of later work is really chasing: finding such subnetworks earlier in training, rather than only after the expensive dense run is done.

## Beyond magnitude

For all its simplicity, magnitude pruning only ever looks at a weight's own size — never its actual effect on the loss. A small weight can still matter a lot; a large one can be surprisingly redundant.

<div class="callout note" markdown="1">
**The more careful lineage is older than you'd think.** Loss-aware criteria trace back to LeCun, Denker, and Solla's *Optimal Brain Damage* (1989), which uses gradient or Hessian information to estimate what removing a weight really costs. Structured variants of the same idea apply it to whole channels, using calibration-data statistics rather than raw magnitude.
</div>

<div class="takeaways" markdown="1">
<span class="label">Key takeaways</span>

- **Train → prune → fine-tune** is enough to shrink AlexNet **9×** and VGG-16 **13×** with no reported accuracy loss.
- **Unstructured** sparsity shrinks the checkpoint; **structured** sparsity shrinks the dense compute — only the latter is fast on ordinary hardware.
- **Winning tickets** — sparse masks rewound to their original initialization — train to match the dense network at under 10–20% of its parameters.
- Magnitude ignores the loss; **loss-aware criteria** going back to *Optimal Brain Damage* (1989) ask what removing a weight actually costs.
</div>
