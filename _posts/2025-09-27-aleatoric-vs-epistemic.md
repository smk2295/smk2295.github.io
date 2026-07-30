---
layout: post
title: Paper notes — What Uncertainties Do We Need in Bayesian Deep Learning?
date: 2025-09-27 10:00:00+0900
description: Kendall and Gal (NeurIPS 2017) give the standard recipe for modeling aleatoric and epistemic uncertainty jointly in a single network, on segmentation and depth regression.
tags: uncertainty bayesian-deep-learning
categories: paper-notes
related_posts: false
---

**Kendall, A., & Gal, Y. (2017). [What Uncertainties Do We Need in Bayesian Deep Learning for Computer Vision?](https://arxiv.org/abs/1703.04977) NeurIPS 2017.**

## Two Kinds of "Uncertain"

- **Aleatoric** uncertainty is inherent to the observation itself — sensor noise, occlusion, genuinely ambiguous input. It does not shrink with more training data, because the ambiguity isn't in the model.
- **Epistemic** uncertainty lives in the model's parameters. It does shrink with more relevant training data, and is largest on inputs unlike anything in the training distribution.

## Modeling Aleatoric Uncertainty: A Heteroscedastic Loss

Instead of a network that outputs only a point prediction $\hat y$, the network outputs both a prediction and a predicted variance $\sigma^2(x)$ for that specific input. For regression, the negative log-likelihood under a Gaussian with input-dependent variance is

$$
\mathcal{L} = \frac{1}{2\sigma^2(x)} \|y - \hat y\|^2 + \frac{1}{2}\log \sigma^2(x).
$$

This loss naturally penalizes overconfident wrong answers less on inputs the network flags as noisy (large $\sigma^2$), while forcing confident correctness on inputs it doesn't. No separate supervision for "what counts as noisy" is needed — the variance head is trained end-to-end, purely from this loss.

## Modeling Epistemic Uncertainty: Monte Carlo Dropout

The network keeps dropout **active at test time** (not just during training) and is run $T$ times on the same input, producing $T$ slightly different outputs from the different dropout masks. Gal and Ghahramani's earlier result — which this paper builds on — shows this is a valid approximate sample from a Bayesian posterior over network weights. The variance across the $T$ outputs is the epistemic uncertainty estimate.

## Combining Both

The full model has both the heteroscedastic variance head and MC-dropout sampling at test time. Total predictive variance is derived as:

$$
\text{Var}[\text{total}] = \underbrace{\mathbb{E}[\sigma^2(x)]}_{\text{aleatoric}} + \underbrace{\text{Var}[\hat y]}_{\text{epistemic, across MC samples}}
$$

The two uncertainty types add — they are estimated jointly in one model, not by two disconnected pipelines.

## Tasks and Results

Evaluated on semantic segmentation (CamVid, Cityscapes) and monocular depth regression (Make3D, NYUv2):

| Finding | Detail |
|---|---|
| Aleatoric uncertainty location | Largest at object boundaries and on small/distant objects — regions genuinely ambiguous from a single image |
| Epistemic uncertainty location | Largest on inputs and object classes underrepresented in training |
| Combined vs. either alone | Better-calibrated, lower predictive loss |
| Effect of scale | As training data grows, aleatoric uncertainty dominates and epistemic uncertainty's relative share shrinks — consistent with the definitions |

## Takeaways

1. Conflating the two uncertainty types produces worse-calibrated models than modeling them jointly, even though modeling both costs little extra machinery.
2. The heteroscedastic loss and MC-dropout mechanics here are vision-specific, but the *decomposition itself* — and the finding that it matters — is the template later NLP and in-context-learning uncertainty work explicitly imports.
3. Neither mechanism transfers directly to autoregressive generation or to a no-retraining setting like ICL, which is exactly the gap that later work (including the paper this reading list supports) has to fill with a different estimator.
