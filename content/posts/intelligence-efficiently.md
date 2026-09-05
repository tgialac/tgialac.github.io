---
title: "Intelligence, Efficiently"
date: 2026-09-05
lastmod: 2026-09-05
description: "How model capability, inference engineering, and agent design combine to make frontier AI faster, cheaper, and more useful."
summary: "A systems view of frontier AI efficiency, from tokens and GPU kernels to prompt caching, tool discovery, and agent loops."
tags: ["AI Infrastructure", "Inference", "AI Agents", "Efficiency"]
draft: false
---

<figure class="article-figure article-figure-hero">
  <img src="/images/covers/intelligence-efficiently.png" width="1672" height="941" loading="eager" fetchpriority="high" decoding="async" alt="An organised modern restaurant kitchen where a head chef, kitchen staff, and manager coordinate work across specialised stations.">
  <figcaption>Frontier AI is a systems problem. The chef, the kitchen, and the manager become efficient only when they are designed to work together.</figcaption>
</figure>

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

This also changes how we should interpret the phrase **more work per token**. It does not mean compressing every answer until it becomes terse. A short answer that omits the decisive fact is not efficient. A long chain of reasoning that prevents three failed attempts may be. The useful question is whether each token increases the probability of reaching the required result.

That probability creates a quality floor. Below the floor, lower cost is false economy. Above it, additional reasoning may have diminishing returns. The efficient operating point lies between those extremes: enough intelligence to solve the task reliably, without spending capability where it no longer changes the outcome.

This is the same production accounting problem explored in [The Cheapest LLM Is Often the Most Expensive One](/posts/the-cheapest-llm-is-often-the-most-expensive-one/): the price of the first call and the cost of reaching a useful result are different quantities.

The same principle explains why GPT-5.6 is a family rather than one model for every request.

| Model | Role in the family | A practical fit |
| --- | --- | --- |
| [GPT-5.6 Sol](https://developers.openai.com/api/docs/models/gpt-5.6-sol) | Flagship capability for complex professional work | Difficult coding, reasoning, research, and long agent tasks |
| [GPT-5.6 Terra](https://developers.openai.com/api/docs/models/gpt-5.6-terra) | Balance between intelligence and cost | Everyday production work where quality and budget both matter |
| [GPT-5.6 Luna](https://developers.openai.com/api/docs/models/gpt-5.6-luna) | Efficiency for high volume workloads | Narrower, frequent, or latency sensitive tasks |

The principle is not **always use the strongest model**. It is **use the right amount of intelligence for the task**. Sending every request to the most capable option is like renting a supercomputer to calculate `2 + 2`. Sending a genuinely difficult task to an inadequate model is the opposite mistake. It may look cheap until the retries begin.

Model choice is only one control. Reasoning effort, tool access, context length, and stopping criteria also determine how much intelligence is allocated. A useful production system can route routine extraction to a fast model, send ambiguous analysis to a balanced model, and reserve the most capable model for cases where the evidence or consequences justify it. The route can change when a task becomes harder than it first appeared.

This makes frontier efficiency a resource allocation problem. Intelligence is not a switch that is either on or off. It is a budget that can be directed toward the moments where it has the highest marginal value.

## The Model Is Only One Layer

Training a model does not produce answers by itself. Every time a user submits a prompt, machines must execute the trained model to generate a response. That execution process is **inference**.

At a simplified level, a language model performs a vast number of tensor operations. Those operations move data through accelerator memory, multiply matrices, build attention state, and generate tokens in sequence. Two systems can serve the same model and still deliver very different latency and cost because they organise this work differently.

The job of an inference stack is therefore not merely to keep the model running. It must turn a fixed amount of hardware into as much useful output as possible while preserving response quality, availability, and reliability.

This introduces a second frontier. Model researchers ask how much capability can be learned. Systems engineers ask how efficiently that capability can be executed.

Three constraints make the problem especially interesting.

First, requests are uneven. A short question with a long answer behaves differently from a long document with a short answer. Second, generated text is sequential. Each new token depends on the tokens before it. Third, memory is limited. The system must retain intermediate attention state for many active requests without allowing that state to crowd everything else off the accelerator.

These constraints are why inference optimisation is not one trick. It is a collection of decisions about routing, scheduling, memory, kernels, and prediction.

It helps to divide a request into two broad phases. **Prefill** processes the input prompt and constructs the initial attention state. **Decode** generates the response token by token. A long document with a one sentence answer is dominated by prefill. A tiny prompt that asks for a long story is dominated by decode.

This distinction explains why two users can describe the same system in opposite ways. One experiences a long pause followed by a fast answer. Another sees the first word immediately but waits while the answer unfolds. The first is sensitive to **time to first token**. The second is sensitive to **inter token latency** and total generation time.

Throughput adds another perspective. A service may generate more total tokens per second across its fleet while an individual user waits longer in a queue. High utilisation is economically attractive, but pushing utilisation too far can make latency unstable. Efficient serving is therefore a balancing act between hardware utilisation, per request responsiveness, and predictable tail latency.

## Inside the Inference Kitchen

Imagine one thousand requests arriving at a cluster of accelerators. If one device receives one hundred requests while another receives five and a third receives none, the fleet may have enough total capacity and still feel slow.

**Load balancing** decides where each request should go. A useful routing decision may consider geography, available capacity, accelerator type, current load, prompt length, expected output length, and whether a reusable cache already exists on a particular worker. The best destination is not always the nearest machine or the emptiest machine. It is the machine that can complete this request efficiently without making the rest of the queue worse.

That last condition is difficult because the future is partly unknown. The system sees the prompt length, but it may not know whether the answer will take twenty tokens or two thousand. It can estimate from the task, model, requested output limit, and historical traffic, then update scheduling decisions as work unfolds.

Routing also interacts with **cache locality**. Moving a request to a less busy worker may discard a reusable prefix or KV cache already present elsewhere. Keeping it on the original worker saves computation but may deepen that worker's queue. The scheduler must compare the cost of waiting with the cost of rebuilding state.

Average performance can conceal the worst failures. A routing policy might improve the median while allowing a small group of long requests to block everything behind them. Production systems therefore care about percentiles such as P95 and P99, not only the average. Users remember the request that stalled for a minute more vividly than the nine that completed in two seconds.

### Kernels determine how well the hardware is used

The model describes the computation. A **kernel** is code close to the hardware that carries out part of that computation on an accelerator.

Two kernels can produce the same mathematical result and have very different performance. One may move data unnecessarily between memory levels. Another may keep useful values close to the compute units. One may launch many small operations that leave the hardware waiting. Another may combine work and keep more of the device occupied.

This is why optimising AI systems is not only about reducing the number of calculations. Data movement, memory layout, parallel execution, and idle time can matter just as much.

Modern accelerators can perform enormous amounts of arithmetic, but computation is useful only when the required data arrives in time. A theoretically smaller operation may still be slower if it repeatedly moves values through expensive memory paths. Kernel engineering tries to keep data close to where it is consumed, combine compatible operations, and avoid synchronisation that leaves thousands of compute units waiting.

Some work can also be shifted out of the critical path. If part of a calculation depends only on model weights or stable configuration, it may be prepared before a user request arrives. The amount of mathematics does not necessarily change, but the amount performed while the user is waiting does.

In the engineering account behind this article, OpenAI reports using GPT-5.6 Sol and Codex to inspect production traffic, identify imbalances, propose routing strategies, find computation that could be prepared in advance, and help write production kernels. OpenAI reports that this work, combined with other kernel improvements, reduced its total serving cost by about 20 percent.

That figure should be read carefully. It is an internal result reported by OpenAI for its own serving stack. It is not a promise that every GPT-5.6 request, workload, or external application becomes exactly 20 percent cheaper.

### Speculative decoding lets a smaller model draft ahead

Normal language generation is mostly sequential:

| Step | What is known |
| --- | --- |
| Generate token 1 | The prompt |
| Generate token 2 | The prompt and token 1 |
| Generate token 3 | The prompt and tokens 1 to 2 |
| Generate token 4 | The prompt and tokens 1 to 3 |

The model cannot fully generate token 4 before tokens 1 through 3 are known. That dependency limits how much of the process can run in parallel.

**Speculative decoding** changes the execution strategy. A smaller and faster draft model proposes several likely next tokens. The primary model then checks those proposals together. When the draft is correct, the system can accept several tokens while preserving the output behaviour of the primary model.

Think of a teacher checking a familiar sequence. Instead of writing `1, 2, 3, 4, 5, 6` one item at a time, the teacher lets a student draft the sequence and verifies it. When the student is usually right, verification can be faster than producing every item from scratch.

The benefit depends on how quickly the draft model runs and how often its proposals are accepted. A poor draft creates extra work. A good one reduces the expensive serial work performed by the primary model.

The acceptance rate is only part of the equation. A draft model can be accurate but too slow to help. A very fast draft can be useless if most proposals are rejected. The length of each proposed block, the verification cost, and the shape of the workload all influence the result. There is no universally best speculator, just as there is no universally best kitchen station layout.

This is an important pattern in systems optimisation: an elegant technique on paper can lose to overhead in production. The relevant question is not whether speculation exists, but whether it reduces end to end cost and latency under real traffic.

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

Long contexts make this problem sharper. Keeping more attention state available can prevent recomputation, but memory devoted to one request cannot serve another. The system may page state between memory tiers, split work across devices, or reduce the number of simultaneous requests. Each choice trades computation, communication, memory pressure, and queue time.

This is why context windows should not be treated as free storage merely because a model supports them. Capacity is not the same as efficiency. A million token window can be extraordinarily useful when the task needs it. Filling that window with duplicated logs, stale tool results, or irrelevant documents simply turns available capacity into recurring cost.

## The Harness Decides How Much Work Exists

The inference system makes a model call efficient. The agentic harness decides how many model calls exist in the first place.

A chatbot may receive a prompt and return one answer. An agent can read files, search the web, inspect deployment history, call an API, edit code, run tests, review the result, and repeat. One user request can become dozens of model calls and tool operations.

This changes the economics. If a small inefficiency adds half a second to one chatbot response, the user may not notice. If it appears in fifty steps, it adds 25 seconds to the path. If a repeated tool definition adds several thousand tokens to every model call, the cost grows with every turn.

The harness is therefore an efficiency layer, not mere glue code.

### Context is a budget

An agent may have access to one hundred tools, dozens of skills, several plugins, a long conversation history, and a library of documents. Sending all of them to the model on every turn feels comprehensive. It also creates **context bloat**.

More context means more tokens to process. It can also make the relevant evidence harder to identify, especially when stale plans, repeated tool output, and unrelated instructions remain in view.

The right distinction is not small context versus large context. It is **useful context versus unnecessary context**.

Useful context changes the model's decision. Unnecessary context is present merely because the system knows how to retrieve it. That difference sounds obvious, but agents often accumulate information defensively. A full file is included when three lines would answer the question. A tool returns ten thousand rows when the next decision needs five. A plan remains in the history after the plan has already been executed.

Good context engineering has three jobs. It must select the right evidence, preserve enough state for continuity, and discard or compress information whose original form no longer matters. The aim is not to make the prompt minimal at any cost. It is to maintain the smallest faithful representation of the work.

One solution is deferred discovery. The model initially sees a compact description of what is available. Detailed definitions are loaded only when they become relevant. If the user asks for spreadsheet analysis, the agent does not need the complete schemas for every email, music, project management, and design integration.

OpenAI's [tool search documentation](https://developers.openai.com/api/docs/guides/tools-tool-search) describes this pattern directly. Deferred tools can be discovered and loaded at runtime, which avoids placing every tool definition in the context at the beginning. Newly discovered tools are added at the end of the context so earlier content remains stable.

That last detail connects context management to caching.

### Reuse the prefix instead of computing it again

Agent requests often share the same broad structure:

| Position | Typical content | Cache implication |
| --- | --- | --- |
| Beginning | System instructions and stable policies | Keep identical whenever possible |
| Middle | Tool definitions and durable conversation state | Use deterministic ordering |
| End | The newest user message or tool result | Append changing information here |

**Prompt caching** allows the system to reuse work for a matching prefix instead of processing the same prefix again. The first request pays to construct the reusable state. Later requests can benefit when their opening tokens remain compatible with it.

The sequence matters. If new information is inserted near the beginning, the shared prefix ends early. If tool definitions appear in a different order on every turn, the token sequence changes even though the available tools are logically identical.

This is why histories that grow by appending new information, along with deterministic tool ordering, can reduce cost. Stable information stays in place. Tools appear in a consistent order. The architecture gives the cache something reliable to recognise.

The current [OpenAI prompt caching guide](https://developers.openai.com/api/docs/guides/prompt-caching) goes further by supporting explicit cache breakpoints for reusable content. It also notes an important tradeoff: compaction can shorten context while reducing cache reuse because it changes the prefix. A lower cache hit rate can still be worthwhile when the total number of input tokens falls enough.

This is a good example of why one metric is not sufficient. Maximising cache hits is not the goal. Minimising total cost while preserving the outcome is the goal.

Compaction deserves special attention in long agent runs. Instead of carrying every raw observation forever, the harness can replace older material with a concise state: what has been established, what changed, which decisions were made, and what remains unresolved. This reduces future input, but a careless summary can delete the clue needed several steps later.

The right compaction policy is therefore task aware. Exact error messages may matter during debugging. The precise wording of a contract may matter during legal analysis. Repeated progress logs may not matter once their final status is known. Compression should preserve decision relevant information, not merely reduce token count.

The harness also determines when to stop. An agent that continues researching after the evidence is sufficient wastes tokens even if every individual call is fast. One that stops too early saves tokens but fails the task. Clear acceptance criteria give the system a boundary: continue while a material requirement is unresolved, then stop when the requested outcome has been verified.

## Small Improvements Compound

The most important word in the entire discussion is **compounding**.

Suppose an agent begins with an illustrative cost index of 100. Four independent changes reduce the remaining cost:

| Improvement | Illustrative reduction |
| --- | ---: |
| Better kernels | 20 percent |
| Better cache reuse | 15 percent |
| Better routing | 10 percent |
| Fewer model calls | 20 percent |

The reductions should not be added as if they remove 65 points from the original cost. Each acts on what remains. In this example, **100 × 0.80 × 0.85 × 0.90 × 0.80 is approximately 49**.

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

The model is both the object being optimised and a tool used in the optimisation. That creates a reinforcing cycle:

| Stage | Effect on the next stage |
| --- | --- |
| More capable AI | Accelerates engineering and experimentation |
| Faster experimentation | Produces better models, kernels, routing, and harnesses |
| Better systems | Lower cost and latency |
| Lower cost | Makes more experiments affordable |
| More experiments | Create the conditions for more capable AI |

This does not mean the loop runs automatically. Engineers still define goals, design measurements, review changes, and decide what enters production. A model can propose a faster kernel that is numerically wrong, a routing policy that improves average latency while damaging the tail, or a cache strategy that violates an isolation boundary. Greater automation raises the value of good evaluation rather than removing it.

The strategic change is that AI can now accelerate parts of the work required to make AI itself more efficient. If that capability improves, the pace of systems optimisation may rise with model capability instead of depending only on human engineering time.

## What Builders Should Optimise First

The number of available techniques can make efficiency work feel like a hunt for exotic infrastructure. In practice, the best first step is usually to trace one representative task from the user's request to the accepted result.

Write down every model call, every tool call, the input and output token counts, waiting time, retry, cache result, and human intervention. This simple trace reveals whether the real problem is an expensive model, an unnecessary loop, a giant tool response, a slow external API, or a result that repeatedly fails evaluation.

A useful optimisation order is:

1. **Define the accepted outcome.** Decide what success means before reducing anything. Otherwise the system can appear cheaper simply because it does less of the required work.
2. **Remove unnecessary calls.** Deterministic transformations, validation, routing, and data lookup often belong in ordinary software. Do not ask a model to rediscover a value the application already knows.
3. **Reduce irrelevant context.** Retrieve narrower evidence, cap tool output, remove duplication, and compact completed history while preserving facts needed later.
4. **Parallelise independent work.** Two searches that do not depend on each other can run together. A dependent chain cannot. Parallelism shortens the critical path without pretending dependencies do not exist.
5. **Stabilise reusable prefixes.** Keep instructions and tool definitions consistent, append changing state, and measure actual cache use.
6. **Route by difficulty.** Use the least costly model and reasoning effort that reliably clears the quality bar. Escalate when uncertainty, task complexity, or consequence warrants it.
7. **Tune the serving layer.** Once request structure is sound, optimise batching, memory, kernels, speculation, and routing for the workloads that actually dominate production.
8. **Run the evaluation again.** An optimisation is real only if accepted outcomes remain stable or improve.

The order matters. A ten percent faster model call cannot compensate for five model calls that should never have happened. Likewise, deleting context before defining what the task needs can create retries that erase the apparent saving.

There is also a distinction between **latency work** and **cost work**. Parallel execution can reduce wall clock time while consuming roughly the same total resources. Batching can improve throughput and cost while making one request wait slightly longer. A product must decide which constraint matters for each path instead of treating efficiency as one number.

Interactive coding, voice, and search experiences are sensitive to responsiveness. Offline document processing may tolerate more waiting in exchange for higher throughput. A safety review may rationally spend more compute because the cost of a missed issue is larger than the cost of inference. The efficient configuration is always relative to the job.

## How to Measure Frontier Efficiency

Efficiency claims are easy to misunderstand because the denominator can be changed until almost any result looks favourable.

Tokens per second measure generation speed. They do not tell us whether the answer is correct. Price per million tokens measures a tariff. It does not tell us how many tokens the task requires. Average latency can hide a painful tail. GPU utilisation can rise while users wait longer in a queue.

A more complete measurement should include several dimensions:

| Dimension | Example metric | Useful question |
| --- | --- | --- |
| Quality | Accepted outcome rate | Did the result meet the acceptance criteria? |
| Cost | Cost per accepted outcome | What did the complete path consume, including retries and tools? |
| Input efficiency | Input tokens per outcome | How much context was necessary? |
| Output efficiency | Output tokens per outcome | Did generation contribute useful work? |
| Orchestration | Model and tool calls per outcome | How much work did the harness create? |
| Responsiveness | Time to first token | How quickly did visible progress begin? |
| Completion latency | End to end P50, P95, and P99 | How long did the whole task take, including the tail? |
| Reliability | Completion rate before deadline | Did the system finish predictably? |
| Infrastructure | Throughput, utilisation, and cache reuse | Did hardware capacity become useful results? |

The unit should be an **accepted outcome**. For a coding agent, that may mean the requested change is complete and the relevant tests pass. For research, it may mean the answer is supported by valid sources. For customer support, it may mean the issue is resolved without an incorrect action or unnecessary escalation.

That quality boundary needs evaluation at the outcome, trajectory, and operational levels. I cover that process in more detail in [Evals from First Principles](/posts/evals-from-first-principles/).

This also clarifies how to interpret vendor numbers. OpenAI's reported serving cost and token generation improvements are useful evidence about the direction and scale of its internal engineering work. They are not independent benchmarks, and they do not predict every application. Real results depend on the selected model, reasoning effort, context length, output length, cache behaviour, tool calls, concurrency, and workload distribution.

The only reliable way to know whether an optimisation works for a product is to test it against representative tasks and traffic that resembles production.

This requires paired measurements. Compare systems on the same task distribution, under similar concurrency, with the same acceptance standard. Report both the centre and the tail. Separate cached from uncached traffic. Record failures and retries instead of excluding them as anomalies.

Most importantly, resist metric substitution. If the business needs resolved support cases, tokens per second is a diagnostic metric, not the objective. If users need correct code changes, a lower price per call is useful only when the change still works. Infrastructure metrics explain the machine. Outcome metrics explain whether the machine is worth operating.

## Intelligence Is the Budget

For years, the simplest story of AI progress was that larger models produced more intelligence. That story is no longer enough.

As AI moves from single responses to long agent workflows, the surrounding system becomes increasingly important. The model determines what may be possible. The inference stack determines how efficiently each token is computed. The harness determines which tokens, tools, and model calls are necessary at all.

The restaurant analogy now comes back into focus. A better chef expands the menu, but the restaurant still needs an organised kitchen and a competent manager. In fact, the more valuable the chef's time becomes, the more expensive waste becomes around them.

The next frontier is therefore not only more intelligence. It is more useful work from every token, every accelerator, every model call, every second, and every dollar.

That is what it means to build intelligence, efficiently.
