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

## The Question

When a language model sees a few-shot prompt like "hot → cold, big → small, fast → ___", it produces the right answer with no gradient update at all. Somewhere in the forward pass, the model has to represent *which task the demonstrations are specifying*. Where does that representation live?

## Method: Averaging Activations Into a Single Vector

The paper studies several autoregressive transformers (GPT-J, GPT-NeoX, Llama-2, and others) across a broad set of in-context tasks — lexical mappings (antonym, synonym, translation) as well as more abstract relations.

The construction has two steps:

1. **Locate the relevant heads.** A causal-mediation analysis identifies the small set of attention heads whose activations, when altered, change the model's task-following behavior the most.
2. **Average across examples.** For a fixed task, the activations of those heads are averaged across many different few-shot prompts of that task, at the last token position. Averaging cancels out example-specific noise and leaves one vector per task — the **function vector (FV)**.

## The Causal Test

The important experiment isn't just finding a vector that correlates with a task — it's showing the vector *causes* the behavior:

- The FV is added to the residual stream during a **zero-shot** forward pass — a prompt with no demonstrations at all.
- The model performs the task anyway, recovering a large fraction of true few-shot accuracy, despite never seeing a single input–output example in that forward pass.

## Robustness Checks

| Check | Result reported |
|---|---|
| Different prompt template | Same FV still induces the task when patched into a differently-worded prompt |
| Head count | Only a small, consistent subset of heads (tens, out of thousands) carries the effect |
| Ablation | Removing the identified heads kills ICL for that task; removing random heads of the same count does not |
| Vector arithmetic | Combining FVs for related tasks produces semantically related (though not always fully interpretable) behavior |

## Takeaways

1. "Learning the task from examples" reduces to a single, low-dimensional, addable object — not a diffuse, whole-network state.
2. That object lives in a sparse, identifiable set of attention heads, not spread evenly across the model.
3. Because the FV is a concrete, extractable thing, it becomes a natural object to reason about uncertainty *over* — not just a lever for inducing behavior. That reframing is the starting point for treating ICL uncertainty as uncertainty about a latent task representation, rather than only about the model's final output.
