---
layout: post
title: "Quantizing LLMs: three ways to stop outliers from setting the scale"
date: 2026-03-21 10:00:00+0900
description: Naive low-bit quantization breaks on large language models because of a few huge activations. LLM.int8(), SmoothQuant, and GPTQ each take a different route around them.
tags: model-compression quantization efficiency
categories: research-notes
series: Efficient Inference
related_posts: false
toc:
  sidebar: right
---

<div class="papers">
  <div class="paper">
    <span class="paper-venue">NeurIPS<br>2022</span>
    <p class="paper-title"><a href="https://arxiv.org/abs/2208.07339">LLM.int8(): 8-bit Matrix Multiplication for Transformers at Scale</a></p>
    <p class="paper-authors">T. Dettmers, M. Lewis, Y. Belkada, L. Zettlemoyer</p>
  </div>
  <div class="paper">
    <span class="paper-venue">ICML<br>2023</span>
    <p class="paper-title"><a href="https://arxiv.org/abs/2211.10438">SmoothQuant: Accurate and Efficient Post-Training Quantization for Large Language Models</a></p>
    <p class="paper-authors">G. Xiao, J. Lin, M. Seznec, H. Wu, J. Demouth, S. Han</p>
  </div>
  <div class="paper">
    <span class="paper-venue">ICLR<br>2023</span>
    <p class="paper-title"><a href="https://arxiv.org/abs/2210.17323">GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers</a></p>
    <p class="paper-authors">E. Frantar, S. Ashkboos, T. Hoefler, D. Alistarh</p>
  </div>
</div>

<div class="tldr" markdown="1">
<span class="label">TL;DR</span>

- Plain round-to-grid quantization struggles on large transformers because a few **outlier feature dimensions** force the scale so wide that everything else loses precision.
- **LLM.int8()** routes outliers through FP16, **SmoothQuant** shifts the difficulty from activations onto weights, and **GPTQ** solves for the quantized weights instead of just rounding them.
- The lower you push the bit-width — from 8 bits toward 2 — the more **targeted machinery** you need.
</div>

## The outlier problem

The simplest quantizer maps a real value $$x$$ onto an integer grid using a single scale $$s$$:

$$
q = \text{round}(x / s), \qquad \hat x = s \cdot q
$$

That is the whole idea, and on large transformers it barely works. The culprit isn't random noise. LLMs develop specific feature dimensions whose activation magnitudes are **many times larger** than everything around them.

<div class="flow">
  <div class="flow-node hot"><strong>A few outliers</strong>huge activation magnitudes</div>
  <div class="flow-arrow">→</div>
  <div class="flow-node"><strong>Wide scale</strong>stretched to cover them</div>
  <div class="flow-arrow">→</div>
  <div class="flow-node"><strong>Crushed values</strong>ordinary values collapse onto a few grid points</div>
</div>
<p class="flow-caption">One shared scale lets a handful of dimensions dictate precision for everyone else.</p>

<div class="callout warn" markdown="1">
**The outliers aren't the ones who suffer.** Covering them is easy. The damage lands on the ordinary values, which get squeezed into a handful of grid points once $$s$$ is wide enough for the extremes.
</div>

The three papers below all refuse to let outliers set the scale for everyone else. They just disagree on how.

## LLM.int8(): give the outliers their own lane

LLM.int8() doesn't force every value through one shared 8-bit scale. It quantizes the bulk of values with a **vector-wise** scheme, so different rows and columns get their own scales.

The small set of outlier feature dimensions is pulled out entirely and computed in **FP16** through a separate mixed-precision path. Everything else stays in int8.

<div class="stats">
  <div class="stat"><p class="stat-value">&gt;99.9%</p><p class="stat-label">of values still computed in int8</p></div>
  <div class="stat"><p class="stat-value">175B</p><p class="stat-label">OPT-175B converted to Int8 with no measured performance loss vs. 16-bit</p></div>
  <div class="stat"><p class="stat-value">~2×</p><p class="stat-label">less memory — roughly half of the 16-bit original</p></div>
</div>

<div class="callout" markdown="1">
**Isolate, don't average.** Because the outliers live in specific dimensions, you can carve them out and keep the rest of the matrix multiply fully low-precision.
</div>

## SmoothQuant: move the difficulty, don't hide it

SmoothQuant starts from a blunt observation: **weights are easy to quantize, activations aren't.** So rather than special-casing outliers at runtime, it rescales the model *before* quantization happens.

1. **Divide** each troublesome activation channel by a per-channel smoothing factor.
2. **Multiply** the matching weight channel by the same factor.
3. **Quantize both.** The matrix product is mathematically unchanged, but the difficulty has migrated from activations (hard) onto weights (easy).
{: .steps}

That shift is enough to unlock full **W8A8** — weights *and* activations at 8 bits, not just weights.

<div class="stats">
  <div class="stat"><p class="stat-value">1.56×</p><p class="stat-label">speedup with negligible accuracy loss</p></div>
  <div class="stat"><p class="stat-value">~2×</p><p class="stat-label">memory reduction</p></div>
  <div class="stat"><p class="stat-value">530B</p><p class="stat-label">model served on a single node</p></div>
</div>

## GPTQ: stop rounding, start solving

GPTQ throws out the fixed-rule approach entirely. It treats quantization as a **per-layer optimization problem**: given a small calibration set, find the quantized weights that minimize that layer's output reconstruction error.

Approximate second-order information decides the order in which weights get quantized and how to compensate for each one's error. It is one-shot, with no retraining.

<div class="stats">
  <div class="stat"><p class="stat-value">3–4 bits</p><p class="stat-label">per weight for a 175B model</p></div>
  <div class="stat"><p class="stat-value">~4</p><p class="stat-label">GPU-hours to quantize it</p></div>
  <div class="stat"><p class="stat-value">~3.25×</p><p class="stat-label">measured speedup, with the model fitting on a single GPU</p></div>
</div>

<div class="callout note" markdown="1">
**It holds up surprisingly far down.** Pushed all the way to **2-bit**, GPTQ still produces a usable model — territory where naive rounding simply stops working.
</div>

## Side by side

<div class="compare" markdown="1">
<div class="compare-card" markdown="1">
<span class="label">LLM.int8()</span>
#### Separate the outliers

- Outliers in FP16, the rest in INT8
- W8 (mixed precision)
- 175B model, no degradation, ~2× less memory
</div>
<div class="compare-card" markdown="1">
<span class="label">SmoothQuant</span>
#### Rebalance before quantizing

- Rescales activations ↔ weights ahead of time
- W8A8
- 1.56× speedup, ~2× memory reduction
</div>
<div class="compare-card" markdown="1">
<span class="label">GPTQ</span>
#### Solve for the weights

- Minimizes per-layer reconstruction error
- W3–W4, down to 2-bit
- 175B model on 1 GPU, ~3.25× speedup
</div>
</div>

<div class="pullquote" markdown="1">
None of these is "quantization" as one fixed algorithm. Each found a structural weak point and built the method around it.
</div>

LLM.int8() exploits *which dimensions* are outliers. SmoothQuant exploits *which operand* is actually hard to quantize. GPTQ exploits what a layer's *own reconstruction error* looks like.

That is also roughly the order of difficulty. Going from 8 bits toward 2 demands progressively more of this kind of targeted machinery, because simple rounding gives out somewhere along the way.

<div class="takeaways" markdown="1">
<span class="label">Key takeaways</span>

- Naive quantization of LLMs fails because of **outlier activations**, not random noise.
- **LLM.int8()**: keep >99.9% of values in int8 and route outliers through FP16 — OPT-175B with no measured loss.
- **SmoothQuant**: migrate difficulty from activations to weights to get full **W8A8**.
- **GPTQ**: solve per-layer reconstruction to reach **3–4 bits**, still usable at 2-bit.
- Lower bit-widths need **more targeted machinery** aimed at a specific structural weak point.
</div>
