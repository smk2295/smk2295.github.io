---
layout: post
title: "Quantization: fewer bits per weight, without losing the model"
date: 2026-03-21 10:00:00+0900
description: LLM.int8(), SmoothQuant, and GPTQ — three concrete answers to the same problem, outlier activations that make naive low-bit quantization of LLMs fail.
tags: model-compression quantization efficiency
categories: research-notes
related_posts: false
---

> Dettmers, T., Lewis, M., Belkada, Y., & Zettlemoyer, L. (2022). [LLM.int8(): 8-bit Matrix Multiplication for Transformers at Scale](https://arxiv.org/abs/2208.07339). NeurIPS 2022.
>
> Xiao, G., Lin, J., Seznec, M., Wu, H., Demouth, J., & Han, S. (2023). [SmoothQuant: Accurate and Efficient Post-Training Quantization for Large Language Models](https://arxiv.org/abs/2211.10438). ICML 2023.
>
> Frantar, E., Ashkboos, S., Hoefler, T., & Alistarh, D. (2023). [GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers](https://arxiv.org/abs/2210.17323). ICLR 2023.

A basic quantizer maps a real value $x$ onto an integer grid using a scale $s$:

$$
q = \text{round}(x / s), \qquad \hat x = s \cdot q
$$

That's the whole idea, and it barely works on large transformers. The reason isn't random noise — it's **outliers**. LLMs develop specific feature dimensions with activation magnitudes many times larger than everything around them, and if $s$ has to be wide enough to cover those, every ordinary value gets crushed into a handful of grid points. Three papers, three different ways of refusing to let outliers set the scale for everyone else.

## Give the outliers their own lane

LLM.int8() doesn't force every value through one shared 8-bit scale. It quantizes the bulk of values with a vector-wise scheme and pulls the small number of outlier feature dimensions out entirely, computing just those in FP16 through a separate mixed-precision path — over 99.9% of values still end up in int8. The payoff: OPT-175B converts to Int8 and runs with no measured performance loss against the 16-bit original, at roughly half the memory.

## Move the difficulty, don't hide it

SmoothQuant's framing is blunt: weights are easy to quantize, activations aren't. So instead of special-casing outliers at runtime, it rescales *before* quantization even happens — dividing the troublesome activation channels by a per-channel factor and multiplying the matching weight channels by the same factor. The matrix product is untouched, but the difficulty has moved from activations (hard) onto weights (easy). That's enough to get full W8A8 — both weights *and* activations at 8 bits, not just weights — for roughly 1.56x speedup, about 2x less memory, and negligible accuracy loss, up to serving a 530B model on a single node.

## Stop rounding, start solving

GPTQ throws out the fixed-rule approach entirely. It treats quantization as a per-layer optimization problem: given a small calibration set, solve directly for the quantized weights that minimize that layer's output reconstruction error, using approximate second-order information to decide the order weights get quantized in and how to compensate for each one's error. One-shot, no retraining — and it still compresses a 175B model to 3–4 bits per weight in about four GPU-hours, fitting it on a single GPU with roughly 3.25x measured speedup, remaining usable even pushed down to 2-bit.

| Method | What it changes | Precision | Headline result |
|---|---|---|---|
| LLM.int8() | Outliers in FP16, rest in INT8 | W8 (mixed) | 175B model, no degradation, ~2x less memory |
| SmoothQuant | Rescales activations ↔ weights pre-quantization | W8A8 | 1.56x speedup, ~2x memory reduction |
| GPTQ | Solves per-layer reconstruction error | W3–W4, down to 2-bit | 175B model on 1 GPU, ~3.25x speedup |

None of these three is "quantization" as a single fixed algorithm — each one found a specific structural weak point (which dimensions are outliers, which operand is actually hard to quantize, what a layer's own reconstruction error looks like) and built the method to exploit it. That's also roughly the order of difficulty: going from 8 bits down toward 2 needs progressively more of this kind of targeted machinery, because naive rounding simply stops working somewhere around there.
