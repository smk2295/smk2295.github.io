---
layout: post
title: Paper notes — Function Vectors in Large Language Models
date: 2025-07-22 10:00:00+0900
description: Todd, Li, Sharma, Mueller, Wallace, and Bau (ICLR 2024) show that in-context learning induces a single, portable vector that causally encodes the task.
tags: uncertainty in-context-learning interpretability
categories: paper-notes
related_posts: false
---

**Todd, E., Li, M. L., Sharma, A. S., Mueller, A., Wallace, B. C., & Bau, D. (2024). [Function Vectors in Large Language Models](https://arxiv.org/abs/2310.15213). ICLR 2024.**

**Setup.** The paper studies autoregressive transformer LMs (GPT-J, GPT-NeoX, Llama-2, and others) on a broad set of in-context learning tasks — lexical mappings (antonym, synonym, English–French translation) as well as more abstract relations — presented as few-shot prompts.

**Method.** For a given task, the authors run the model on many few-shot prompts and average the activations of a specific set of attention heads at the last token position, across many different ICL examples of the same task. Which heads are included is decided by a causal-mediation analysis: heads whose activations, when altered, change the model's output the most for that task are kept; the rest are discarded. Averaging over examples cancels out example-specific noise and leaves a single vector — the **function vector (FV)** — that the authors claim represents "the operation the demonstrations are specifying," not any particular demonstration.

**Core experiment.** The causal test is the important part: the FV is added to the residual stream at the corresponding layer during a **zero-shot** forward pass — a prompt with no demonstrations at all — and the model performs the task anyway. Across the tasks tested, this zero-shot-plus-FV setup recovers a large fraction of the accuracy of the true few-shot prompt, despite the model never seeing a single input–output example in that forward pass.

**Robustness checks reported in the paper:**
- The same FV, extracted from one prompt template, still induces the task when patched into a differently-formatted prompt.
- FVs are computed from a small, consistent subset of heads (on the order of tens out of thousands across the model) — most heads contribute little to the effect, and ablating the identified heads specifically removes the model's ICL ability on the task while ablating random heads of the same count does not.
- Vector arithmetic on FVs (e.g., combining FVs for related tasks) produces semantically related — though not always fully interpretable — behavior, suggesting some compositional structure to the space FVs live in.

**Takeaway.** The paper's central claim is that whatever "learning the task from examples" means mechanistically, it collapses to a single, low-dimensional, addable object located in a sparse set of attention heads — not a diffuse, whole-network state. That object is what later work (including our own on uncertainty in ICL) treats as the thing to be uncertain *about*, rather than only a thing to be *induced* with.
