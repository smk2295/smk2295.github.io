---
layout: post
title: "Inference speed isn't just a model-size problem"
date: 2026-06-11 10:00:00+0900
description: PagedAttention/vLLM, speculative decoding, and early-exit networks all speed up inference without shrinking a single parameter — three papers, one point.
tags: model-compression inference-acceleration efficiency
categories: research-notes
related_posts: false
---

> Kwon, W., Li, Z., Zhuang, S., Sheng, Y., Zheng, L., Yu, C. H., Gonzalez, J. E., Zhang, H., & Stoica, I. (2023). [Efficient Memory Management for Large Language Model Serving with PagedAttention](https://arxiv.org/abs/2309.06180). SOSP 2023.
>
> Leviathan, Y., Kalman, M., & Matias, Y. (2023). [Fast Inference from Transformers via Speculative Decoding](https://arxiv.org/abs/2211.17192). ICML 2023.
>
> Teerapittayanon, S., McDanel, B., & Kung, H. T. (2017). [BranchyNet: Fast Inference via Early Exiting from Deep Neural Networks](https://arxiv.org/abs/1709.01686).

None of these three papers prunes or quantizes a single weight, which is exactly why they're worth reading together after two posts about doing precisely that. A model's serving latency isn't only a function of how many parameters it has — it's also shaped by how decoding gets scheduled and how memory for the KV cache gets managed, and shrinking the model does nothing for either.

## The KV cache is a memory-management problem, not a model problem

Autoregressive generation caches key/value activations for every previous token so they don't get recomputed at each step. For long sequences and many concurrent requests, that cache can end up larger than the model weights themselves, and its size keeps changing as generation proceeds. Standard serving systems allocate it contiguously per request — which fragments memory and wastes capacity in a way operating systems solved for process memory decades ago. PagedAttention just imports that solution: allocate KV-cache memory in fixed-size, non-contiguous blocks the way an OS pages virtual memory, and let the vLLM system built on top share blocks across requests wherever possible. Reported gain: 2–4x throughput at the same latency, with the model itself completely untouched.

## The sequential bottleneck doesn't have to stay sequential

Decoding one token at a time is inherently sequential — the next forward pass depends on the last token produced. Leviathan et al.'s observation is that a lot of tokens in a typical generation are, in hindsight, easy: predictable enough that a small, cheap draft model would have guessed the same token the large model eventually does. So let the draft model propose several tokens ahead, and have the large model verify the whole proposed run in one parallel pass, keeping whichever prefix matches. A rejection-sampling correction makes this exact, not approximate — the output distribution is identical to standard decoding, just faster. Reported: 2–3x speedup on T5-XXL, no retraining, no architecture change.

## Not every input deserves the same depth

BranchyNet is the odd one out — an architectural idea rather than a scheduling one. Attach classifiers at intermediate layers, and let an input exit through an early branch the moment its prediction there is confident enough, skipping whatever layers remain. Only the inputs that actually need the full network get it. It's a bet that difficulty varies across inputs, and that spending the same fixed depth on all of them is wasted computation on the easy majority.

| Method | What it changes |
|---|---|
| PagedAttention | How memory is laid out around a fixed model |
| Speculative decoding | How many forward passes are needed |
| Early exit | How much of the network an input actually traverses |

Stack these next to the pruning and quantization posts and the picture gets clearer: "make it faster" isn't one problem with one lever. A perfectly pruned, perfectly quantized model can still crawl if it's served on a bad memory layout, decoded one token at a time when it didn't have to be, and run through every layer regardless of how easy the input was. Model size and inference speed are correlated. They are not the same thing.
