---
layout: post
title: "A task, folded into a single vector"
date: 2025-07-22 10:00:00+0900
description: In-context learning leaves a fingerprint. Todd et al. (ICLR 2024) extract it as a single, portable vector — and show it doesn't just track the task, it causes it.
tags: uncertainty in-context-learning interpretability
categories: paper-notes
series: Uncertainty in In-Context Learning
related_posts: false
toc:
  sidebar: right
---

<div class="papers">
  <div class="paper">
    <span class="paper-venue">ICLR<br>2024</span>
    <p class="paper-title"><a href="https://arxiv.org/abs/2310.15213">Function Vectors in Large Language Models</a></p>
    <p class="paper-authors">E. Todd, M. L. Li, A. S. Sharma, A. Mueller, B. C. Wallace, D. Bau</p>
  </div>
</div>

<div class="tldr" markdown="1">
<span class="label">TL;DR</span>

- During in-context learning, a **small set of attention heads** carries a compact representation of *which task* the demonstrations describe.
- Averaging those heads' outputs over many prompts gives a **function vector** — a single vector that stands in for the task.
- Add it to a **zero-shot** prompt and the model performs the task anyway. The vector doesn't just correlate with the task; it *causes* it.
</div>

## Where does the task live?

Show a language model "hot → cold, big → small, fast → ___" and it fills in the blank correctly, with no training involved. Somewhere inside that single forward pass, the model must be holding onto *which task the examples are asking for*.

Todd et al. went looking for that thing. What they found is smaller, and far more portable, than you might expect.

<div class="callout question" markdown="1">
**If the model has inferred a task from the demonstrations, can we point to where that inference is stored** — and pull it out as an object of its own?
</div>

## Extracting a function vector

The recipe has two steps, and the second one is where the magic happens.

1. **Find the heads that matter.** A causal-mediation analysis narrows down which attention heads actually drive the task — the heads whose activations, when nudged, move the model's output the most.
2. **Average them across prompts.** Take those heads' activations at the final token position, averaged over many few-shot prompts of the same task. Whatever is specific to any single example washes out.
{: .steps}

<div class="flow">
  <div class="flow-node"><strong>Few-shot prompts</strong>many, same task</div>
  <div class="flow-arrow">→</div>
  <div class="flow-node"><strong>Causal heads</strong>found by mediation</div>
  <div class="flow-arrow">→</div>
  <div class="flow-node"><strong>Average</strong>at the final token</div>
  <div class="flow-arrow">⇒</div>
  <div class="flow-node hot"><strong>Function vector</strong>the task itself</div>
</div>
<p class="flow-caption">What survives the averaging looks like a representation of the task, not of any one example.</p>

The analysis spans GPT-J, GPT-NeoX, Llama-2, and others, on tasks ranging from simple lexical mappings — antonym, synonym, translation — to more abstract relations.

## The test that matters

Finding a vector that *correlates* with a task is easy. The interesting claim is that it *causes* the task, and the experiment behind that claim is clean.

<div class="compare" markdown="1">
<div class="compare-card" markdown="1">
<span class="label">Few-shot</span>
#### The usual setup

- Real demonstrations in the prompt
- The model infers the task on its own
- Serves as the reference accuracy
</div>
<div class="compare-card" markdown="1">
<span class="label">Zero-shot + FV</span>
#### The intervention

- No demonstrations at all
- The function vector is added to the hidden state
- The model performs the task anyway, recovering most of the few-shot accuracy
</div>
</div>

A few extra checks make the causal story much harder to dismiss as coincidence:

<div class="stats">
  <div class="stat"><p class="stat-value">Portable</p><p class="stat-label">a vector extracted from one prompt template still works when patched into a differently-worded prompt</p></div>
  <div class="stat"><p class="stat-value">Localized</p><p class="stat-label">ablating the few causal heads kills ICL on the task; ablating the same number of random heads doesn't</p></div>
  <div class="stat"><p class="stat-value">Composable</p><p class="stat-label">vectors for related tasks can be combined and still produce sensible behavior</p></div>
</div>

<div class="callout note" markdown="1">
**Composition is suggestive, not settled.** Combined vectors produce sensible but not always fully interpretable behavior — a hint of compositional structure in whatever space these vectors live in, rather than a clean algebra of tasks.
</div>

<div class="pullquote" markdown="1">
Zero demonstrations, one added vector, and the model does the task. That is what turns a correlate into a cause.
</div>

## Why I care: a handle on the task

What I find most useful here isn't the mechanism on its own — it's what the result hands you afterward. "The task the model thinks it's solving" stops being a vague notion and becomes a **concrete object** you can extract, perturb, and measure.

That is exactly the kind of object my work on uncertainty in in-context learning needs. Asking how *uncertain* a model is about which task it's even doing is very hard when the only handle you have is its final output. A function vector gives you a handle one level deeper.

<div class="takeaways" markdown="1">
<span class="label">Key takeaways</span>

- ICL induces a **compact task representation** carried by a small set of attention heads.
- Averaging those heads over many prompts yields a **function vector** that stands in for the task.
- Adding it to a **zero-shot** prompt recovers most of the few-shot accuracy — evidence of a **causal** role.
- It transfers across prompt templates, is **localized** to specific heads, and shows hints of **composition**.
- For uncertainty research, it turns "the inferred task" into something you can actually **measure**.
</div>
