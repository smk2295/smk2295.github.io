---
layout: post
title: "Why joint compression beats doing it in sequence"
date: 2026-04-30 10:00:00+0900
description: APQ (Wang et al., CVPR 2020) shows what changes when architecture, pruning, and quantization are searched for jointly instead of tuned one after another.
tags: model-compression joint-optimization efficiency
categories: research-notes
related_posts: false
---

**Wang, T., Wang, K., Cai, H., Lin, J., Liu, Z., & Han, S. (2020). [APQ: Joint Search for Network Architecture, Pruning and Quantization Policy](https://arxiv.org/abs/2006.08509). CVPR 2020.**

## The Problem With Sequential Pipelines

The default way to combine compression techniques is to apply them in stages: design or select an architecture, then decide a pruning policy for it, then decide a quantization policy for what's left, fine-tuning between stages.

Each stage makes its decision based only on the model as it exists *at that point*, with no knowledge of what the next stage will do to it. A channel judged "important enough to keep" by a pruning stage that only sees the full-precision model isn't necessarily the channel most worth keeping once quantization error is also accounted for — the two decisions are entangled, but a staged pipeline resolves them independently anyway.

## APQ's Approach: Search the Joint Space, But Make It Cheap

APQ treats architecture choice, per-layer pruning ratio, and per-layer bit-width as **one combined search space** and searches over it jointly rather than staging the three decisions.

The obstacle is cost: naively, evaluating one point in this joint space means training or fine-tuning a candidate configuration and measuring its accuracy, and the joint space is far larger than any one of the three sub-spaces alone. APQ's contribution is a way around that cost:

1. Train a single accuracy predictor for full-precision candidate architectures using a "once-for-all" network — built so many sub-architectures can be evaluated without separately training each one. This predictor itself requires no extra training cost per candidate.
2. Transfer that full-precision predictor into a **quantization-aware predictor**, using a comparatively small number of actual quantized (architecture, pruning, quantization) evaluations, rather than collecting a large quantized-accuracy dataset from scratch.

## Reported Results

On ImageNet, at a matched efficiency budget:

| Comparison | Result |
|---|---|
| vs. staged search (architecture → pruning → quantization separately) | +2.3% accuracy |
| vs. MobileNetV2 + HAQ (strong prior quantization-search method) | ~2x lower latency, ~1.3x lower energy |
| Search/data-collection cost | Substantially lower than prior joint NAS+compression approaches |

## Why the Accuracy Gain Isn't Surprising, Given the Setup

The point isn't that joint search finds some clever trick unavailable to staged pipelines — it's that a staged pipeline is leaving information on the table **by construction**. The pruning decision in a staged pipeline literally cannot see the quantization policy applied afterward, so it can't account for how the two interact. A joint search, even an approximate one, has access to that interaction.

The real engineering content of the paper is elsewhere: a joint search space is combinatorially larger, and most of APQ's method is about making evaluation of that larger space cheap enough to be tractable — not about a fundamentally different compression technique.

## Takeaways

1. Staged compression pipelines are structurally blind to interactions between stages — not just suboptimal by chance.
2. Framing "how much to prune" and "how many bits to use" as one joint decision under a shared budget turns compression into a constrained combinatorial search problem.
3. Progress in this direction usually comes from making the joint search *cheap enough to run*, not from a better pruning or quantization criterion in isolation.
