---
layout: post
title: "Why joint compression beats doing it in sequence"
date: 2026-04-30 10:00:00+0900
description: APQ (Wang et al., CVPR 2020) shows what changes when architecture, pruning, and quantization are searched for jointly instead of tuned one after another.
tags: model-compression joint-optimization efficiency
categories: research-notes
related_posts: false
---

> Wang, T., Wang, K., Cai, H., Lin, J., Liu, Z., & Han, S. (2020). [APQ: Joint Search for Network Architecture, Pruning and Quantization Policy](https://arxiv.org/abs/2006.08509). CVPR 2020.

The default way to combine pruning and quantization is to stage them: pick an architecture, decide a pruning policy for it, then decide a quantization policy for whatever survived pruning, fine-tuning in between. It's a reasonable-sounding pipeline, and it has a structural flaw that has nothing to do with how carefully you tune each stage. A pruning stage that only ever sees the full-precision model has no way to know which channels will turn out to matter most once quantization error enters the picture — the two decisions are entangled in reality, but the pipeline resolves them as if they weren't.

## Searching all three at once, cheaply

APQ's answer is to stop staging and treat architecture choice, per-layer pruning ratio, and per-layer bit-width as one combined space to search jointly. The obvious problem is cost — naively evaluating a single point in that space means training a candidate and measuring its accuracy, and the joint space dwarfs any one of the three sub-spaces on its own.

Their way around it is a two-step trick with the training cost pushed as far to the side as possible. First, train one accuracy predictor for full-precision architectures using a "once-for-all" network — built so many sub-architectures can be evaluated without training each one separately, so the predictor itself is nearly free per candidate. Then transfer that predictor into a quantization-aware version, using a comparatively small number of *actual* quantized evaluations rather than building a large quantized-accuracy dataset from scratch.

## What that's worth

On ImageNet, at matched efficiency budgets:

| Comparison | Result |
|---|---|
| vs. staged search (architecture → pruning → quantization) | +2.3% accuracy |
| vs. MobileNetV2 + HAQ | ~2x lower latency, ~1.3x lower energy |
| Search cost | Substantially below prior joint NAS+compression methods |

None of this should read as "joint search discovered a clever trick." A staged pipeline is leaving accuracy on the table *by construction* — the pruning decision literally cannot see the quantization policy that comes after it, so it has no way to account for the interaction between the two. A joint search, even an approximate one, at least has access to that interaction. The actual engineering work in the paper isn't a smarter pruning or quantization rule; it's making a combinatorially larger joint space cheap enough to search at all.

That's the general shape of the argument, beyond this one paper: once "how much to prune" and "how many bits to use" get treated as one decision under a shared budget instead of two independent knobs, compression turns into a constrained search problem, and most of the progress comes from making that search tractable rather than from a better criterion for either piece in isolation.
