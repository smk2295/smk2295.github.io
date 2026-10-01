---
layout: post
title: "Two ways for a model to say \"I don't know\""
date: 2025-09-27 10:00:00+0900
description: Noise in the world and gaps in the model are different problems with different fixes. Kendall and Gal (NeurIPS 2017) show how to estimate both at once, in a single network.
tags: uncertainty bayesian-deep-learning
categories: paper-notes
series: Uncertainty in In-Context Learning
related_posts: false
toc:
  sidebar: right
---

<div class="papers">
  <div class="paper">
    <span class="paper-venue">NeurIPS<br>2017</span>
    <p class="paper-title"><a href="https://arxiv.org/abs/1703.04977">What Uncertainties Do We Need in Bayesian Deep Learning for Computer Vision?</a></p>
    <p class="paper-authors">A. Kendall, Y. Gal</p>
  </div>
</div>

<div class="tldr" markdown="1">
<span class="label">TL;DR</span>

- **Aleatoric** uncertainty lives in the data and never goes away with more training. **Epistemic** uncertainty lives in the model and shrinks as relevant data accumulates.
- Kendall and Gal capture the first with a **learned per-input variance** in the loss, and the second with **MC dropout** at test time, in one network.
- Modeling both together gives **better calibration and lower predictive loss** than either alone, on segmentation and depth regression.
</div>

## Two kinds of "unsure"

A model can be uncertain for two completely different reasons. It is worth being pedantic about which one you are looking at, because the remedy is not the same.

<div class="compare" markdown="1">
<div class="compare-card" markdown="1">
<span class="label">Aleatoric</span>
#### The world is ambiguous

- Baked into the observation: sensor noise, occlusion, a genuinely ambiguous input
- More training data does **not** reduce it
- The input is the problem, not the model
</div>
<div class="compare-card" markdown="1">
<span class="label">Epistemic</span>
#### The model hasn't seen enough

- Lives in the model's parameters
- Largest on inputs unlike anything seen in training
- **Shrinks** as relevant data accumulates
</div>
</div>

Kendall and Gal is the paper that made this split precise enough to actually build, with both quantities estimated side by side in one model.

## Learning your own noise

For the aleatoric half, the network stops returning a single point prediction $$\hat y$$. It outputs a prediction *and* a variance $$\sigma^2(x)$$ specific to that input, and both go into a Gaussian negative log-likelihood:

$$
\mathcal{L} = \frac{1}{2\sigma^2(x)} \|y - \hat y\|^2 + \frac{1}{2}\log \sigma^2(x)
$$

Nobody labels which inputs are noisy. The loss works it out: a large $$\sigma^2(x)$$ discounts the error term, while the $$\log \sigma^2(x)$$ term stops the network from inflating it everywhere.

<div class="callout" markdown="1">
**The uncertainty is learned for free.** Getting a noisy input wrong costs less once $$\sigma^2(x)$$ is allowed to grow, so the network widens its own error bars exactly where it needs to, just by minimizing one objective.
</div>

## Measuring your own ignorance

For the epistemic half, the trick is almost absurdly cheap. Gal and Ghahramani had already shown that dropout's randomness approximates sampling from a Bayesian posterior over the weights, so all you need is to keep it running at inference.

1. **Leave dropout on at test time.** Don't switch the network into deterministic mode.
2. **Run the same input $$T$$ times.** Each pass samples a slightly different set of weights.
3. **Measure the disagreement.** The variance across the $$T$$ outputs is an approximate, but genuine, estimate of what the model doesn't know.
{: .steps}

Put the two pieces in one network and the total predictive variance splits cleanly:

$$
\text{Var}[\text{total}] = \underbrace{\mathbb{E}[\sigma^2(x)]}_{\text{aleatoric}} + \underbrace{\text{Var}[\hat y]}_{\text{epistemic, across MC samples}}
$$

<div class="pullquote" markdown="1">
Two numbers, estimated jointly, rather than two disconnected pipelines bolted together after the fact.
</div>

## What the experiments show

The method is tested on **semantic segmentation** (CamVid, Cityscapes) and **depth regression** (Make3D, NYUv2). The two uncertainties show up in different places, which is exactly what their definitions predict.

| Finding | Detail |
|---|---|
| Where aleatoric is largest | Object boundaries, small or distant objects: regions that are genuinely ambiguous from a single image |
| Where epistemic is largest | Inputs and classes underrepresented in training |
| Combined vs. either alone | Better calibrated, lower predictive loss |
| More training data | Aleatoric comes to dominate; epistemic's share shrinks |

<div class="callout note" markdown="1">
**The data-scaling result is the sanity check.** As training data grows, only the epistemic share falls. If the two estimates were just picking up the same signal twice, they would move together.
</div>

## Why it still matters for ICL

None of the machinery here transfers directly to in-context learning. There is no retraining loop to sample a weight posterior from, and the heteroscedastic loss assumes you are the one training the model.

<div class="callout question" markdown="1">
**So what survives?** Not the heteroscedastic loss, and not MC dropout. What survives is the *decomposition*, and the finding that conflating the two gives a worse-calibrated model than separating them.
</div>

That is the part I keep seeing in later NLP and ICL uncertainty work. Those papers throw out nearly all of the original machinery, but they still bring in the split between noise in the data and gaps in the model almost unchanged.

<div class="takeaways" markdown="1">
<span class="label">Key takeaways</span>

- **Aleatoric** = irreducible noise in the input; **epistemic** = the model's ignorance, which shrinks with data.
- A **learned variance** in a Gaussian NLL captures aleatoric uncertainty without any noise labels.
- **MC dropout** ($$T$$ stochastic passes) approximates a weight posterior and captures epistemic uncertainty.
- Modeling both jointly is **better calibrated** with **lower predictive loss** than either alone.
- The machinery is vision-specific, but the **decomposition** is what later ICL uncertainty work keeps.
</div>
