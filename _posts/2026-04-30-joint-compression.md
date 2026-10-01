---
layout: post
title: "Why joint compression beats doing it in sequence"
date: 2026-04-30 10:00:00+0900
description: Pruning and quantization interact. APQ (CVPR 2020) stops pretending they don't — and the hard part turns out to be making the joint search cheap, not smart.
tags: model-compression joint-optimization efficiency
categories: research-notes
series: Efficient Inference
related_posts: false
toc:
  sidebar: right
---

<div class="papers">
  <div class="paper">
    <span class="paper-venue">CVPR<br>2020</span>
    <p class="paper-title"><a href="https://arxiv.org/abs/2006.08509">APQ: Joint Search for Network Architecture, Pruning and Quantization Policy</a></p>
    <p class="paper-authors">Tianzhe Wang, Kuan Wang, Han Cai, Ji Lin, Zhijian Liu, Song Han</p>
  </div>
</div>

<div class="tldr" markdown="1">
<span class="label">TL;DR</span>

- Running architecture → pruning → quantization **one stage at a time** hides the interaction between them — each stage optimizes blind to the next.
- APQ searches all three **as one joint space**, and makes that affordable with a once-for-all network plus an accuracy predictor that is *transferred* to the quantized setting.
- Result: **+2.3% accuracy** over the staged pipeline, and **~2× lower latency** than MobileNetV2 + HAQ.
</div>

## The pipeline everyone uses

The default recipe for compressing a network looks sensible on paper. You fix the architecture, decide how much to prune, then decide how many bits each surviving layer gets — fine-tuning in between each step.

<div class="flow">
  <div class="flow-node"><strong>Architecture</strong>pick a backbone</div>
  <div class="flow-arrow">→</div>
  <div class="flow-node"><strong>Pruning</strong>per-layer ratio</div>
  <div class="flow-arrow">→</div>
  <div class="flow-node"><strong>Quantization</strong>per-layer bits</div>
  <div class="flow-arrow">→</div>
  <div class="flow-node"><strong>Deploy</strong>fine-tuned model</div>
</div>
<p class="flow-caption">The staged pipeline: every decision is locked in before the next one is made.</p>

The problem isn't how carefully each stage is tuned. It's structural: **the pruning stage only ever sees a full-precision model.** It has no way to know which channels will matter most once quantization error enters the picture.

<div class="callout warn" markdown="1">
**Stages that can't see each other can't trade off against each other.** A layer that looks safe to prune at FP32 may be exactly the one that can't afford aggressive quantization — but by the time the quantizer finds out, the pruning decision is already made.
</div>

## Searching all three at once

APQ's move is simple to state: treat **architecture, per-layer pruning ratio, and per-layer bit-width as a single search space**, and optimize them together under one hardware budget.

<div class="flow">
  <div class="flow-node hot"><strong>Architecture</strong></div>
  <div class="flow-arrow">+</div>
  <div class="flow-node hot"><strong>Pruning</strong></div>
  <div class="flow-arrow">+</div>
  <div class="flow-node hot"><strong>Quantization</strong></div>
  <div class="flow-arrow">⇒</div>
  <div class="flow-node"><strong>One joint search</strong>under a latency / energy budget</div>
</div>
<p class="flow-caption">The joint view: one decision, one budget.</p>

The catch is cost. Evaluating a single point in this space normally means training a candidate and measuring its accuracy — and the joint space is combinatorially larger than any of the three on its own. So the real contribution is a way to make evaluation nearly free:

1. **Train one once-for-all network.** A single supernet whose sub-networks can be evaluated directly, without training each candidate from scratch.
2. **Fit a full-precision accuracy predictor.** Because sampling sub-networks is cheap, collecting (architecture, accuracy) pairs is cheap too.
3. **Transfer the predictor to the quantized setting.** Instead of building a large quantized-accuracy dataset, fine-tune the predictor with a comparatively small number of *real* quantized evaluations.
4. **Search with the predictor.** Now scoring a candidate costs a forward pass through a tiny model, and the joint space becomes searchable.
{: .steps}

<div class="callout" markdown="1">
**The clever part is the predictor transfer.** Full-precision accuracy data is abundant and cheap; quantized accuracy data is expensive. APQ spends the cheap data first and only a little of the expensive data at the end.
</div>

## What it buys

Measured on ImageNet at matched efficiency budgets:

<div class="stats">
  <div class="stat"><p class="stat-value">+2.3%</p><p class="stat-label">accuracy vs. the staged architecture → pruning → quantization pipeline</p></div>
  <div class="stat"><p class="stat-value">~2×</p><p class="stat-label">lower latency than MobileNetV2 + HAQ</p></div>
  <div class="stat"><p class="stat-value">~1.3×</p><p class="stat-label">lower energy than MobileNetV2 + HAQ</p></div>
</div>

And the search itself is substantially cheaper than earlier joint NAS + compression methods — which matters, because a joint search that costs too much to run is just a thought experiment.

<div class="pullquote" markdown="1">
A staged pipeline leaves accuracy on the table *by construction*. The joint search doesn't find a clever trick — it just gets to see the interaction.
</div>

## The bigger picture

Beyond this one paper, the argument generalizes. Once "how much to prune" and "how many bits to use" are treated as **one decision under a shared budget** rather than two independent knobs, compression becomes a constrained search problem.

<div class="compare" markdown="1">
<div class="compare-card" markdown="1">
<span class="label">Staged</span>
#### Tune each knob in turn

- Simple, modular, easy to debug
- Each stage is blind to the next
- Leaves accuracy on the table even when every stage is tuned well
</div>
<div class="compare-card" markdown="1">
<span class="label">Joint</span>
#### Search one combined space

- Sees pruning × quantization interactions
- Needs a cheap way to evaluate candidates
- Progress comes from making search *tractable*, not from a better criterion
</div>
</div>

That last line is the one I keep coming back to in my own work on joint sparsification and quantization: most of the gain doesn't come from a smarter pruning score or a smarter rounding rule in isolation. It comes from letting the two decisions see each other at all.

<div class="takeaways" markdown="1">
<span class="label">Key takeaways</span>

- Pruning and quantization are **entangled** — optimizing them in sequence cannot capture that.
- APQ makes a joint search affordable with a **once-for-all network** and a **transferred accuracy predictor**.
- **+2.3%** over the staged pipeline; **~2× faster** and **~1.3× more energy-efficient** than MobileNetV2 + HAQ.
- The hard part of joint compression is **search cost**, not the criterion.
</div>
