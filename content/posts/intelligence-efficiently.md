---
title: "Intelligence, Efficiently"
date: 2026-09-05
description: "Why faster and cheaper AI comes from co-designing the model, the inference system, and the agentic harness—not merely choosing a smaller model."
summary: "A systems view of AI efficiency across three layers: the model, the inference system, and the agentic harness."
tags: ["AI Infrastructure", "Inference", "AI Agents", "Efficiency"]
math: true
draft: false
---

You can think of an AI system as a restaurant.

**The model is the chef.** A better chef can handle harder dishes: ambiguous requests, multi-step reasoning, code, images, and large bodies of information.

But the chef does not work in a vacuum.

**The inference system is the kitchen:** the ovens, refrigerators, workstations, order queue, and the way ingredients are prepared. Put an exceptional chef in a badly organised kitchen and customers will still wait. Ingredients are fetched twice. Equipment sits in the wrong place. Tasks that could happen together are performed one after another. The meal may still be excellent, but it takes too long and costs too much to produce.

**The agentic harness is the restaurant manager.** It decides which information the chef needs, which tools are available, what work can be delegated, when a result needs checking, and when the dish is ready to leave the kitchen. It should also remember what has already been done so the system does not repeat the same search, tool call, or line of reasoning.

These three layers are often discussed separately. Research labs improve model capability. Infrastructure teams make inference faster. Product teams build agents that use tools and complete longer workflows. In production, however, the efficiency of an AI application belongs to none of these layers alone.

It emerges from all three:

1. **The model** determines the underlying reasoning capability.
2. **The inference system** determines how efficiently that capability is executed.
3. **The agentic harness** determines how much inference is needed to finish the job.

The important move is not simply to “make the model smarter.” It is to optimise **all three layers together**. A faster attention kernel, a cached prompt prefix, one unnecessary tool call removed, and two independent steps executed in parallel may each look like a small improvement. Across millions of requests—or one agent that runs for dozens of steps—they compound into a large difference in latency, throughput, and cost.

> Cheaper and faster AI does not necessarily come from a smaller model. It can come from running the same powerful model inside a much smarter system.

## Efficiency Is a Property of the Path

The first mistake in AI optimisation is choosing the wrong unit of analysis.

A model call has a price and a latency. A useful outcome has a path. That path may include retrieval, several model calls, tool execution, verification, retries, fallbacks, and human review. Optimising one call while making the path longer is not progress.

For a request \(r\), a simplified cost model is:

<p class="concept-equation">\[C(r) = \sum_{i=1}^{n} \left(C_{\text{input},i} + C_{\text{output},i} + C_{\text{compute},i}\right) + C_{\text{tools}} + C_{\text{recovery}}\]</p>

The corresponding latency is not always the sum of every step. Independent work may run concurrently, so end-to-end latency follows the **critical path**: the longest chain of dependent operations.

This distinction explains several apparent paradoxes:

- A more expensive model can lower total cost if it reaches the quality bar without retries.
- A faster model can produce a slower application if the harness makes too many sequential calls.
- A larger context window can make an agent less efficient if the harness fills it with low-signal history.
- More parallelism can reduce latency while increasing total compute.
- Higher throughput can coexist with worse tail latency when the scheduler optimises the fleet rather than the individual request.

The correct target is therefore not tokens per second in isolation. It is something closer to **cost and latency per accepted outcome**, measured under a defined quality and safety contract.

That contract matters. An answer delivered in two seconds is not efficient if it is wrong. A coding agent is not efficient because it edited a file quickly if the change fails the test suite. A customer-support agent is not efficient because it avoided escalation if it invented the customer’s account state.

Efficiency begins only after “good enough” has been defined.

## Layer One: The Model

The model sets the capability frontier. It determines how much useful work can be extracted from a given amount of computation, but “model size” is an incomplete description of that trade-off.

### Capability per unit of active computation

A model may contain many parameters without activating all of them for every token. Sparse mixture-of-experts architectures route an input through a subset of specialised parameters. The [Switch Transformer paper](https://arxiv.org/abs/2101.03961) demonstrated the core idea at scale: parameter count and per-token computation do not have to grow together.

This is important because two models with similar headline size can have very different serving characteristics. Architecture, numerical precision, memory footprint, active parameters, sequence length, and hardware utilisation all influence the actual cost of inference.

There are several levers at this layer:

- **Architecture:** dense versus sparse activation, attention design, and the balance between compute and memory traffic.
- **Training:** better data, objectives, and post-training can increase capability without a proportional increase in serving cost.
- **Precision:** lower-precision weights and activations can reduce memory use and accelerate supported hardware, subject to an acceptable quality loss.
- **Distillation:** a smaller model can learn a narrower capability from a stronger teacher when the workload is stable enough.
- **Reasoning allocation:** some requests benefit from additional test-time computation; others do not.

The optimisation is constrained by a quality floor. Shrinking or compressing a model until it becomes unreliable merely moves cost into retries, verification, and human repair. The cheapest useful model is not the smallest model available. It is the least expensive model that reliably satisfies the contract for that class of request.

This is why routing belongs near the boundary between the model and the harness. Easy, well-specified work can go to a smaller or lower-reasoning path. Ambiguous or high-stakes work can go directly to a stronger one. Routing is valuable only when the cost of classification and misrouting is lower than the work it saves.

## Layer Two: The Inference System

If the model is the recipe, the inference system is how the kitchen executes it.

Autoregressive language-model inference has two operationally different phases:

1. **Prefill** processes the input prompt and constructs the key-value cache used by attention.
2. **Decode** produces new tokens one by one while reading and extending that cache.

Prefill can process many input tokens in parallel and is usually compute-intensive. Decode is sequential across generated tokens and often constrained by memory bandwidth. Treating them as one undifferentiated operation hides where time and resources are actually going.

### Move less data

Modern accelerators are extremely fast at arithmetic, but performance can still be dominated by moving data through the memory hierarchy. [FlashAttention](https://arxiv.org/abs/2205.14135) made this point explicit. Its exact attention algorithm reduces reads and writes between high-bandwidth memory and on-chip SRAM through tiling. The mathematical attention operation remains the same; the implementation performs it with a better awareness of the machine.

This is a useful general lesson: fewer floating-point operations do not automatically mean lower wall-clock latency. An algorithm must also respect memory traffic, kernel-launch overhead, device topology, and the shapes of real workloads.

### Manage the KV cache as a shared resource

The key-value cache grows with each sequence and consumes valuable accelerator memory. Poor allocation wastes capacity through fragmentation and over-reservation, reducing the number of requests that can be served together.

The [vLLM PagedAttention paper](https://arxiv.org/abs/2309.06180) applies an operating-system idea to this problem. Instead of requiring each sequence to occupy one contiguous block, it divides the KV cache into pages that can be allocated and shared more flexibly. In the paper’s evaluated configurations, the resulting system delivered substantially higher throughput at comparable latency than the baselines used by the authors. The durable idea is not a universal multiplier; it is that memory management is part of model-serving performance.

### Batch continuously, not statically

Traditional batching waits for a group of requests, processes them together, and waits for the whole batch to finish. That is awkward for generation because responses have different lengths. Short requests can be trapped behind long ones, while finished sequences leave unused capacity.

Continuous batching lets completed sequences leave and new sequences join between decoding iterations. This can keep the accelerator busier, but it also creates a scheduling policy problem. An interactive request cares about time to first token and smooth token delivery. A batch summarisation job may care only about total throughput. A good scheduler distinguishes those service classes rather than pretending that one queue discipline fits both.

### Reuse work that is actually identical

Agents repeatedly send stable instructions, tool definitions, policy text, and shared documents. If those tokens form the same prompt prefix, the system may be able to reuse previously computed state instead of processing the prefix again.

[OpenAI’s prompt-caching documentation](https://developers.openai.com/api/docs/guides/prompt-caching) recommends placing stable content at the beginning of a request and dynamic content later to improve cache reuse. The exact discounts and retention rules are provider-specific and change over time. The architectural principle is stable: arrange repeated computation so the serving layer can recognise and reuse it.

Caching is not free correctness. A cache needs an identity boundary, an invalidation policy, and observability. Reusing the wrong user’s context or stale policy state is worse than a cache miss.

### Predict, then verify

Autoregressive decoding appears irreducibly serial: produce one token, feed it back, then produce the next. Speculative decoding changes the execution strategy. A cheaper draft model proposes several tokens; the target model verifies them together and accepts the valid prefix.

The original [speculative-decoding paper](https://arxiv.org/abs/2211.17192) showed that this can accelerate sampling without changing the target model’s output distribution. It is a striking example of systems intelligence: the strong model is not replaced, and the result is not approximated. The system reorganises how the same result is computed.

The gain depends on draft speed, acceptance rate, target-model behaviour, batch size, and hardware. Speculation is not automatically faster in every environment. It must be measured on the workload it will serve.

### Measure the latency users experience

“Latency” should be decomposed into at least:

| Metric | What it reveals |
| --- | --- |
| Queue time | Capacity pressure and scheduling behaviour |
| Time to first token | Prefill, queueing, and perceived responsiveness |
| Inter-token latency | Decode speed and streaming smoothness |
| End-to-end latency | The full user-visible wait |
| Throughput | Fleet efficiency under load |
| P95/P99 latency | Whether the long tail violates the product contract |

An optimisation that improves average tokens per second but doubles P99 time to first token may be a regression for an interactive product. Infrastructure efficiency is always efficiency for a particular workload and service-level objective.

## Layer Three: The Agentic Harness

An efficient inference engine can still be buried under an inefficient agent loop.

The harness decides what enters the model context, which model is called, which tools are exposed, what can run in parallel, what state persists, how failures are recovered, and when the task stops. In long-running agents, these decisions can dominate total cost.

### Treat context as a working set, not a transcript archive

The easiest implementation of a multi-turn agent is to append every message and tool result forever. It is also a reliable way to make each successive call more expensive.

Long context creates three kinds of waste:

1. **Computational waste:** more tokens must be processed or retrieved from cache.
2. **Attention waste:** relevant evidence competes with stale plans, duplicated output, and verbose tool logs.
3. **Control waste:** obsolete instructions or failed approaches remain available to influence the next decision.

[Anthropic’s context-engineering guidance](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) frames context as a finite resource and recommends selecting the smallest high-signal set of tokens that improves the probability of the desired outcome. That means preserving decisions, constraints, evidence, and unresolved work—not blindly preserving every byte the agent has seen.

A practical harness can combine:

- stable instructions at the beginning for cache locality;
- structured state for facts, decisions, and completed work;
- retrieval for details that can be loaded on demand;
- summaries for old interactions whose exact wording no longer matters;
- references to large tool outputs rather than repeated inline copies;
- explicit invalidation when files, policies, or external data change.

Compaction is lossy by design. The harness therefore needs to know which information may be summarised and which must remain exact. A database identifier, test failure, legal clause, or user prohibition should not be softened into a vague memory.

### Make tools easy to select and cheap to call

Every tool made visible to the model consumes context and expands the decision surface. Two overlapping tools with unclear descriptions increase the chance of a wrong selection. A tool that returns an entire document when the agent needs one field increases both latency and future context cost.

Efficient tools have narrow contracts, unambiguous names, concise outputs, and explicit side-effect semantics. They support filtering and pagination. Read operations are separated from writes. Retry-safe actions carry idempotency keys. Errors explain what can be corrected instead of inviting the model to repeat the same call unchanged.

Tool search can also be hierarchical: expose a small catalogue first, then load the full schema only when a tool becomes relevant. The goal is not to give the model every possible capability at every step. It is to make the next useful capability easy to discover.

### Parallelise independence, preserve dependency

An agent often performs sequential work simply because its loop is sequential. It searches one source, reads it, searches another, and reads that—even when the searches are independent.

If three operations take two seconds each, executing them sequentially creates a six-second critical path. Executing them concurrently can approach two seconds, plus orchestration overhead. Total compute may be similar, but the user waits less.

Parallelism is appropriate when tasks do not depend on one another: retrieving independent sources, running separate tests, or analysing unrelated files. It is inappropriate when order carries state, when operations mutate the same resource, or when the second step requires evidence from the first.

[Anthropic’s agent architecture guide](https://www.anthropic.com/engineering/building-effective-agents) makes the same distinction between workflows such as routing, parallelisation, orchestrator-workers, and evaluator-optimizer loops. Each pattern pays for additional coordination. It should be used because the task structure demands it, not because “more agents” sounds more capable.

### Remember progress outside the prompt

A reliable harness should not require the model to reconstruct task state from prose on every turn. Durable state can record:

- the current objective and acceptance criteria;
- completed steps and their outputs;
- file or data versions already inspected;
- tool calls and their idempotency identifiers;
- failed approaches that should not be repeated;
- remaining work and the reason it remains.

This turns “do not do the same work twice” from a hopeful instruction into a system property. It also enables recovery. If a long task is interrupted after step seventeen, the agent can resume from a checkpoint instead of replaying the first sixteen steps and hoping for the same trajectory.

### Stop when the outcome is accepted

Agents need explicit completion criteria. Without them, a loop may stop too early because the answer looks plausible, or continue polishing after the task is already complete.

The stop condition should be observable: tests pass, required fields are present, cited evidence supports the claims, the requested file exists, or the user-approved action has completed. A verifier is useful only when it checks a defined contract. An open-ended “review your work again” loop can consume unlimited inference without converging.

Retries need budgets and classifications as well. A transient network failure may deserve backoff. Invalid tool arguments may deserve one corrected attempt. A permissions failure should stop and request authority. A deterministic validation error should not be retried unchanged.

The harness earns its keep by preventing expensive intelligence from being spent on work the system already knows cannot succeed.

## The Gains Compound

The three layers are not independent add-ons. They multiply one another.

Consider an illustrative research agent. The baseline uses twelve model calls, processes 18,000 input tokens across the trajectory, generates 4,000 output tokens, and performs eight tool calls. Now imagine four changes:

- better task routing removes two unnecessary model calls;
- context management reduces repeated input by 30 percent;
- prompt-prefix reuse accelerates the stable portion of the remaining calls;
- independent retrieval steps run concurrently.

No single change is a revolution. Together they alter the shape of the entire path: fewer calls, fewer processed tokens per call, less repeated prefill work, and a shorter critical path. If the model also becomes better at tool use, the acceptance rate may rise, eliminating repair loops that were not visible in per-call benchmarks.

The reverse compounding is equally important. A weak tool contract creates invalid calls. Invalid calls create retries. Retries append error messages to context. Larger context increases prefill work. Longer runs occupy cache memory and queue capacity. Queue pressure worsens tail latency for unrelated users.

One small design flaw in the harness can become an infrastructure problem.

This is why local optimisation is dangerous. The model team can celebrate a benchmark gain while the product’s task-completion rate falls. The serving team can maximise accelerator utilisation while interactive latency deteriorates. The agent team can add a verifier that improves quality by one point but doubles calls for every request.

All three teams need a shared measurement boundary.

## A Practical Optimisation Order

When an AI product is slow or expensive, replacing the model is rarely the best first diagnostic step. Start by making the path visible.

### 1. Trace one accepted outcome end to end

Record model, tokens, cache hits, queue time, time to first token, generation time, tool latency, retries, errors, and the final acceptance decision under one request identifier. Aggregate metrics without traces tell you that the system is slow; a trace tells you where.

### 2. Remove work before accelerating it

Look for duplicated retrieval, repeated context, unused tool output, redundant verification, retries without changed inputs, and model calls whose result can be computed deterministically. The fastest token is the one the system never needs to process.

### 3. Shorten the critical path

Parallelise independent reads, stream useful partial output, preload predictable resources, and separate interactive work from batch queues. Do not confuse lower latency with lower total compute.

### 4. Improve reuse

Arrange stable prompt prefixes, cache immutable retrieval results, reuse structured state, and checkpoint long workflows. Define invalidation rules before relying on a cache.

### 5. Match capability to difficulty

Route only after the workload has clear classes and the quality floor is measurable. Compare the full path cost, including classification, fallbacks, and misroutes—not merely the first-call price.

### 6. Tune the serving layer under representative load

Benchmark realistic prompt lengths, output lengths, concurrency, tool pauses, and service tiers. Measure the tail, not just the average. An inference configuration that wins on isolated short prompts may lose under long-context agent traffic.

### 7. Re-run evals after every efficiency change

Quantisation, context compaction, routing, smaller models, fewer verifiers, and tighter retry budgets can all change behaviour. Cost and latency improvements count only if the application continues to meet its acceptance contract.

## Intelligence Is the Budget; Architecture Decides How It Is Spent

The AI industry naturally focuses on models. Models are where new capabilities appear, and capability is the most visible part of the product. But once a model enters a real application, its intelligence is mediated by a system.

The inference layer decides how efficiently each token is computed. The harness decides which tokens need to be computed at all. The product succeeds only when those decisions produce an acceptable outcome at a cost and speed the user can tolerate.

The restaurant analogy has one final implication. Hiring a better chef can transform the menu. It does not remove the need to organise the kitchen, prepare ingredients, coordinate staff, and know when an order is complete. In fact, the more capable—and expensive—the chef becomes, the more valuable good organisation becomes.

The future of efficient AI will not be one smaller model, one faster kernel, or one clever agent framework. It will be co-design across all three layers: models that expose useful capability, inference systems that execute it with minimal waste, and harnesses that spend it only where it changes the outcome.

That is what it means to build intelligence, efficiently.
