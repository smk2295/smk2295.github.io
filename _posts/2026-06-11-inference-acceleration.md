---
layout: post
title: "Inference speed isn't just a model-size problem"
date: 2026-06-11 10:00:00+0900
description: PagedAttention/vLLM, speculative decoding, and early-exit networks all speed up inference without shrinking a single parameter — three papers, one point.
tags: model-compression inference-acceleration efficiency
categories: research-notes
related_posts: false
---

**Kwon, W., Li, Z., Zhuang, S., Sheng, Y., Zheng, L., Yu, C. H., Gonzalez, J. E., Zhang, H., & Stoica, I. (2023). [Efficient Memory Management for Large Language Model Serving with PagedAttention](https://arxiv.org/abs/2309.06180). SOSP 2023.**
**Leviathan, Y., Kalman, M., & Matias, Y. (2023). [Fast Inference from Transformers via Speculative Decoding](https://arxiv.org/abs/2211.17192). ICML 2023.**
**Teerapittayanon, S., McDanel, B., & Kung, H. T. (2017). [BranchyNet: Fast Inference via Early Exiting from Deep Neural Networks](https://arxiv.org/abs/1709.01686).**

None of these three papers prunes or quantizes anything, and that's the point of pairing them here: a transformer's serving latency is also shaped by how autoregressive decoding is scheduled and how memory for the KV cache is managed — axes that shrinking the model doesn't automatically address.

## PagedAttention / vLLM: the KV Cache Is a Memory-Management Problem

During autoregressive generation, a transformer caches key/value activations for every previous token so they aren't recomputed at each step. For long sequences and many concurrent requests, this cache can be *larger than the model weights themselves*, and its size changes dynamically as generation proceeds.

**Diagnosis:** standard serving systems allocate this cache contiguously per request, causing memory fragmentation and wasted capacity — the same problem operating systems solved for process memory decades ago.

**Fix:** PagedAttention allocates KV-cache memory in fixed-size, non-contiguous blocks, the way an OS pages virtual memory. The vLLM serving system built on top achieves near-zero cache waste and shares cache blocks across requests where possible.

**Result:** 2–4x throughput improvement over prior state-of-the-art serving systems at the same latency, with larger gains for longer sequences and larger models — entirely a scheduling and memory-layout change, with the model itself untouched.

## Speculative Decoding: Make the Sequential Bottleneck Parallel

Autoregressive decoding is inherently sequential — one forward pass produces one token, and the next token's forward pass depends on it.

**Observation:** many tokens in a generated sequence are, in retrospect, "easy" — predictable enough that a much smaller, cheaper draft model would have guessed the same token as the large target model.

**Method:** the small draft model proposes several tokens ahead; the large model verifies the entire proposed continuation in a single parallel forward pass, accepting whichever prefix matches what it would have produced on its own. A rejection-sampling correction guarantees the output distribution is *exactly* the same as standard decoding — not an approximation.

**Result:** 2–3x speedup on T5-XXL, with no retraining or architecture change to either model, and no change to the output distribution.

## Early Exit / BranchyNet: Not Every Input Needs the Full Network

Side-branch classifiers are attached at intermediate layers. An input exits through an early branch once its prediction there is confident enough, skipping the remaining layers entirely; only inputs that fail the confidence check proceed deeper.

Unlike the two methods above, this is an **architectural** premise, not a scheduling or parallelism one: it assumes inputs vary in difficulty, and spends compute proportionally rather than running every input through the same fixed depth.

## Why Grouping These Three Together Matters

| Method | What it changes | Touches model weights? |
|---|---|---|
| PagedAttention | How memory is laid out around a fixed model | No |
| Speculative decoding | How many forward passes are needed | No |
| Early exit | How much of the network an input traverses | No |

All three are legitimate "make it faster" techniques that never touch a single weight value. A serving system that only invests in a smaller model — via pruning or quantization — while ignoring scheduling, memory layout, and input-dependent compute is leaving comparable or larger gains unaddressed.

## Takeaways

1. Model size and inference speed are correlated, not identical — several of the largest available speedups live entirely outside the model itself.
2. KV-cache management, decoding parallelism, and input-dependent depth are three genuinely separate bottlenecks, not three names for the same one.
3. A well-compressed model served on a bad memory/scheduling stack can still be slow — compression and serving-system design are complementary, not substitutes.
