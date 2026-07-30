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

**Definitions used in the paper.** *Aleatoric* uncertainty is uncertainty inherent to the observation (sensor noise, occlusion, inherently ambiguous input) — it does not shrink with more training data. *Epistemic* uncertainty is uncertainty in the model's parameters — it does shrink with more relevant training data, and is largest on inputs unlike anything in the training distribution.

**Modeling aleatoric uncertainty: heteroscedastic loss.** Instead of a network that outputs only a point prediction, the network is modified to output both a prediction and a predicted variance for that specific input. For regression, the negative log-likelihood loss under a Gaussian with input-dependent variance naturally penalizes overconfident wrong answers less on inputs the network flags as noisy, and forces confident correctness on inputs it does not — the variance head is trained end-to-end with no separate supervision for what "noisy" means, purely from the interaction between the two output heads in this loss.

**Modeling epistemic uncertainty: Monte Carlo dropout.** The network keeps dropout active at test time (not just during training) and is run T times on the same input, producing T slightly different outputs due to the different dropout masks sampled each pass. Gal and Ghahramani's earlier result (which this paper builds on) shows this is a valid approximate sample from a Bayesian posterior over network weights. The variance across the T outputs is the epistemic uncertainty estimate.

**Combining both.** The paper's combined model has both a heteroscedastic aleatoric variance head and MC-dropout sampling at test time; the total predictive variance is derived as the sum of the expected aleatoric variance and the variance of the mean predictions across MC samples — i.e., the two uncertainty types add rather than being estimated by two disconnected pipelines.

**Tasks and results.** The method is evaluated on semantic segmentation (CamVid, Cityscapes) and monocular depth regression (Make3D, NYUv2). Reported findings include: aleatoric uncertainty is largest at object boundaries and on small/distant objects (regions genuinely ambiguous from a single image, matching the definition), while epistemic uncertainty is largest on inputs and object classes underrepresented in the training set; combining both uncertainty types gives a better-calibrated and lower predictive loss than modeling either alone; and for large datasets and large models, aleatoric uncertainty dominates the total predictive uncertainty and becomes near-essential to model explicitly, while epistemic uncertainty's relative contribution shrinks as training data grows — consistent with the definitions (epistemic uncertainty should shrink with more data; aleatoric should not).

**Relevance beyond computer vision.** The paper's tasks are vision-specific, but the decomposition — and specifically the finding that conflating the two uncertainty types produces worse-calibrated models than modeling them jointly — is the template that later uncertainty-quantification work in NLP and in-context learning explicitly imports, even though the original heteroscedastic-loss and MC-dropout machinery doesn't transfer directly to autoregressive generation or to the no-retraining setting of ICL.
