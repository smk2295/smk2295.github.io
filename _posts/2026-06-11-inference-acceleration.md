---
layout: post
title: "Inference speed isn't just a model-size problem"
date: 2026-06-11 10:00:00+0900
description: PagedAttention, speculative decoding, and early exit all make inference faster without removing a single parameter. Three papers that show "smaller" and "faster" are different goals.
tags: model-compression inference-acceleration efficiency
categories: research-notes
series: Efficient Inference
related_posts: false
toc:
  sidebar: right
---

<div class="papers">
  <div class="paper">
    <span class="paper-venue">SOSP<br>2023</span>
    <p class="paper-title"><a href="https://arxiv.org/abs/2309.06180">Efficient Memory Management for Large Language Model Serving with PagedAttention</a></p>
    <p class="paper-authors">W. Kwon, Z. Li, S. Zhuang, Y. Sheng, L. Zheng, et al.</p>
  </div>
  <div class="paper">
    <span class="paper-venue">ICML<br>2023</span>
    <p class="paper-title"><a href="https://arxiv.org/abs/2211.17192">Fast Inference from Transformers via Speculative Decoding</a></p>
    <p class="paper-authors">Y. Leviathan, M. Kalman, Y. Matias</p>
  </div>
  <div class="paper">
    <span class="paper-venue">arXiv<br>2017</span>
    <p class="paper-title"><a href="https://arxiv.org/abs/1709.01686">BranchyNet: Fast Inference via Early Exiting from Deep Neural Networks</a></p>
    <p class="paper-authors">S. Teerapittayanon, B. McDanel, H. T. Kung</p>
  </div>
</div>

<div class="tldr" markdown="1">
<span class="label">TL;DR</span>

- **PagedAttention / vLLM** treats the KV cache like OS virtual memory: fixed-size, non-contiguous blocks. **2–4× throughput** at the same latency, model untouched.
- **Speculative decoding** lets a small draft model guess ahead and the large model verify in one parallel pass. **2–3× speedup** on T5-XXL, with *exactly* the same output distribution.
- **Early exit** (BranchyNet) lets easy inputs leave the network through intermediate classifiers instead of paying for full depth.
</div>

## Why read these after pruning and quantization

The last two posts in this series were about shrinking a model: fewer weights, fewer bits. None of the three papers here prunes or quantizes anything. That is exactly why they belong together.

Serving latency isn't only a function of parameter count. It is also shaped by **how memory for the KV cache is managed** and **how decoding is scheduled** — and shrinking the model does nothing for either.

## PagedAttention: the KV cache is a memory problem

Autoregressive generation caches key/value activations for every previous token, so they don't get recomputed at each step. With long sequences and many concurrent requests, that cache can end up **larger than the model weights themselves**. Worse, its size keeps changing as generation proceeds.

Standard serving systems allocate this cache contiguously per request. That fragments memory and wastes capacity — a problem operating systems solved for process memory decades ago.

<div class="flow">
  <div class="flow-node"><strong>Contiguous</strong>one reserved chunk per request</div>
  <div class="flow-arrow">→</div>
  <div class="flow-node hot"><strong>Paged</strong>fixed-size, non-contiguous blocks</div>
  <div class="flow-arrow">→</div>
  <div class="flow-node"><strong>Shared</strong>blocks reused across requests in vLLM</div>
</div>
<p class="flow-caption">PagedAttention borrows OS-style paging for the KV cache.</p>

PagedAttention simply imports that solution. KV-cache memory is allocated in fixed-size blocks, the way an OS pages virtual memory, and the vLLM system built on top shares blocks across requests wherever possible.

<div class="stats">
  <div class="stat"><p class="stat-value">2–4×</p><p class="stat-label">throughput at the same latency</p></div>
  <div class="stat"><p class="stat-value">Untouched</p><p class="stat-label">the model itself — no weight is changed</p></div>
</div>

<div class="callout" markdown="1">
**The bottleneck wasn't the model.** It was the bookkeeping around it. Fix the memory layout, and the same weights serve far more requests.
</div>

## Speculative decoding: sequential doesn't have to stay sequential

Decoding one token at a time is inherently sequential: each forward pass depends on the token the last one produced. But Leviathan et al. noticed that many tokens in a typical generation are, in hindsight, easy. A small, cheap draft model would have guessed the same token the large model eventually picks.

So split the work into drafting and verifying:

1. **Draft.** A small model proposes several tokens ahead.
2. **Verify.** The large model checks the whole proposed run in *one parallel pass*.
3. **Keep the matching prefix.** Accepted tokens are kept; decoding resumes from the first mismatch.
4. **Correct.** A rejection-sampling step makes the result exact rather than approximate.
{: .steps}

<div class="callout note" markdown="1">
**This is not an approximation.** Thanks to the rejection-sampling correction, the output distribution is identical to standard decoding. You get the same model, just faster.
</div>

<div class="stats">
  <div class="stat"><p class="stat-value">2–3×</p><p class="stat-label">speedup on T5-XXL</p></div>
  <div class="stat"><p class="stat-value">Exact</p><p class="stat-label">same output distribution as standard decoding</p></div>
  <div class="stat"><p class="stat-value">None</p><p class="stat-label">retraining or architecture change needed</p></div>
</div>

## Early exit: not every input deserves the same depth

BranchyNet is the odd one out. It is an architectural idea rather than a scheduling one.

Attach classifiers at intermediate layers. When an input's prediction at an early branch is confident enough, it exits right there and skips the remaining layers. Only inputs that actually need the full network get it.

<div class="flow">
  <div class="flow-node"><strong>Input</strong></div>
  <div class="flow-arrow">→</div>
  <div class="flow-node hot"><strong>Early branch</strong>confident? exit here</div>
  <div class="flow-arrow">→</div>
  <div class="flow-node"><strong>Deeper layers</strong>only for the hard cases</div>
  <div class="flow-arrow">→</div>
  <div class="flow-node"><strong>Final exit</strong></div>
</div>
<p class="flow-caption">Easy inputs leave early; hard ones pay for the full depth.</p>

<div class="callout question" markdown="1">
**The underlying bet:** difficulty varies across inputs. Spending the same fixed depth on all of them wastes computation on the easy majority.
</div>

## Putting it next to pruning and quantization

Line the three up and each one pulls a different lever — none of which is model size.

<div class="compare" markdown="1">
<div class="compare-card" markdown="1">
<span class="label">PagedAttention</span>
#### Memory layout

- Changes how memory is laid out around a **fixed** model
- Targets KV-cache fragmentation
- Wins on serving throughput
</div>
<div class="compare-card" markdown="1">
<span class="label">Speculative decoding</span>
#### Forward passes

- Changes **how many** large-model passes are needed
- Draft cheaply, verify in parallel
- Exact, not approximate
</div>
<div class="compare-card" markdown="1">
<span class="label">Early exit</span>
#### Depth per input

- Changes **how much** of the network an input traverses
- Confident inputs exit early
- Spends compute where it's needed
</div>
</div>

Stack these next to the pruning and quantization posts and the picture gets clearer: "make it faster" isn't one problem with one lever. A perfectly pruned, perfectly quantized model can still crawl if it's served on a bad memory layout, decoded one token at a time when it didn't have to be, and run through every layer regardless of how easy the input was.

<div class="pullquote" markdown="1">
Model size and inference speed are correlated. They are not the same thing.
</div>

<div class="takeaways" markdown="1">
<span class="label">Key takeaways</span>

- Serving latency depends on **memory management** and **decoding schedule**, not just parameter count.
- **PagedAttention / vLLM** pages the KV cache like OS memory: **2–4× throughput** at the same latency.
- **Speculative decoding** drafts and verifies in parallel: **2–3× speedup** on T5-XXL with an identical output distribution.
- **Early exit** lets easy inputs skip the remaining layers once an intermediate classifier is confident.
- Compression and these techniques pull **different levers** — a fast system needs more than a small model.
</div>
