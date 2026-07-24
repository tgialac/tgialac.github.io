---
title: "Tokens for Thought"
date: 2026-07-24T09:00:00+07:00
tags: ["ai", "llm", "economics", "inference", "agents", "systems"]
math: true
summary: "Tokens are not value. They are the meter on a production system that turns computation, memory, latency, authority, and risk into decisions. AI tokenomics is the discipline of designing that meter well."
---

> *A token is not a unit of intelligence. It is the meter on an intelligent production system.*

An agent can solve a difficult problem for a few cents. It can also burn millions of tokens producing an answer nobody uses. Both events are recorded in the same unit: tokens. Only one creates value.

That is why AI tokenomics should be more than an API price sheet. It is the design discipline for deciding where to buy another unit of machine thought, where to stop, and how to make the result economically and operationally defensible.

The central claim of this essay is simple:

> **Tokens are priced production inputs. They create value only when they improve a decision, an action, or a risk outcome by more than they cost.**

A token is closer to a kilowatt-hour than to money. It is a measured consumption unit. Its usefulness depends on the system that turns it into an accepted outcome.

## The map: from tokens to value

A serious LLM request spans at least five layers. An optimisation that ignores one of them is usually a local optimum with a nicer dashboard.

| Layer | The economic question | The common mistake |
| --- | --- | --- |
| **Tokens** | How many input, output, cached, hidden-reasoning, and tool-context tokens were used? | Counting only visible output. |
| **Serving system** | How do GPU memory, batching, networking, and SLOs turn tokens into cost and latency? | Treating a token as a fixed quantity of compute or time. |
| **Model policy** | Which model, route, cache, or verifier achieves the required quality at lowest total cost? | Sending every task to the strongest model. |
| **Workflow** | Which extra token changes the probability of a successful business outcome? | Minimising cost per request rather than cost per accepted outcome. |
| **Market and governance** | Who pays, captures surplus, bears risk, and controls access? | Equating API list price with provider marginal cost or user value. |

Quanyan Zhu's recent framework is useful precisely because it connects the technical and economic layers: tokens touch information processing, compute, memory, energy, price, allocation, and value. Its most important warning is that token expenditure and economic value are distinct variables; productivity, workflow position, hidden reasoning, risk, and downstream propagation all matter. [Zhu, *AI Tokenomics* (2026)](https://arxiv.org/abs/2606.24616)

## 1. Do not confuse price, cost, and value

For an API user, the visible bill is often approximately

$$
C_{\text{bill}} = p_{\text{in}}T_{\text{in}} + p_{\text{out}}T_{\text{out}} + p_{\text{cache}}T_{\text{cache}} + C_{\text{tools}}.
$$

This is an invoice, not the provider's physical cost, and certainly not the user's value. A more useful workflow-level account is

$$
C_{\text{run}} = \sum_{k\in\text{calls}} C_{\text{bill},k}
+ C_{\text{retrieval}} + C_{\text{tool}} + C_{\text{human}} + C_{\text{failure}}.
$$

The neglected term is usually $C_{\text{failure}}$: rework, a wrongly routed case, a refund, a delayed operation, a bad decision, or lost trust. Once an agent can act, that term can dominate the token bill.

For a deliberately simplified binary decision, expected value is

$$
\mathbb{E}[V] = qG - (1-q)L - C_{\text{run}},
$$

where $q$ is the probability that the decision is correct or accepted, $G$ is the gain when it is right, and $L$ is the loss when it is wrong. An extra token that changes $q$ by a small amount can be rational for a high-loss decision; a long reasoning trace can be wasteful for a low-value task that software can verify deterministically.

This motivates a production metric that is harder to game:

$$
\text{CPAO} = \frac{C_{\text{total over a cohort}}}{\#\,\text{accepted or successfully completed outcomes}}.
$$

Cost per accepted outcome does not replace quality metrics. It makes them economically accountable. A very cheap request is not cheap if one in three outputs must be redone. A costly case may be excellent if it prevents a much larger loss.

### Prices are falling, but not uniformly

The frontier has moved fast. Epoch AI estimates that the price needed to achieve GPT-4-level performance on GPQA Diamond fell by roughly 40-fold per year in its historical comparison; across benchmarks and milestones, its estimates range from 9-fold to 900-fold annually. The authors explicitly warn that the fastest recent declines may not persist. [Epoch AI (2025)](https://epoch.ai/data-insights/llm-inference-price-trends)

That does not mean AI becomes free. It means the frontier moves and demand may expand in response. A 2026 preprint estimates roughly a 600-fold decline in token prices from 2020 to 2026, while finding that flagship reasoning services do not fit a simple exponential trend. Treat this as a useful hypothesis about market segmentation, not a settled law. [Du, *Tiered Super-Moore's Law* (2026, preprint)](https://arxiv.org/abs/2603.28576)

The strategic implication is not to lock in a price forecast. It is to build a system that remains healthy when models, prices, and demand distributions change.

## 2. A token does not have one technical cost

Transformer serving has two distinct phases.

- **Prefill** reads the prompt and constructs the key-value cache. With full attention, attention work grows roughly quadratically with context length during this phase.
- **Decode** generates one token at a time. Each new token attends over accumulated context, so the cost of a decoding step grows with sequence length and is often limited by memory bandwidth rather than raw FLOPs.

A useful approximation for KV-cache memory is

$$
M_{\text{KV}} \approx 2LSH_{\text{KV}}d_hb \quad \text{bytes},
$$

where $L$ is the number of layers, $S$ is sequence length, $H_{\text{KV}}$ is the number of KV heads, $d_h$ is head dimension, and $b$ is bytes per element. Architecture, quantisation, and sharing change the coefficient, but not the conclusion: **long context is live memory debt for the duration of a request.**

Two workloads with the same token total can therefore have radically different cost, time-to-first-token (TTFT), time-per-output-token (TPOT), and GPU occupancy. A long prompt with a short answer is not economically equivalent to a multi-turn chat with a long decode.

PagedAttention made this concrete by treating KV cache like paged virtual memory, reducing fragmentation and enabling sharing. Its vLLM evaluation reported 2--4 times higher throughput at comparable latency than the systems it compared against, with larger gains for longer sequences and more complex decoding. [Kwon et al., *PagedAttention* (2023)](https://arxiv.org/abs/2309.06180)

DistServe separates prefill from decode because colocating them creates interference and couples resource-allocation choices that have different latency objectives. Its evaluation reported up to 7.4 times more requests served or 12.6 times tighter latency SLOs under the evaluated workloads. [Zhong et al., *DistServe* (OSDI 2024)](https://www.usenix.org/system/files/osdi24-zhong-yinmin.pdf)

Ege Erdil formalises the broader point: at a given cost per token, serial generation speed is jointly constrained by arithmetic, memory bandwidth, network bandwidth, latency, batch size, and parallelism. Deployment produces a speed--cost Pareto frontier, not a fixed conversion rate from tokens to money. [Erdil, *Inference economics of language models* (2025)](https://arxiv.org/abs/2506.04645)

### Software changes the frontier too

The token bill is not the only lever. ORCA introduced iteration-level scheduling and selective batching for generative serving; on its GPT-3 175B evaluation it reported 36.9 times the throughput of FasterTransformer at comparable latency. The number is workload-specific, but the economic lesson is general: scheduling can convert the same model and same hardware into a different cost--latency frontier. [Yu et al., *ORCA* (OSDI 2022)](https://www.usenix.org/conference/osdi22/presentation/yu)

Speculative decoding makes the same point from another direction. A small draft model proposes several tokens; the large target model verifies them in parallel, preserving the target distribution. Leviathan, Kalman, and Matias reported a 2--3 times speedup on T5-XXL without retraining or changing outputs. [*Fast Inference from Transformers via Speculative Decoding* (2023)](https://arxiv.org/abs/2211.17192) This is why a list price or a benchmark score never fully specifies deployment economics.

### Goodput, not token throughput

Users do not buy cluster tokens per second. They buy a usable response on time. The operative metric is

$$
\text{Goodput} = \#\{\text{requests that meet quality and SLO}\}/\text{time}.
$$

A job that generates twenty thousand tokens and times out creates no goodput. A router that halves spend but pushes p95 latency beyond an SLA can destroy value. The correct decision is constrained optimisation:

$$
\min C \quad \text{s.t.}\quad Q\ge Q^*,\; \text{TTFT}_{p95}\le\tau_1,\; \text{TPOT}_{p95}\le\tau_2,\; R\le R^*.
$$

## 3. Training tokens, inference tokens, and volume

Chinchilla altered training intuition: under a fixed compute budget, model size and training-token count should scale together, and many earlier large models were undertrained. [Hoffmann et al., *Training Compute-Optimal Large Language Models* (2022)](https://arxiv.org/abs/2203.15556) But training-optimal is not deployment-optimal.

If a model is invoked $N$ times, a simplified lifecycle objective is

$$
C_{\text{lifecycle}}(\theta)=C_{\text{train}}(\theta)+N\,C_{\text{infer}}(\theta).
$$

At large $N$, a smaller model trained longer can beat a larger, Chinchilla-optimal model because inference repeats. Sardana et al. extend scaling laws to include inference demand and find that at roughly one billion requests, a smaller, longer-trained model can be optimal; their experiments also show continued quality gains at very high tokens-per-parameter ratios. [*Beyond Chinchilla-Optimal* (2024/2025)](https://arxiv.org/abs/2401.00448)

Training does not merely create capability. It creates the marginal cost curve for every future request. Demand forecasts, context distributions, latency targets, and cache locality therefore belong in model decisions from the beginning, not in a later FinOps spreadsheet.

## 4. Test-time compute: when is another thought worth buying?

Reasoning models make one old fact vivid: compute can be spent at inference time rather than entirely embedded in model weights. But “more thinking” is not a policy.

Let $b_i$ be the token or compute budget for prompt $i$, and let $V_i(b_i)$ be its expected value after risk. For a cohort budget $B$:

$$
\max_{b_1,\ldots,b_n}\sum_iV_i(b_i)
\quad \text{s.t.}\quad \sum_ic_i(b_i)\le B.
$$

At an interior optimum,

$$
\frac{\partial V_i/\partial b_i}{\partial c_i/\partial b_i}=\lambda.
$$

Allocate the next unit of compute to the request with the highest marginal value per marginal cost until it falls to the shadow price $\lambda$. Easy prompts should stop early. Difficult, high-value prompts may justify search, tools, or a verifier. Difficult but low-value prompts should often abstain or escalate rather than think indefinitely.

Snell et al. show why this must be prompt-conditional: the effectiveness of test-time scaling methods varies strongly with prompt difficulty. In their setting, adaptive compute allocation was more than four times as efficient as best-of-$N$. [*Scaling LLM Test-Time Compute Optimally* (2024)](https://arxiv.org/abs/2408.03314)

The production problem is that $V_i$ and difficulty are unobserved. Entropy, self-consistency, verifier scores, tool-result disagreement, and router scores are proxies. They require calibration on the actual task distribution.

Fixed reasoning caps fail in two directions. *Plan and Budget* names them overthinking on simple problems and underthinking on hard ones; it introduces a Budget Allocation Model and E3, a correctness--efficiency metric. [Lin et al. (2025/2026)](https://arxiv.org/abs/2505.16122)

A practical stopping policy does not need to inspect private chain-of-thought. It can stop or escalate when marginal expected value falls below cost plus latency penalty; deterministic or grounded checks pass; samples disagree persistently; or a token, tool, or action budget is reached. A budget is a conditional spending right, not an arbitrary `max_tokens` value.

## 5. Routing, cascades, caching, and substitution

Not every request deserves the same model. A router should choose among a small model, large model, retrieval, deterministic code, or human review according to conditional utility:

$$
a^*(x)=\arg\max_a\;\mathbb{E}[V\mid x,a]-C(a)-\rho R(x,a).
$$

The risk price $\rho$ differs by use case. A trading, medical, or payment workflow should not share a routing policy with an internal chat assistant merely because both consume text tokens.

A **cascade** is the simple implementation: try a cheaper option, then escalate only when a calibrated confidence or verifier threshold fails. FrugalGPT combines prompt adaptation, approximation, and cascades; in its evaluated tasks it matched the strongest individual model with up to 98% lower cost, or improved accuracy at the same cost. That is a result for its setting, not a production guarantee. [Chen, Zaharia, and Zou (2023)](https://arxiv.org/abs/2305.05176)

RouteLLM learns a router from preference data to select between stronger and weaker models, reporting more than twofold cost reductions in some benchmarks without a quality loss. [Ong et al. (2024/2025)](https://arxiv.org/abs/2406.18665) A well-calibrated router creates selection value; an uncalibrated router adds another failure point and another latency stage.

Caching has a deeper role than a discount. When prefixes repeat, it avoids recreating KV state. Measure prefix locality, cache-hit rate, tokens avoided, and TTFT saved by workflow and tenant. Retrieval and deterministic tools are also factor substitution: a precise query or calculation may improve grounding while replacing a large amount of generative work. A poor retriever, however, can lower $q$ quickly. Route processing steps, not just models.

## 6. Agent tokenomics: coordination is a real cost

In a single-turn chat, tokens are mainly variable cost. In an agent system, tokens are also messages, state, delegation, and sometimes authority. More agents talking is not evidence of more work being done.

With $m$ agents, the number of possible communication channels grows as

$$
E_{\text{possible}}=\frac{m(m-1)}{2}.
$$

Not every topology uses every edge, but the formula captures the coordination tax: delegation, repeated context, debate, checking, repair, and handoff. The survey *Token Economics for LLM Agents* treats tokens as production factors, media of exchange, and units of account, connecting single-agent factor substitution with multi-agent transaction-cost and principal--agent problems. [Chen et al. (2026, survey)](https://arxiv.org/abs/2605.09104)

A worker can optimise a local objective—complete a subtask, provide more evidence, call more tools—while worsening the global one through duplicate work, delay, or context leakage. The unit of control should therefore be a responsible trace, not a chat transcript.

| Trace field | Question it answers |
| --- | --- |
| workflow, task class, tenant, policy version | Who incurred spend, and under which rule? |
| input, output, reasoning, and cached tokens when available | Where did tokens go? |
| model, route, prompt version, retrieval, and tools | Which decisions changed cost and quality? |
| TTFT, TPOT, end-to-end latency, retries | Where did time go? |
| verifier result, acceptance, human intervention | Did it create a successful outcome? |
| action, approval, reversibility, incident link | What downstream risk did it create? |

Without this trace, token optimisation becomes blind trimming: shorten the prompt and lose grounding, cap the answer and create rework, or switch models and damage a rare but expensive cohort.

## 7. Pricing is a screening mechanism, not only a meter

On the provider side, pricing resolves information asymmetry. Users know more than providers about their task mix, desired quality, and willingness to pay. *Menu Pricing of Large Language Models* models heterogeneous task valuations and derives committed-spend contracts in which buyers purchase a budget and allocate it across token classes priced at marginal cost. The paper relates the mechanism to observed token-budget menus, spend commitments, multi-model versioning, and linear API pricing. [Bergemann, Bonatti, and Smolin (2025/2026)](https://arxiv.org/abs/2502.07736)

This explains why the market can support different input/output rates, cached-input discounts, batch and priority lanes, subscriptions, credits, committed use, reasoning tiers, and open-weight alternatives. None is naturally more neutral. Each partitions surplus, transfers demand risk, and changes incentives for routing and caching.

Compare vendors using total cost of ownership for the workload distribution, not a headline price card: context length, cache semantics, SLOs, retries, availability, tool and egress costs, data controls, engineering cost, and switching cost all belong in the economic contract.

## 8. Energy is not an optional footnote

A token price hides a physical production system. A simple operational energy and emissions account is

$$
C_{\text{energy}}=\text{PUE}\cdot e\cdot p_e,
\qquad
E_{\text{CO}_2}=\text{PUE}\cdot e\cdot g,
$$

where $e$ is IT energy per accepted run, $p_e$ is electricity price, $g$ is grid carbon intensity, and PUE captures facility overhead. This is not a universal per-token formula: workload shape, hardware, batching, utilisation, cooling, and location matter. But it makes the missing variables explicit.

Luccioni, Jernite, and Strubell measure energy and carbon for 1,000 inferences across task-specific and general-purpose systems. They find that generative, multi-purpose architectures can be orders of magnitude more energy intensive than task-specific systems, even after controlling for parameter count. Task structure and output length matter materially. [*Power Hungry Processing* (2023)](https://arxiv.org/abs/2311.16863)

At system scale, the International Energy Agency estimates that data centres consumed about 415 TWh in 2024, around 1.5% of global electricity demand, and projects around 945 TWh by 2030 in its base case. It also notes that demand is geographically concentrated, so local grid integration can be difficult even when the global share appears modest. [IEA, *Energy and AI* (2025)](https://www.iea.org/reports/energy-and-ai/energy-demand-from-ai)

The tokenomic conclusion is not that every product should report a speculative carbon number. It is that energy, cooling, capacity, and location are production constraints. Long-context, low-value generation is not merely a higher invoice; at scale it consumes scarce infrastructure and creates externalities that the API meter may not price.

## 9. The production frontier and the operating model

Every configuration—model, quantisation, batch policy, router threshold, context policy, toolchain—can be represented as a point $(C,Q,L,R)$. Any configuration that is more expensive, lower quality, slower, and riskier than another is dominated. The remaining points form a production frontier.

A recent preprint builds an LLM inference production frontier from WiNEval-3.0 and reports diminishing marginal cost, diminishing returns to scale, and an optimal cost-effectiveness region. [Zhuang et al. (2025, preprint)](https://arxiv.org/abs/2510.26136) Another calls the tension among fine-grained valuation, real-time performance, and allocation optimality the Token Economics Trilemma. [Wu and Deng (2026, preprint)](https://arxiv.org/abs/2605.17410) The practical lesson is sharp: the cost of deciding how to spend tokens belongs inside the objective. A theoretically optimal router that needs the full prompt and half a second to score it may destroy the surplus it promises.

A disciplined operating loop looks like this:

1. **Write a utility contract.** Name the user, decision, upside, downside, useful latency, and human-review boundary.
2. **Segment demand before selecting a model.** Separate easy and hard, online and offline, long and short context, repeated and novel, low- and high-risk work.
3. **Build the smallest credible baseline.** A single model plus deterministic validation is usually a better baseline than a premature agent swarm.
4. **Measure CPAO and traces.** Capture quality, acceptance, end-to-end latency, cost, retries, policy decisions, and incidents.
5. **Change one lever at a time.** Evaluate context pruning, retrieval, cache policy, routing, verifier, and test-time budget separately against a stable holdout.
6. **Route by expected utility, with abstention.** “Escalate” is a valid action; model preference alone is not a cost-of-error policy.
7. **Set budgets by cohort and authority.** Research may deserve a large token budget but a small action budget. Exhaustion must be observable, not a silent truncation.
8. **Recompute the frontier.** Prices, models, cache locality, workload mix, and capacity change. Re-run the same quality--cost--latency--risk evaluation on consequential releases.

## 10. Open problems

1. **Hidden reasoning and actual compute.** API users can see billed tokens, but not necessarily internal routing, speculative work, or all inference compute. Cross-provider comparisons remain opaque.
2. **Marginal token productivity.** How much does token 1,000 increase success probability on a real task distribution? This needs counterfactual budget experiments, not correlation between length and score.
3. **Calibration for routing and stopping.** Model confidence is not a calibrated probability. Routers must be tested under distribution shift, model replacement, and rare high-loss cases.
4. **Value attribution in agent traces.** After twelve tool calls and four agents, which token, tool, or human checkpoint created the gain? Credit assignment determines whether optimisation helps or harms.
5. **SLO-aware economic scheduling.** Online allocation must balance tenant fairness, tail latency, energy, batch efficiency, and value density. Neither highest-value-first nor shortest-job-first is universally correct.
6. **Security and privacy externalities.** Context can carry personal data, proprietary information, prompt-injection payloads, stale policy, and secrets. Its cost is not linear in token price.
7. **Market design and interoperability.** Commitments, menus, open weights, prompt and trace portability, and switching costs will determine who captures surplus as inference commoditises.
8. **Energy- and location-aware allocation.** The same service can face different energy prices, grid intensity, cooling efficiency, and capacity constraints by time and location.
9. **Benchmark-to-business transfer.** A benchmark delta rarely identifies a delta in expected business value. Evaluations need acceptance, abstention, latency, and cost-of-error, not only exact match.

## The macroeconomic guardrail

A local token saving is not automatically an economic productivity gain. Acemoglu's task-based analysis makes the right warning explicit: if AI's microeconomic effects operate through task-level cost savings, aggregate productivity depends on the fraction of tasks actually affected and the magnitude of those verified savings. [Acemoglu, *The Simple Macroeconomics of AI* (2024)](https://www.nber.org/papers/w32487)

This is a useful guardrail for AI tokenomics. Do not multiply a lower API bill by every employee, every process, or the entire economy. First establish adoption, task coverage, quality-adjusted time saving, complementarity with human work, and the cost of the controls required to use the system safely.

## Conclusion: cheaper tokens do not remove the need to think

As tokens become cheaper, the question changes from “can we afford AI?” to “where should we buy another unit of thought?”

The answer is rarely “everywhere.” Buy compute where it changes the probability of a valuable outcome. Use systems design so that repeated context is not paid for twice. Use routing so routine work does not receive flagship treatment. Use evaluation to discover whether extra reasoning helps. Use governance so that a cheap request cannot turn into an expensive mistake.

That is the practical meaning of **Tokens for Thought**. Tokens are the meter. The work is to build a system that knows when to run the meter, when to stop it, and how to ensure that every additional tick creates more value than risk.

## Sources and further reading

### Token economics, pricing, and production frontiers

- [Bergemann, Bonatti, and Smolin — *Menu Pricing of Large Language Models*](https://arxiv.org/abs/2502.07736)
- [Zhu — *AI Tokenomics: The Economics of Tokens, Computation, and Pricing in Foundation Models*](https://arxiv.org/abs/2606.24616)
- [Epoch AI — *LLM inference prices have fallen rapidly but unequally across tasks*](https://epoch.ai/data-insights/llm-inference-price-trends)
- [Du — *Tiered Super-Moore's Law*](https://arxiv.org/abs/2603.28576) *(preprint)*
- [Zhuang et al. — *Beyond Benchmarks: The Economics of AI Inference*](https://arxiv.org/abs/2510.26136) *(preprint)*
- [Wu and Deng — *Computational Challenges in Token Economics*](https://arxiv.org/abs/2605.17410) *(preprint)*

### Scaling, routing, and test-time compute

- [Hoffmann et al. — *Training Compute-Optimal Large Language Models*](https://arxiv.org/abs/2203.15556)
- [Sardana et al. — *Beyond Chinchilla-Optimal*](https://arxiv.org/abs/2401.00448)
- [Snell et al. — *Scaling LLM Test-Time Compute Optimally*](https://arxiv.org/abs/2408.03314)
- [Lin et al. — *Plan and Budget*](https://arxiv.org/abs/2505.16122)
- [Chen, Zaharia, and Zou — *FrugalGPT*](https://arxiv.org/abs/2305.05176)
- [Ong et al. — *RouteLLM*](https://arxiv.org/abs/2406.18665)
- [Leviathan, Kalman, and Matias — *Fast Inference from Transformers via Speculative Decoding*](https://arxiv.org/abs/2211.17192)

### Serving systems, energy, and agents

- [Kwon et al. — *Efficient Memory Management for LLM Serving with PagedAttention*](https://arxiv.org/abs/2309.06180)
- [Zhong et al. — *DistServe* (OSDI 2024)](https://www.usenix.org/system/files/osdi24-zhong-yinmin.pdf)
- [Yu et al. — *ORCA* (OSDI 2022)](https://www.usenix.org/conference/osdi22/presentation/yu)
- [Erdil — *Inference economics of language models*](https://arxiv.org/abs/2506.04645)
- [Chen et al. — *Token Economics for LLM Agents*](https://arxiv.org/abs/2605.09104) *(survey/preprint)*
- [Luccioni, Jernite, and Strubell — *Power Hungry Processing*](https://arxiv.org/abs/2311.16863)
- [International Energy Agency — *Energy and AI*](https://www.iea.org/reports/energy-and-ai)
- [Acemoglu — *The Simple Macroeconomics of AI*](https://www.nber.org/papers/w32487)

Preprints are useful for framing hypotheses and research questions, but should not be the sole independent evidence behind an investment decision. Reproduce the relevant quality, latency, cost, and risk trade-offs on your own workload before treating a paper result as a deployment decision.
