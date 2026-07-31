---
layout: post
title: Learning without a gradient step
date: 2025-08-09 10:00:00+0900
description: Xie, Raghunathan, Liang, and Ma (ICLR 2022) give a formal Bayesian account of why in-context learning improves with more demonstrations without any gradient update.
tags: uncertainty in-context-learning theory
categories: paper-notes
related_posts: false
---

> Xie, S. M., Raghunathan, A., Liang, P., & Ma, T. (2022). [An Explanation of In-Context Learning as Implicit Bayesian Inference](https://arxiv.org/abs/2111.02080). ICLR 2022.

Here's the thing about in-context learning that should bother you more than it usually does: nothing is being learned, in the sense we normally mean it. No gradient touches the weights between seeing the demonstrations and producing an answer. And yet performance keeps improving as you add more examples, which is exactly what learning is supposed to look like. Xie et al. wanted a pretraining story under which this isn't a mystery but an inevitability.

## A pretraining world built to make the theory provable

They construct a synthetic pretraining distribution out of a mixture of Hidden Markov Models, one HMM per latent "document concept." Generating a document means: sample a concept, then sample a long token sequence from that concept's HMM, keeping the concept fixed for the whole document — a crude but workable stand-in for the fact that real documents keep a consistent topic or style from start to finish.

Under this setup, they can actually prove something: a transformer trained to predict the next token, given enough pretraining data, ends up implicitly performing Bayesian inference over *which concept generated the document it's currently reading*, using only the tokens seen so far. Nobody engineered that behavior in. It falls out of next-token prediction once the pretraining distribution has this document-concept structure.

## Why that explains few-shot prompting

An ICL prompt is just a concatenation of demonstrations, and to a model trained this way, it looks like a partial document. Each demonstration is evidence about which concept is active; more demonstrations narrow the posterior over concepts further; and predicting under a sharper posterior is what, from the outside, reads as "the model learned the task." All of it happens inside a single forward pass, over a belief the model holds about which concept applies — never over the weights themselves.

The theory gets a synthetic testbed to match: **GINC**, built directly from an HMM mixture satisfying the paper's own assumptions. On it, accuracy climbs with more demonstrations and holds up reasonably well under reordering — both exactly what the Bayesian account predicts. Real pretrained models (GPT-2, GPT-3) show similar demonstration-count trends on natural tasks, though the authors are careful to call this suggestive rather than proof that real LMs satisfy their formal setup.

What I keep coming back to is the shift in what "uncertain" even means once you buy this framing. If the model is inferring a concept, then uncertainty is just a posterior that hasn't concentrated — and now there's a real question to ask: is the posterior spread out because the demonstrations themselves are genuinely ambiguous about which concept applies, or because the model's read of clear evidence is bad? The first is a fact about the prompt. The second is a fact about the model. Nothing in the paper measures this split — it just hands you the vocabulary to ask for it.
