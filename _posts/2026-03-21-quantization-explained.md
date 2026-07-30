---
layout: post
title: "Quantization: fewer bits per weight, without losing the model"
date: 2026-03-21 10:00:00+0900
description: LLM.int8(), SmoothQuant, and GPTQ — three concrete answers to the same problem, outlier activations that make naive low-bit quantization of LLMs fail.
tags: model-compression quantization efficiency
categories: research-notes
related_posts: false
---

**Dettmers, T., Lewis, M., Belkada, Y., & Zettlemoyer, L. (2022). [LLM.int8(): 8-bit Matrix Multiplication for Transformers at Scale](https://arxiv.org/abs/2208.07339). NeurIPS 2022.**
**Xiao, G., Lin, J., Seznec, M., Wu, H., Demouth, J., & Han, S. (2023). [SmoothQuant: Accurate and Efficient Post-Training Quantization for Large Language Models](https://arxiv.org/abs/2211.10438). ICML 2023.**
**Frantar, E., Ashkboos, S., Hoefler, T., & Alistarh, D. (2023). [GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers](https://arxiv.org/abs/2210.17323). ICLR 2023.**

## The Shared Problem: Outliers

Naively rounding a large transformer's 16-bit weights and activations to 8 or 4 bits produces unacceptable accuracy loss, and the reason isn't random noise — it's **outliers**. Large transformers develop specific feature dimensions with activation magnitudes many times larger than the rest of the tensor. A quantization scale sized to cover those outliers leaves almost no resolution for the "normal" values, which is where most of the model's useful signal lives.

A basic uniform quantizer maps a real value $x$ to an integer level using a scale $s$:

$$
q = \text{round}(x / s), \qquad \hat x = s \cdot q
$$

If $s$ has to be large enough to cover a rare outlier, every ordinary value gets rounded far more coarsely than it needs to be. Each of the three papers below is a different answer to this exact tension.

## LLM.int8(): Keep the Outliers in Higher Precision

Rather than force every value through one 8-bit scale, LLM.int8() uses vector-wise quantization for the bulk of values and identifies the small number of outlier feature dimensions, computing *those specific dimensions* in 16-bit floating point via a separate mixed-precision path — while quantizing more than 99.9% of values to int8.

**Result:** a 175B-parameter model (OPT-175B) converts to Int8 and runs with no performance degradation relative to the 16-bit original, roughly halving the memory footprint and making models of that scale runnable on far more modest hardware.

## SmoothQuant: Move the Difficulty Instead of Hiding It

The framing: "weights are easy to quantize, activations are not." Instead of special-casing outlier dimensions at runtime, SmoothQuant applies a mathematically equivalent rescaling *before* quantization — dividing problematic activation channels by a per-channel smoothing factor and multiplying the corresponding weight channels by the same factor. The matrix product is unchanged, but quantization difficulty shifts from the hard-to-quantize activations onto the easy-to-quantize weights.

This enables full **W8A8** (8-bit weights, 8-bit activations) — both operands low-precision, not just weights.

**Result:** up to 1.56x inference speedup and roughly 2x memory reduction with negligible accuracy loss, including serving a 530B-parameter model on a single compute node.

## GPTQ: Solve for the Weights Directly, and Go Lower

Rather than a fixed rounding rule, GPTQ formulates quantization as a layer-by-layer optimization problem: given a small calibration dataset, solve for the quantized weight values that minimize the layer's output reconstruction error, using approximate second-order (Hessian) information to decide the order and adjustment as each weight is quantized. It's one-shot and post-training — no retraining loop.

**Result:** compresses a 175B-parameter model to 3–4 bits per weight in about four GPU-hours while preserving accuracy, fits it onto a single GPU for inference, and reports roughly 3.25x measured latency speedup on A100 GPUs — remaining reasonably accurate even pushed to 2-bit or ternary weights.

## Comparing the Three

| Method | What it changes | Precision achieved | Headline result |
|---|---|---|---|
| LLM.int8() | Keeps outlier dims in FP16, rest in INT8 | W8 (mixed) | 175B model, no degradation, ~2x less memory |
| SmoothQuant | Rescales activations↔weights before quantizing | W8A8 | 1.56x speedup, ~2x memory reduction |
| GPTQ | Solves per-layer reconstruction error directly | W3–W4 (down to 2-bit) | 175B model on 1 GPU, ~3.25x speedup |

## Takeaways

1. None of these three treats "quantization" as one fixed algorithm — each identifies a specific structural property (outlier channels, easy-vs-hard-to-quantize operands, per-layer reconstruction error) and designs around it.
2. Handling outliers well is the difference between quantization that works on LLMs and quantization that collapses accuracy.
3. Going from 8-bit to 4-bit or lower needs progressively more careful machinery (GPTQ's per-layer optimization) — naive rounding doesn't scale down gracefully on its own.
