---
layout: post
title: "In-context learning, without a gradient step"
date: 2025-08-09 10:00:00+0900
description: Nothing in the weights changes, yet more demonstrations still help. Xie et al. (ICLR 2022) argue that in-context learning is implicit Bayesian inference over a latent concept.
tags: uncertainty in-context-learning theory
categories: paper-notes
series: Uncertainty in In-Context Learning
related_posts: false
toc:
  sidebar: right
---

<div class="papers">
  <div class="paper">
    <span class="paper-venue">ICLR<br>2022</span>
    <p class="paper-title"><a href="https://arxiv.org/abs/2111.02080">An Explanation of In-Context Learning as Implicit Bayesian Inference</a></p>
    <p class="paper-authors">S. M. Xie, A. Raghunathan, P. Liang, T. Ma</p>
  </div>
</div>

<div class="tldr" markdown="1">
<span class="label">TL;DR</span>

- In-context learning gets better with more demonstrations even though **no gradient ever touches the weights**. That should be more puzzling than it usually is.
- Xie et al. pretrain on a **mixture of HMMs, one per latent document concept**, and prove that next-token prediction then amounts to **implicit Bayesian inference over the concept**.
- A few-shot prompt reads like a partial document. Each demonstration **sharpens the posterior** over concepts, and from the outside that looks like learning.
</div>

## The puzzle: learning with frozen weights

In-context learning should bother us more than it does. Between reading the demonstrations and producing an answer, nothing in the model is updated. By the usual definition, nothing is being learned.

Yet accuracy keeps climbing as you add examples, which is exactly what learning looks like. Xie et al. set out to find a pretraining story in which this stops being a mystery and becomes something you would expect.

<div class="stats">
  <div class="stat"><p class="stat-value">0</p><p class="stat-label">gradient updates between seeing the demonstrations and answering</p></div>
  <div class="stat"><p class="stat-value">1</p><p class="stat-label">forward pass: all of the "learning" happens inside it</p></div>
  <div class="stat"><p class="stat-value">More shots</p><p class="stat-label">still means better accuracy, just as if the model were being trained</p></div>
</div>

## A pretraining world built for a proof

To prove anything, they need a pretraining distribution simple enough to analyze. They pick a **mixture of Hidden Markov Models**, with one HMM for each latent "document concept."

1. **Sample a concept.** It stays fixed for the whole document.
2. **Sample a long token sequence** from that concept's HMM.
3. **Train a transformer** to predict the next token on many such documents.
{: .steps}

Keeping the concept fixed is a crude stand-in for something real: actual documents tend to hold one topic or style from beginning to end. It is enough structure to prove a result.

<div class="callout" markdown="1">
**The result:** with enough pretraining data, a next-token predictor trained on this distribution ends up doing Bayesian inference over *which concept generated the document it is reading*, using only the tokens it has seen so far. Nobody built that behavior in. It falls out of next-token prediction once the data has this concept structure.
</div>

## Why a prompt looks like a document

An ICL prompt is just demonstrations stuck together. To a model pretrained this way, that looks like the start of a document whose concept it has to infer.

<div class="flow">
  <div class="flow-node"><strong>Demonstrations</strong>concatenated prompt</div>
  <div class="flow-arrow">→</div>
  <div class="flow-node"><strong>Evidence</strong>about the active concept</div>
  <div class="flow-arrow">→</div>
  <div class="flow-node hot"><strong>Sharper posterior</strong>over concepts</div>
  <div class="flow-arrow">→</div>
  <div class="flow-node"><strong>Prediction</strong>that looks "learned"</div>
</div>
<p class="flow-caption">Every step happens in one forward pass, over a belief about the concept. The weights never change.</p>

Each demonstration is evidence about which concept is active. More demonstrations narrow the posterior, and predicting under a narrower posterior is what we see from outside as "the model learned the task."

<div class="pullquote" markdown="1">
The model isn't updating its weights. It is updating a belief about which concept it is in.
</div>

## What the evidence shows

The theory comes with a synthetic testbed built to match it: **GINC**, generated from an HMM mixture that satisfies the paper's own assumptions. Real pretrained models give a second, looser check.

<div class="compare" markdown="1">
<div class="compare-card" markdown="1">
<span class="label">Synthetic</span>
#### GINC

- Built directly from the paper's HMM-mixture assumptions
- Accuracy **rises with more demonstrations**
- Holds up reasonably well when the examples are **reordered**
- Both results are what the Bayesian account predicts
</div>
<div class="compare-card" markdown="1">
<span class="label">Real models</span>
#### GPT-2 and GPT-3

- Show **similar trends with demonstration count** on natural tasks
- The authors call this **suggestive**, not proof
- Nothing shows that real LMs satisfy the formal setup
</div>
</div>

<div class="callout note" markdown="1">
The careful framing matters. The proof holds in a world the authors built. Whether real pretraining corpora look enough like an HMM mixture is still an open question.
</div>

## What "uncertain" means now

This is the part I keep coming back to. If the model is inferring a concept, then its uncertainty is simply **a posterior that hasn't concentrated yet**, and that raises a sharper question than "how confident is the model?"

<div class="callout question" markdown="1">
**Why is the posterior still spread out?** Maybe the demonstrations really are ambiguous about which concept applies, which is a fact about the *prompt*. Or maybe the evidence is clear and the model reads it badly, which is a fact about the *model*.
</div>

The paper never measures this split. What it gives me is the vocabulary to ask for it, and pulling those two sources apart is where my own work on uncertainty in in-context learning starts.

<div class="takeaways" markdown="1">
<span class="label">Key takeaways</span>

- In-context learning improves with more demonstrations **with no weight updates**. Xie et al. explain why.
- Pretraining on a **mixture of concept-specific HMMs** turns next-token prediction into **implicit Bayesian inference** over the latent concept.
- A prompt acts as a **partial document**: each demonstration sharpens the posterior over concepts inside a single forward pass.
- **GINC** matches the theory (more shots help, and reordering is tolerated). **GPT-2 and GPT-3** trends are suggestive, not proof.
- Seen this way, uncertainty is an **unconcentrated posterior**. The open question is whether the prompt or the model is to blame.
</div>
