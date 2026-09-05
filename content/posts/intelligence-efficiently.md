---
title: "Intelligence, Efficiently"
date: 2026-09-05
lastmod: 2026-09-05
description: "How model capability, inference engineering, and agent design combine to make frontier AI faster, cheaper, and more useful."
summary: "A systems view of frontier AI efficiency, from tokens and GPU kernels to prompt caching, tool discovery, and agent loops."
tags: ["AI Infrastructure", "Inference", "AI Agents", "Efficiency"]
draft: false
---

You can think of an AI system as a restaurant.

**The model is the chef.** A better chef can handle harder dishes. In AI, that means difficult coding, complex reasoning, research, tool use, and work that unfolds across many steps.

**The inference system is the kitchen.** It includes the ovens, refrigerators, workstations, order queue, and the way ingredients are prepared. Put an exceptional chef in a poorly organised kitchen and customers will still wait. Equipment sits idle in one corner while another station is overloaded. Ingredients are fetched twice. Tasks that could happen together are performed one after another.

**The agentic harness is the manager.** It decides which information the chef needs, which tools are available, what work has already been completed, and when the result is ready to leave the kitchen.

Most conversations about AI focus on the chef. A new model is more capable, scores higher on a benchmark, or solves a task that the previous generation could not. That progress matters, but it is only one part of the system.

The larger question is this:

> How do we make powerful intelligence fast and affordable enough to use at enormous scale?

The answer is not simply to make the model smaller. It is to improve the model, the inference system, and the agentic harness together. A few percent saved in one layer may look modest. Repeated across every token, every model call, and every step in a long agent run, those savings become substantial.

This is the central idea behind **frontier efficiency**. The goal is not less intelligence. The goal is to waste less of it.

## The Real Unit of Efficiency

**Frontier intelligence** describes capability near the leading edge of what current AI systems can do. It includes the ability to reason through unfamiliar problems, modify large codebases, conduct research, use tools reliably, and remain coherent across long workflows.

**Frontier efficiency** means reaching that level of capability with fewer resources. That can mean fewer input tokens, fewer generated tokens, less accelerator time, fewer model calls, lower latency, or a lower total cost for the completed task.

The distinction matters because price per token is not the same as price per result.

Consider two hypothetical models:

| Model | Price per million tokens | Tokens needed | Cost to finish |
| --- | ---: | ---: | ---: |
| Model A | $2.00 | 1,000,000 | $2.00 |
| Model B | $3.00 | 400,000 | $1.20 |

Model B is more expensive per token, yet cheaper per completed task. It uses its tokens more effectively. If both outputs meet the same quality bar, Model B has better **intelligence per token** and better **performance per dollar**.

This is why the useful unit of optimisation is not the first model call. It is the accepted outcome. A cheap call that produces an incomplete answer, triggers a retry, repeats a tool call, and eventually requires human repair may be the most expensive path through the system.

This is the same production accounting problem explored in [The Cheapest LLM Is Often the Most Expensive One](/posts/the-cheapest-llm-is-often-the-most-expensive-one/): the price of the first call and the cost of reaching a useful result are different quantities.

The same principle explains why GPT-5.6 is a family rather than one model for every request.

| Model | Role in the family | A practical fit |
| --- | --- | --- |
| [GPT-5.6 Sol](https://developers.openai.com/api/docs/models/gpt-5.6-sol) | Flagship capability for complex professional work | Difficult coding, reasoning, research, and long agent tasks |
| [GPT-5.6 Terra](https://developers.openai.com/api/docs/models/gpt-5.6-terra) | Balance between intelligence and cost | Everyday production work where quality and budget both matter |
| [GPT-5.6 Luna](https://developers.openai.com/api/docs/models/gpt-5.6-luna) | Efficiency for high volume workloads | Narrower, frequent, or latency sensitive tasks |

The principle is not **always use the strongest model**. It is **use the right amount of intelligence for the task**. Sending every request to the most capable option is like renting a supercomputer to calculate `2 + 2`. Sending a genuinely difficult task to an inadequate model is the opposite mistake. It may look cheap until the retries begin.

## The Model Is Only One Layer

Training a model does not produce answers by itself. Every time a user submits a prompt, machines must execute the trained model to generate a response. That execution process is **inference**.

At a simplified level, a language model performs a vast number of tensor operations. Those operations move data through accelerator memory, multiply matrices, build attention state, and generate tokens in sequence. Two systems can serve the same model and still deliver very different latency and cost because they organise this work differently.

The job of an inference stack is therefore not merely to keep the model running. It must turn a fixed amount of hardware into as much useful output as possible while preserving response quality, availability, and reliability.

This introduces a second frontier. Model researchers ask how much capability can be learned. Systems engineers ask how efficiently that capability can be executed.

Three constraints make the problem especially interesting.

First, requests are uneven. A short question with a long answer behaves differently from a long document with a short answer. Second, generated text is sequential. Each new token depends on the tokens before it. Third, memory is limited. The system must retain intermediate attention state for many active requests without allowing that state to crowd everything else off the accelerator.

These constraints are why inference optimisation is not one trick. It is a collection of decisions about routing, scheduling, memory, kernels, and prediction.

## Inside the Inference Kitchen

Imagine one thousand requests arriving at a cluster of accelerators. If one device receives one hundred requests while another receives five and a third receives none, the fleet may have enough total capacity and still feel slow.

**Load balancing** decides where each request should go. A useful routing decision may consider geography, available capacity, accelerator type, current load, prompt length, expected output length, and whether a reusable cache already exists on a particular worker. The best destination is not always the nearest machine or the emptiest machine. It is the machine that can complete this request efficiently without making the rest of the queue worse.

### Kernels determine how well the hardware is used

The model describes the computation. A **kernel** is code close to the hardware that carries out part of that computation on an accelerator.

Two kernels can produce the same mathematical result and have very different performance. One may move data unnecessarily between memory levels. Another may keep useful values close to the compute units. One may launch many small operations that leave the hardware waiting. Another may combine work and keep more of the device occupied.

This is why optimising AI systems is not only about reducing the number of calculations. Data movement, memory layout, parallel execution, and idle time can matter just as much.

In the engineering account behind this article, OpenAI reports using GPT-5.6 Sol and Codex to inspect production traffic, identify imbalances, propose routing strategies, find computation that could be prepared in advance, and help write production kernels. OpenAI reports that this work, combined with other kernel improvements, reduced its total serving cost by about 20 percent.

That figure should be read carefully. It is an internal result reported by OpenAI for its own serving stack. It is not a promise that every GPT-5.6 request, workload, or external application becomes exactly 20 percent cheaper.

### Speculative decoding lets a smaller model draft ahead

Normal language generation is mostly sequential:

```text
token 1
token 2
token 3
token 4
```

The model cannot fully generate token 4 before tokens 1 through 3 are known. That dependency limits how much of the process can run in parallel.

**Speculative decoding** changes the execution strategy. A smaller and faster draft model proposes several likely next tokens. The primary model then checks those proposals together. When the draft is correct, the system can accept several tokens while preserving the output behaviour of the primary model.

Think of a teacher checking a familiar sequence. Instead of writing `1, 2, 3, 4, 5, 6` one item at a time, the teacher lets a student draft the sequence and verifies it. When the student is usually right, verification can be faster than producing every item from scratch.

The benefit depends on how quickly the draft model runs and how often its proposals are accepted. A poor draft creates extra work. A good one reduces the expensive serial work performed by the primary model.

OpenAI reports that GPT-5.6 Sol helped design and run hundreds of experiments on its speculator models, including changes to their size, architecture, and features. It also helped monitor training and investigate hardware failures or instability. According to OpenAI, the resulting changes improved token generation efficiency by more than 15 percent in its internal system.

Again, this is a reported systems result, not a universal speed guarantee. Its broader importance is the method: the model being served can also help improve the mechanism that serves it.

### The KV cache is working memory for generation

Suppose a user provides twenty pages of text and asks for a summary. The model should not reprocess all twenty pages from the beginning every time it generates another word.

Transformers retain intermediate attention information in a **key value cache**, usually called the KV cache. During the initial processing of uncached input, the system builds this state. While generating output, it repeatedly reads and extends the state.

The KV cache saves computation, but it consumes memory and grows with the sequence. That creates a scheduling problem. Consider two requests:

| Workload | Input | Output |
| --- | ---: | ---: |
| Short prompt, long generation | 100 tokens | 2,000 tokens |
| Long prompt, short answer | 100,000 tokens | 20 tokens |

These requests stress the system differently. The first spends much of its time generating and extending state. The second performs a large initial pass and then finishes quickly. One combination of batching, sharding, and KV cache management will not be optimal for both.

A strong serving system adapts to workload shape. It does not use the same gear for city traffic, a mountain road, and a motorway.

## The Harness Decides How Much Work Exists

The inference system makes a model call efficient. The agentic harness decides how many model calls exist in the first place.

A chatbot may receive a prompt and return one answer. An agent can read files, search the web, inspect deployment history, call an API, edit code, run tests, review the result, and repeat. One user request can become dozens of model calls and tool operations.

This changes the economics. If a small inefficiency adds half a second to one chatbot response, the user may not notice. If it appears in fifty steps, it adds 25 seconds to the path. If a repeated tool definition adds several thousand tokens to every model call, the cost grows with every turn.

The harness is therefore an efficiency layer, not mere glue code.

### Context is a budget

An agent may have access to one hundred tools, dozens of skills, several plugins, a long conversation history, and a library of documents. Sending all of them to the model on every turn feels comprehensive. It also creates **context bloat**.

More context means more tokens to process. It can also make the relevant evidence harder to identify, especially when stale plans, repeated tool output, and unrelated instructions remain in view.

The right distinction is not small context versus large context. It is **useful context versus unnecessary context**.

One solution is deferred discovery. The model initially sees a compact description of what is available. Detailed definitions are loaded only when they become relevant. If the user asks for spreadsheet analysis, the agent does not need the complete schemas for every email, music, project management, and design integration.

OpenAI's [tool search documentation](https://developers.openai.com/api/docs/guides/tools-tool-search) describes this pattern directly. Deferred tools can be discovered and loaded at runtime, which avoids placing every tool definition in the context at the beginning. Newly discovered tools are added at the end of the context so earlier content remains stable.

That last detail connects context management to caching.

### Reuse the prefix instead of computing it again

Agent requests often share a large beginning:

```text
system instructions
tool definitions
conversation history
new user or tool result
```

**Prompt caching** allows the system to reuse work for a matching prefix instead of processing the same prefix again. The first request pays to construct the reusable state. Later requests can benefit when their opening tokens remain compatible with it.

The sequence matters. If new information is inserted near the beginning, the shared prefix ends early. If tool definitions appear in a different order on every turn, the token sequence changes even though the available tools are logically identical.

This is why histories that grow by appending new information, along with deterministic tool ordering, can reduce cost. Stable information stays in place. Tools appear in a consistent order. The architecture gives the cache something reliable to recognise.

The current [OpenAI prompt caching guide](https://developers.openai.com/api/docs/guides/prompt-caching) goes further by supporting explicit cache breakpoints for reusable content. It also notes an important tradeoff: compaction can shorten context while reducing cache reuse because it changes the prefix. A lower cache hit rate can still be worthwhile when the total number of input tokens falls enough.

This is a good example of why one metric is not sufficient. Maximising cache hits is not the goal. Minimising total cost while preserving the outcome is the goal.

## Small Improvements Compound

The most important word in the entire discussion is **compounding**.

Suppose an agent begins with an illustrative cost index of 100. Four independent changes reduce the remaining cost:

| Improvement | Illustrative reduction |
| --- | ---: |
| Better kernels | 20 percent |
| Better cache reuse | 15 percent |
| Better routing | 10 percent |
| Fewer model calls | 20 percent |

The reductions should not be added as if they remove 65 points from the original cost. Each acts on what remains:

```text
100 × 0.80 × 0.85 × 0.90 × 0.80 ≈ 49
```

In this deliberately simplified example, no single optimisation cuts the cost in half, but their combination does.

Compounding also works in the wrong direction. An unclear tool description causes a failed call. The failure triggers a retry. The error and retry enlarge the context. The larger context takes longer to process. The longer run retains KV cache memory and queue capacity. A small harness flaw has become an inference problem.

OpenAI's [latency optimisation guide](https://developers.openai.com/api/docs/guides/latency-optimization) reflects this systems view. Its principles include processing tokens faster, generating fewer tokens, using fewer input tokens, making fewer requests, parallelising independent work, improving perceived waiting time, and avoiding an LLM when deterministic software is sufficient.

The fastest model call is useful. A model call the system never needed is better.

## A Case Study in AI Improving AI

The most interesting part of the GPT-5.6 story is not one kernel or one cache policy. It is the feedback loop.

OpenAI describes GPT-5.6 Sol and Codex participating in several parts of the optimisation process:

1. Analyse production traffic.
2. Identify routing bottlenecks and workload imbalance.
3. Propose and evaluate new routing strategies.
4. Find computation that can be prepared, removed, or executed in parallel.
5. Write and improve accelerator kernels.
6. Design experiments for speculative models.
7. Monitor training and investigate failures.
8. Tune inference configurations for different workload shapes.

The model is both the object being optimised and a tool used in the optimisation.

That creates a reinforcing loop:

```text
more capable AI
        ↓
faster engineering and experimentation
        ↓
better models, kernels, routing, and harnesses
        ↓
lower cost and latency
        ↓
more experiments become affordable
        ↓
more capable AI
```

This does not mean the loop runs automatically. Engineers still define goals, design measurements, review changes, and decide what enters production. A model can propose a faster kernel that is numerically wrong, a routing policy that improves average latency while damaging the tail, or a cache strategy that violates an isolation boundary. Greater automation raises the value of good evaluation rather than removing it.

The strategic change is that AI can now accelerate parts of the work required to make AI itself more efficient. If that capability improves, the pace of systems optimisation may rise with model capability instead of depending only on human engineering time.

## How to Measure Frontier Efficiency

Efficiency claims are easy to misunderstand because the denominator can be changed until almost any result looks favourable.

Tokens per second measure generation speed. They do not tell us whether the answer is correct. Price per million tokens measures a tariff. It does not tell us how many tokens the task requires. Average latency can hide a painful tail. GPU utilisation can rise while users wait longer in a queue.

A more complete measurement should include four dimensions:

| Dimension | Useful questions |
| --- | --- |
| Quality | Did the result meet the acceptance criteria? |
| Cost | What did the complete path consume, including retries and tools? |
| Latency | How long did the user wait, including queueing and every agent step? |
| Reliability | How often did the system finish correctly before the deadline? |

The unit should be an **accepted outcome**. For a coding agent, that may mean the requested change is complete and the relevant tests pass. For research, it may mean the answer is supported by valid sources. For customer support, it may mean the issue is resolved without an incorrect action or unnecessary escalation.

That quality boundary needs evaluation at the outcome, trajectory, and operational levels. I cover that process in more detail in [Evals from First Principles](/posts/evals-from-first-principles/).

This also clarifies how to interpret vendor numbers. OpenAI's reported serving cost and token generation improvements are useful evidence about the direction and scale of its internal engineering work. They are not independent benchmarks, and they do not predict every application. Real results depend on the selected model, reasoning effort, context length, output length, cache behaviour, tool calls, concurrency, and workload distribution.

The only reliable way to know whether an optimisation works for a product is to test it against representative tasks and traffic that resembles production.

## Intelligence Is the Budget

For years, the simplest story of AI progress was that larger models produced more intelligence. That story is no longer enough.

As AI moves from single responses to long agent workflows, the surrounding system becomes increasingly important. The model determines what may be possible. The inference stack determines how efficiently each token is computed. The harness determines which tokens, tools, and model calls are necessary at all.

The restaurant analogy now comes back into focus. A better chef expands the menu, but the restaurant still needs an organised kitchen and a competent manager. In fact, the more valuable the chef's time becomes, the more expensive waste becomes around them.

The next frontier is therefore not only more intelligence. It is more useful work from every token, every accelerator, every model call, every second, and every dollar.

That is what it means to build intelligence, efficiently.
