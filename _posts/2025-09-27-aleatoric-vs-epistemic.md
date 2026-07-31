---
layout: post
title: Two ways for a model to say "I don't know"
date: 2025-09-27 10:00:00+0900
description: Kendall and Gal (NeurIPS 2017) give the standard recipe for modeling aleatoric and epistemic uncertainty jointly in a single network, on segmentation and depth regression.
tags: uncertainty bayesian-deep-learning
categories: paper-notes
related_posts: false
---

> Kendall, A., & Gal, Y. (2017). [What Uncertainties Do We Need in Bayesian Deep Learning for Computer Vision?](https://arxiv.org/abs/1703.04977) NeurIPS 2017.

A model can be unsure for two completely different reasons, and it's worth being pedantic about the difference because the fix is not the same. **Aleatoric** uncertainty is baked into the observation — sensor noise, occlusion, a genuinely ambiguous input — and no amount of extra training data touches it, because the world is what's ambiguous, not the model. **Epistemic** uncertainty lives in the model's parameters instead: it's largest on inputs unlike anything it's trained on, and it does shrink as relevant data accumulates. Kendall and Gal is the paper that made this split precise enough to actually build.

## Making the model report its own noise

For the aleatoric half, the network stops outputting a single point prediction $\hat y$ and instead outputs a prediction *and* a variance $\sigma^2(x)$ specific to that input. Plug that into a Gaussian negative log-likelihood and you get:

$$
\mathcal{L} = \frac{1}{2\sigma^2(x)} \|y - \hat y\|^2 + \frac{1}{2}\log \sigma^2(x).
$$

Nobody tells the network which inputs are noisy. The loss does it on its own: getting a "noisy" input wrong costs less once $\sigma^2(x)$ is allowed to grow, so the network learns to widen its own uncertainty exactly where it needs to, purely by minimizing this one objective.

## Making the model report its own ignorance

For epistemic uncertainty, the trick is almost absurdly cheap: leave dropout switched on at *test* time, run the same input through the network $T$ times, and look at how much the $T$ outputs disagree. Gal and Ghahramani had already shown this disagreement approximates sampling from a Bayesian posterior over the weights — so variance across those samples becomes a genuine (if approximate) measure of what the model doesn't know.

Put both pieces in one network and the total predictive variance decomposes cleanly:

$$
\text{Var}[\text{total}] = \underbrace{\mathbb{E}[\sigma^2(x)]}_{\text{aleatoric}} + \underbrace{\text{Var}[\hat y]}_{\text{epistemic, across MC samples}}
$$

Two numbers, estimated jointly, not two disconnected pipelines bolted together after the fact.

## What actually shows up in the results

Tested on semantic segmentation (CamVid, Cityscapes) and depth regression (Make3D, NYUv2):

| Finding | Detail |
|---|---|
| Where aleatoric uncertainty is largest | Object boundaries, small or distant objects — regions genuinely ambiguous from one image |
| Where epistemic uncertainty is largest | Inputs and classes underrepresented in training |
| Combined vs. either alone | Better calibrated, lower predictive loss |
| Effect of more training data | Aleatoric uncertainty comes to dominate; epistemic's share shrinks — exactly what the definitions predict |

None of the machinery here — the heteroscedastic loss, MC dropout — transfers directly to a setting like in-context learning, where there's no retraining loop to sample a weight posterior from. But the decomposition itself, and the finding that conflating the two produces a worse-calibrated model than separating them, is the part that later NLP and ICL uncertainty work keeps importing wholesale, even after throwing out everything else.
