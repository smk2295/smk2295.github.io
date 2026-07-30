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

**The shared problem.** All three papers are solving variants of the same issue: naively rounding a large transformer's 16-bit weights and activations to 8 or 4 bits produces unacceptable accuracy loss, and the reason is not random noise — it's **outliers**. Large transformers develop specific feature dimensions with activation magnitudes many times larger than the rest of the tensor. A single quantization scale sized to cover those outliers leaves almost no resolution for the majority of "normal" values, which is where most of the model's useful signal actually lives.

**LLM.int8() (Dettmers et al., 2022): keep the outliers in higher precision.** Rather than force every value through one 8-bit scale, this method uses vector-wise quantization for the bulk of values and identifies the small number of outlier feature dimensions, computing those specific dimensions in 16-bit floating point via a separate mixed-precision path, while quantizing "more than 99.9%" of values to int8. The paper reports that a 175B-parameter model (OPT-175B) can be converted to Int8 and run with no performance degradation relative to the 16-bit original, roughly halving the memory footprint needed for inference and making models of that scale runnable on far more modest hardware.

**SmoothQuant (Xiao et al., 2023): move the difficulty instead of hiding it.** The authors' framing is that "weights are easy to quantize, activations are not." Rather than special-casing outlier dimensions at runtime as LLM.int8() does, SmoothQuant applies a mathematically equivalent rescaling *before* quantization: it divides problematic activation channels by a per-channel smoothing factor and multiplies the corresponding weight channels by the same factor, which leaves the matrix product unchanged but shifts quantization difficulty from the (hard-to-quantize) activations onto the (easy-to-quantize) weights. This enables full W8A8 (8-bit weights, 8-bit activations) quantization — both operands low-precision, not just weights — and the paper reports up to 1.56x inference speedup and roughly 2x memory reduction with negligible accuracy loss, including enabling a 530B-parameter model to be served on a single compute node.

**GPTQ (Frantar et al., 2023): solve for the weights directly, and go lower-precision.** Rather than a fixed rounding rule, GPTQ formulates quantization as a layer-by-layer optimization problem: given a small calibration dataset, solve for the quantized weight values that minimize the layer's output reconstruction error, using approximate second-order (Hessian) information to decide the order and adjustment needed as each weight is quantized. This is a one-shot, post-training method — no retraining loop — yet the paper reports compressing a 175B-parameter model to 3–4 bits per weight in about four GPU-hours while preserving accuracy, fitting such a model onto a single GPU for inference, with roughly 3.25x measured latency speedup on A100 GPUs, and reports the method remains reasonably accurate even pushed to 2-bit or ternary weights.

**The common thread.** None of these three papers treats "quantization" as a single fixed algorithm — each identifies a specific structural property of transformer activations or weights (outlier channels, easy-vs-hard-to-quantize operands, per-layer reconstruction error) and designs the method around exploiting or compensating for it, rather than quantizing blindly.
