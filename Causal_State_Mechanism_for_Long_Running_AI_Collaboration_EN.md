# Causal State Mechanism for Long-Running AI Collaboration

## Overview

Today, major directions such as Google, Anthropic, and LangChain have already shifted their focus from pure prompt design toward context engineering, long-session state management, and cross-session memory. The core goal is no longer to keep appending conversation history into the context window, but to reduce cost, latency, and instability in long-horizon tasks through structured storage, relevance filtering, compression, and on-demand reinjection. Google even describes context as a “compiled view” over a larger state system; Anthropic treats context engineering as a key problem in building long-running agents; and LangGraph has already turned long-term memory, thread state, and persistent agent state into first-class capabilities.

Current mainstream practice focuses more on questions such as “what information should be retained,” “when should it be retrieved,” and “how can context cost be controlled.” This document is more concerned with a different question: once information re-enters later reasoning, how should it be organized by causal structure, routed by priority, and expanded according to the user’s attention habits and tolerance for evidential uncertainty?

In other words, the topic here is not merely a memory system, but an external reasoning protocol for long-running collaboration: instead of simply caching conclusions, it maintains a structured state that carries causality, evidence, scope, and attention guidance.

## Original Method and Its Limitations

My original method was essentially a “conclusion caching” mechanism: after one round of reasoning or experimentation, I did not preserve the full reasoning process, but only the final conclusions, and then reused those conclusions directly in later turns to avoid re-expanding the same reasoning chain. In some cases, I also preserved a small amount of other core semantics for cross-session synchronization.

The goal was straightforward: use conclusions to replace repeated reasoning, thereby reducing extra reasoning cost in long conversations.

This method usually works in lightweight tasks and short reasoning chains. In such settings, the number of conclusions is limited, dependencies among conclusions are weak, the reasoning chain is short, and the reasons, conditions, and evidence behind those conclusions still remain in the recent context. Under these conditions, a “conclusion” can approximately stand in for the process that produced it, and later work depends more on “what judgment has already been made” than on revisiting “why that judgment was valid.”

However, this approximation gradually breaks down in long-running tasks, especially in experiment-heavy workflows. A conclusion is not an isolated information node; it is the compressed result of a set of conditions, evidence, and causal relationships. Preserving only the conclusion effectively discards the prerequisites, scope, evidence strength, and the dependency or override relations between conclusions. In short conversations, this missing information can still be implicitly reconstructed from local context; but across stages, sessions, and experiment rounds, that implicit recovery quickly fails.

Even within a single long-running session, context windows remain finite. Every round of context compression introduces some degree of semantic loss. As compression accumulates, originally precise semantics become less precise, especially the hidden conditions under which reasoning was valid. Later reasoning may depend not only on the conclusions previously reached, but also on the reasons why those conclusions held—particularly in experimental and testing tasks, where later results may overturn earlier conclusions. Without preserving the “why” behind an earlier conclusion, it becomes difficult to derive the next correct one. For this reason, preserving not just conclusions but a causal structure becomes almost necessary in long-running tasks.

In addition, there are situations where, for response speed or to avoid excessive reasoning burden, one must switch to a new session and synchronize state. At that point, preserving only causal structure is still not enough. Most of what remains in a causal state file is already “important.” If the entire file is simply passed into a new session, the context window may become smaller, but the attention burden does not disappear. When the model treats the whole file as important, the actual reasoning burden does not decrease by much. Therefore, preserving causality alone is insufficient; an additional external attention mechanism is also needed.

For this reason, “storing only conclusions” is better understood as a local optimization for lightweight conversations, not as a stable mechanism for long-running collaboration. It can reduce some repeated reasoning, but it is not enough to support persistent alignment across stages. What truly needs to be maintained is not only the conclusions themselves, but also the causal structure behind them, together with the priority and expansion order of that structure under the current task.

## A New Proposal

> Note: this proposal is a personal suggestion. It is not yet a mature or universal solution.

### Causal Structure

Here, “causality” is not an abstract philosophical concept. It is the minimum supporting structure required for a conclusion to hold. That is: why a conclusion holds, under what conditions it holds, what evidence supports it, what assumptions it depends on, and what later judgments it may affect.

At minimum, this causal structure includes six categories of information:

- **Conclusion itself**: what judgment was reached
- **Reason**: why that judgment was reached
- **Evidence**: code (represented by file ranges instead of full source when necessary), logs, experiment results, statistical data, observations
- **Scope**: within what range this conclusion is valid
- **Premises / assumptions**: what conditions it depends on
- **Relations**: what it depends on, what it overrides, what it invalidates

“Causality” here is not a verbatim record of the full reasoning process. It is the minimum logical structure required for a conclusion to be reusable in future reasoning. A minimal example looks like this:

```text
Fact:
  id
  statement
  why
  evidence
  scope
  assumptions
  depends_on
  supersedes / invalidates
  confidence
```

- `statement`: the conclusion itself
- `why`: the minimal cause, without expanding the full reasoning chain
- `evidence`: what supports the conclusion
- `scope`: the applicability range
- `assumptions`: required premises
- `depends_on`: which prior conclusions it relies on
- `supersedes / invalidates`: which older conclusions it overrides or invalidates
- `confidence`: current confidence level

The goal of causal structure is not to reproduce a complete thought process, but to ensure that when conditions change, questions shift, or conclusions come into conflict, the system can still judge whether a conclusion remains usable.

### External Attention Mechanism

The “external attention mechanism” discussed here is not an attempt to replace the model’s internal attention. Instead, before reasoning begins, it performs a structured routing pass over the external state: deciding which information should be prioritized under the current question, which should remain as background, and which should temporarily remain collapsed.

This mechanism answers questions such as:

- Which conclusions are most relevant to the current question?
- Which causal nodes must be expanded?
- Which nodes can remain as summaries only?
- Which older conclusions are important but currently irrelevant?
- Which content is already outdated or overridden?

At its core, it does three things: **selection** (what information should enter reasoning), **ordering** (what enters first), and **expansion control** (what should remain as conclusion only, and what must be expanded into reason and evidence).

When this mechanism operates in a conversation, the model should not assign a permanent internal “weight” to each causal structure. Instead, an external system should dynamically compute a **priority score** or **routing score** for the current round, based on the current query, the existing causal structures, and a policy configuration. This score is not a direct exposure of the model’s internal attention; it is an external routing result used to decide which information enters the current reasoning first, which remains summarized, which must be expanded into reason and evidence, and which can remain collapsed. The model may assist with certain semantic judgments—such as relevance, expansion need, or potential conflict—but the final score should be computed by the external mechanism. When necessary, the user or maintainer can further calibrate the policy.

Part of this mechanism depends on personal style.

**Aspects that are relatively style-independent (more universal):**

- Whether something is directly relevant to the current question
- Whether it is a high-confidence hard fact
- Whether it has already been overturned by newer evidence
- Whether it belongs to the global constraints of the current task
- Whether it is a critical dependency for later reasoning

**Aspects that clearly depend on personal style:**

- Whether one prefers conclusion-first or derivation-first reading
- Tolerance for empirical assumptions
- Required level of evidential completeness
- Whether low-confidence conclusions may temporarily enter reasoning
- A more conservative vs. exploratory preference
- A preference for local optimality vs. global completeness

The skeleton of an external attention mechanism can be universal, but its strategy layer is affected by the user’s attention habits, evidence threshold, and reasoning preferences. Therefore, it is more suitable to design it as **a universal protocol plus a personalized strategy**, rather than as a single fixed rule. A minimal example:

```text
Policy Profile:
  read_order: conclusion_first / causal_first
  evidence_threshold: low / medium / high
  allow_empirical_assumption: true / false
  expansion_rule:
    - high_relevance + low_confidence -> expand why + evidence
    - low_relevance + high_confidence -> keep statement only
```

In implementation, the external attention mechanism does not necessarily require a precise continuous scoring model. At the current stage, a discrete grading mechanism is often more practical. The goal is not to produce a seemingly precise number, but to provide interpretable routing and expansion decisions for the current question. Compared with continuous scoring, grade systems such as A/B/C/D/E/F are easier to define, and also easier for AI to draft initially and for humans to calibrate afterward.

To avoid forcing one grade to represent both “whether it should enter reasoning” and “how deeply it should be expanded,” grading should be split into at least two dimensions: a **routing grade**, which determines whether a causal structure should enter the current reasoning process with priority; and an **expansion grade**, which determines whether it should remain as a conclusion only or be expanded into reasons and evidence. In this way, one can build a workable, interpretable, and adjustable external attention mechanism without first implementing a complex quantitative scoring system.

If greater precision and stability are later required, a finer scoring mechanism can be introduced. But at the current stage, the combination of “AI assigns grades according to rules + human correction” is often already sufficient for high-intensity long-running collaboration.

## Closing

This document is not intended as a finalized standard answer. It is a stage-by-stage summary of how I currently understand long-running AI collaboration mechanisms. The purpose of writing it is not only to share the idea, but also to see whether others have tried similar approaches or encountered similar issues in related directions. If there are existing practices, failed attempts, or different judgments, I would be glad to discuss them.
