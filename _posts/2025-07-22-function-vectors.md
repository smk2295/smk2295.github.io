---
layout: post
title: A task, folded into a single vector
date: 2025-07-22 10:00:00+0900
description: Todd, Li, Sharma, Mueller, Wallace, and Bau (ICLR 2024) show that in-context learning induces a single, portable vector that causally encodes the task.
tags: uncertainty in-context-learning interpretability
categories: paper-notes
related_posts: false
---

> Todd, E., Li, M. L., Sharma, A. S., Mueller, A., Wallace, B. C., & Bau, D. (2024). [Function Vectors in Large Language Models](https://arxiv.org/abs/2310.15213). ICLR 2024.

Show a language model "hot → cold, big → small, fast → ___" and it fills in the blank correctly, no training involved. Somewhere in that single forward pass, the model has to be holding onto *which task the examples are asking for*. Todd et al. went looking for that thing, and found it's smaller and more portable than you'd expect.

## Finding the vector

The recipe has two steps. First, a causal-mediation analysis narrows down which attention heads actually matter for a given task — heads whose activations, when nudged, move the model's output the most. Second, those heads' activations get averaged across many few-shot prompts of the same task, at the final token position. Averaging washes out whatever is specific to any one example and leaves behind something that looks like a representation of the task itself: a **function vector**.

This is tested across GPT-J, GPT-NeoX, Llama-2, and others, on tasks ranging from simple lexical mappings (antonym, synonym, translation) to more abstract relations.

## The test that matters

Finding a vector that correlates with a task is easy. The interesting claim is that it *causes* the task, and the experiment for that is clean: take the function vector, add it to a **zero-shot** prompt — no demonstrations, nothing — and see what happens. The model performs the task anyway, recovering most of the accuracy it would have gotten from the real few-shot examples, despite never seeing one.

A few extra checks make the causal story harder to dismiss as coincidence:

- The same vector, extracted from one prompt template, still works when patched into a differently-worded prompt.
- Only a small, consistent handful of heads carry the effect — ablating exactly those heads kills the model's ICL ability on the task; ablating an equal number of random heads doesn't.
- Vectors for related tasks can be combined and still produce sensible, if not always fully interpretable, behavior — some hint of compositional structure in whatever space these vectors live in.

What I find most useful about this result isn't the mechanism by itself, it's what it hands you afterward: "the task the model thinks it's solving" stops being a vague notion and becomes a concrete object you can extract, perturb, and measure. That's exactly the kind of object you need if you want to ask how *uncertain* a model is about what task it's even doing — a question that's much harder to ask when the only handle you have is the model's final output.
