---
title: "Speculative Decoding"
date: 2026-07-25T10:00:00+07:00
tags: ["ai", "llm", "inference", "systems", "machine learning"]
math: true
summary: "Speculative decoding makes autoregressive generation less serial without surrendering the target model's distribution. This is a technical guide to the exact algorithm, its variants, its limits, and the engineering decisions that decide whether it helps."
---

> *The draft model is allowed to be wrong. The target model is not allowed to be bypassed.*

Large language models are astonishingly parallel machines forced into a serial job. They can evaluate a whole prompt in one dense pass, yet an answer still emerges token by token: the token at position $t+1$ needs the token at $t$. For a latency-sensitive product, that dependency is the bill.

Speculative decoding changes the shape of the work. A cheap process predicts several likely future tokens. The expensive model scores those positions together, accepts the part that is consistent with its own distribution, and corrects the first part that is not. When the acceptance rule is implemented correctly, the final text is sampled from **exactly the same distribution** as ordinary target-model sampling. The speedup comes from reducing expensive *serial* target steps—not from trusting a weaker model.

This distinction is the whole idea. It is also where most shallow explanations go wrong.

The original modern formulation by Leviathan, Kalman, and Matias showed 2–3× acceleration on T5-XXL with unchanged outputs. [*Fast Inference from Transformers via Speculative Decoding* (ICML 2023)](https://proceedings.mlr.press/v202/leviathan23a.html) Concurrently, Xia et al. developed a draft-and-verify formulation for sequence-to-sequence generation, reporting strong task-specific gains. [*Speculative Decoding* (Findings of EMNLP 2023)](https://aclanthology.org/2023.findings-emnlp.257/)

Since then, “speculative decoding” has become a family of methods rather than one algorithm: small draft models, early exits from the target itself, multi-head predictors, retrieval and n-gram drafters, dynamic trees, long-context systems, and even diffusion-language-model adaptations. This article builds the shared mental model first, then explains what genuinely changes across the variants.

## The bottleneck: a model that waits for itself

For an autoregressive target model $p$, a continuation $x_{1:T}$ is generated as

$$
p(x_{1:T}\mid c)=\prod_{t=1}^{T}p(x_t\mid c,x_{<t}),
$$

where $c$ is the prompt. A normal decoder must run the target model once, commit $x_t$, then run it again for $x_{t+1}$. The model's matrix multiplications are parallel; the *chain of decisions* is not.

During decode, a large transformer is frequently limited by moving weights and KV-cache state through memory rather than by peak arithmetic. A target pass that scores several positions at once can therefore be much cheaper than the same number of separate target passes. This is the opening speculative decoding exploits.

There are two costs to keep separate:

- **Target passes on the critical path.** This is the scarce latency resource.
- **Everything required to make a proposal.** A draft model, retrieval lookup, extra heads, tree construction, cache management, and synchronization are not free.

Speculation wins only when it turns enough target passes into one verification pass to pay for the second cost.


## The exact algorithm, without hand-waving

Let $q$ be a cheap draft distribution and $p$ the target distribution. At a verified prefix $x_{<t}$, the drafter samples a block of $\gamma$ tokens:

$$
\tilde{x}_{t:t+\gamma-1}\sim q(\cdot\mid x_{<t}).
$$

The target then evaluates the proposed positions in one causal forward pass. For the $i$-th proposal, define

$$
p_i(v)=p(v\mid x_{<t},\tilde{x}_{t:t+i-1}), \qquad
q_i(v)=q(v\mid x_{<t},\tilde{x}_{t:t+i-1}).
$$

For sampled decoding, the proposal $\tilde{x}_{t+i}$ is accepted with probability

$$
\alpha_i=\min\left(1,\frac{p_i(\tilde{x}_{t+i})}{q_i(\tilde{x}_{t+i})}\right).
$$

Accept proposals from left to right until the first rejection. If the first rejection is at $i$, sample a correction token from the residual distribution

$$
r_i(v)=\frac{\max\bigl(0,p_i(v)-q_i(v)\bigr)}
{\sum_u\max\bigl(0,p_i(u)-q_i(u)\bigr)}.
$$

Then discard every draft token after that point and begin again. If all $\gamma$ proposals are accepted, sample one extra token from the target distribution and continue.

That residual correction is not a detail. It is why the method is exact. The draft distribution contributes the part of probability mass it already proposed; the correction samples exactly the target mass the draft underrepresented. At each position, accepted mass plus residual mass equals $p_i$. By induction over positions, the entire emitted sequence has the target model's distribution.

### Pseudocode

```text
prefix = prompt
while not stopped:
    proposal, q_probs = draft(prefix, gamma)
    p_probs = target_score(prefix + proposal)     # one causal target pass

    for i, token in enumerate(proposal):
        if uniform(0, 1) <= min(1, p_probs[i][token] / q_probs[i][token]):
            emit(token)
            prefix += token
        else:
            correction = sample( normalize(relu(p_probs[i] - q_probs[i])) )
            emit(correction)
            prefix += correction
            break
    else:
        extra = sample(p_probs[gamma])
        emit(extra)
        prefix += extra
```

Real implementations must apply the *same* temperature, top-$p$, top-$k$, repetition penalties, vocabulary mapping, stop conditions, and logits processors to the distributions used in the ratio. “We used the same models” is not enough. If $p$ and $q$ describe differently transformed distributions, the exactness argument no longer applies.

### Greedy decoding is simpler—and different

With greedy decoding, there is no acceptance coin. The target accepts a drafted token while it equals the target's argmax; at the first mismatch it emits the target argmax. This preserves the ordinary greedy output exactly, but it is not the same proof as stochastic speculative sampling.

There is another important boundary: many systems casually called “speculative” use a heuristic accept rule, a quality gate, or a jointly fine-tuned backbone. They may be excellent engineering, but they are not automatically distribution-preserving. **Lossless** means an output-distribution guarantee under stated decoding assumptions; it does not mean “the benchmark score looked unchanged.”

## Where the speedup comes from

Suppose a speculative cycle accepts $A$ drafted tokens and emits one additional target token when the whole block is accepted. Its useful progress is approximately $A+1$ tokens per target verification. Let $C_p$ be a standard target decode step, $C_v(\gamma)$ the target verification pass, and $C_q(\gamma)$ the drafting cost. A rough latency model is

$$
\text{speedup}\ \approx\ \frac{(\mathbb{E}[A]+1)C_p}
{C_v(\gamma)+C_q(\gamma)+C_{\text{overhead}}}.
$$

This equation explains nearly every practical surprise.

- If $q\approx p$, accepted prefixes are long—but an overly large drafter may erase the gain.
- If $\gamma$ is too small, verification leaves parallelism unused. If it is too large, later guesses are routinely discarded and the verification batch/tree becomes costly.
- A fast target kernel may make $C_v(\gamma)$ grow slowly; a quantized or compute-bound target can make a large verification tree expensive.
- Speedups measured in tokens per second at batch size 1 do not automatically become better end-to-end request latency or high-throughput serving.

The operational metric worth logging is not only acceptance rate. Log **mean accepted length**, draft latency, verification latency, target-forward count per emitted token, total wall-clock latency, and the request's context length. An impressive acceptance rate can coexist with a slowdown if the drafter or verifier is too expensive.

## A taxonomy: the verifier is stable; the drafter changes

The cleanest way to organize the literature is to ask: *where do the proposed futures come from?*

| Family | What proposes tokens? | Main advantage | Main cost or risk | Representative work |
| --- | --- | --- | --- | --- |
| **Independent drafter** | A small separately trained LM | Simple, exact, model-agnostic | Extra weights, KV cache, and draft latency | [Leviathan et al.](https://proceedings.mlr.press/v202/leviathan23a.html), [Xia et al.](https://aclanthology.org/2023.findings-emnlp.257/) |
| **Self-speculation** | Early layers of the target | Shared weights, activations, and cache | Requires usable early exits; may need training | [LayerSkip](https://aclanthology.org/2024.acl-long.681/) |
| **Auxiliary heads / features** | Heads or a lightweight feature predictor on the target | Avoids a full separate LM; good tree proposals | Training/serving integration | [Medusa](https://arxiv.org/abs/2401.10774), [EAGLE](https://arxiv.org/abs/2401.15077) |
| **Retrieval / copy** | Prompt, corpus, suffix index, or automaton | No drafter training; superb on repetition | Weak when the continuation is novel | [REST](https://aclanthology.org/2024.naacl-long.88/), [SAM Decoding](https://aclanthology.org/2025.acl-long.595/) |
| **Multi-draft / distributed** | Several drafters or devices | Can trade more parallel hardware for better proposals | Communication and scheduling complexity | [SpecHub](https://aclanthology.org/2024.emnlp-main.1148/), [Distributed Speculative Inference](https://proceedings.mlr.press/v262/timor24a.html) |

These are not mutually exclusive. A retrieval proposal can augment EAGLE; a long-context system can use a small local drafter; a tree can be populated by almost any drafter. The verifier and its acceptance semantics determine whether the combination remains exact.

## Linear blocks are not the only shape

A linear block asks one question: “what is the most likely next sequence?” That is wasteful when the first uncertain token has several plausible alternatives. Tree-based speculation spends the same verification opportunity on branches.

The drafter proposes a prefix tree rather than a single chain. Tree attention assigns each node only the ancestors on its own branch, allowing the target to score many candidates in one pass without letting sibling branches contaminate one another. The system then finds the longest accepted root-to-leaf path and emits its prefix.


[SpecInfer](https://arxiv.org/abs/2305.09781) popularized tree-based speculative serving; [Medusa](https://arxiv.org/abs/2401.10774) generates candidate trees through extra decoding heads; [EAGLE-2](https://aclanthology.org/2024.emnlp-main.422/) makes the tree context-adaptive using draft confidence. The paper's reported 5× figure is a result under its own models, tasks, hardware, and batch setting—not a portable multiplier. That caveat belongs beside every speculative-decoding number.

### Why dynamic trees matter

Static trees assume that “the third drafted token” is equally likely to be accepted in every context. It is not. After a predictable phrase such as a boilerplate closing, depth is valuable. At a high-entropy fork—an identifier, number, code symbol, or open-ended choice—breadth can be more valuable. EAGLE-2 uses the drafter's calibrated confidence to allocate tree nodes dynamically. This is a useful general principle: **speculation capacity should follow conditional predictability, not a fixed position index.**

## The major variants, and what they actually change

### 1. Independent draft models: the reference design

The original design pairs a small $q$ with a large $p$. It is portable: any target that can score a proposed continuation can participate, and the drafter can be distilled or specialized for the workload. The catch is systems cost. Two models need memory; two KV caches need lifecycle management; tokenizer differences may make token-level verification impossible or require special handling; and an inaccurate drafter creates shallow accepted prefixes.

This remains the best conceptual baseline. If a more elaborate method cannot beat a well-matched independent drafter under the same SLO and hardware, its complexity needs a reason to exist.

### 2. Self-speculation: draft from an earlier layer

[LayerSkip](https://aclanthology.org/2024.acl-long.681/) trains intermediate layers to be useful exits using layer dropout and an early-exit loss. An early layer drafts; the remaining layers verify and correct. The attractive property is reuse: draft and verification share parameters, activations, and KV state, lowering memory footprint versus a separate model.

The tradeoff is architectural. Ordinary intermediate-layer logits are rarely good enough by accident. The training recipe must make them predictive; exiting too early collapses acceptance, while exiting too late saves little compute. Self-speculation is therefore not “free acceleration from any checkpoint.”

### 3. Heads and feature-level drafting

[Medusa](https://arxiv.org/abs/2401.10774) adds multiple decoding heads to predict future positions in parallel and uses tree attention to verify their candidates. Its frozen-backbone variant is designed for lossless acceleration; its jointly trained variant pursues stronger proposals at the cost of a more specialized model.

[EAGLE](https://arxiv.org/abs/2401.15077) takes a different route: it drafts at the contextual-feature level, reusing target information before token prediction. [EAGLE-2](https://aclanthology.org/2024.emnlp-main.422/) adds dynamic draft trees. [EAGLE-3](https://arxiv.org/abs/2503.01840) replaces the feature-prediction constraint with direct token prediction and multi-layer feature fusion. These are powerful because the drafter sees rich target-model representations; they are less drop-in because training, checkpoint format, and serving runtime become coupled.

### 4. Retrieval, prompt lookup, and suffix structures

When text is repetitive, the fastest draft model may be no model at all. Prompt lookup and n-gram methods copy a matching continuation from the current context. [REST](https://aclanthology.org/2024.naacl-long.88/) retrieves from an external text database; [SAM Decoding](https://aclanthology.org/2025.acl-long.595/) uses a suffix automaton over the generated sequence and static corpus. These methods can be plug-and-play and exceptionally cheap.

Their failure mode is honest: a corpus cannot retrieve a future that has never occurred in a relevant form. They work best for repeated templates, code, copied context, and domains with regular phrases; they are not a substitute for semantic competence. Their strongest use is often as an extra candidate source alongside a model-based drafter.

### 5. Longer drafts and heterogeneous systems

Some work asks how to make a proposal deeper without multiplying serial draft cost. [Ouroboros](https://aclanthology.org/2024.emnlp-main.742/) builds longer drafts phrase by phrase. [TriForce](https://arxiv.org/abs/2404.11912) and [MagicDec](https://www.together.ai/blog/speculative-decoding-for-high-throughput-long-context-inference) focus on long-context serving, where a fast draft's attention strategy and cache behavior become central. Distributed systems such as [DSI](https://proceedings.mlr.press/v262/timor24a.html) move parts of the work across devices.

These methods illustrate a recurring truth: at production scale, speculative decoding is not only a sampling algorithm. It is a scheduling, memory, communication, and queueing problem.

### 6. Extensions beyond ordinary next-token sampling

Speculation can accelerate a different objective as long as there is a meaningful verifier. [Constrained Decoding with Speculative Lookaheads](https://aclanthology.org/2025.naacl-long.239/) combines a draft model with target and task-specific reward verification. [Fuzzy Speculative Decoding](https://aclanthology.org/2025.findings-acl.1346/) explicitly trades a controlled divergence/quality budget for runtime. [Speculative Diffusion Decoding](https://aclanthology.org/2025.naacl-long.601.pdf) adapts the idea to diffusion language models, where ordinary causal verification is unavailable.

These are useful extensions, but they should not inherit the word *lossless* by association. State the contract: exact target distribution, exact greedy output, bounded divergence, preserved task metric, or an empirical quality comparison. They are different guarantees.

## A worked intuition: acceptance is overlap, not accuracy

People often say “use a very accurate draft model.” The more precise statement is: use a drafter whose distribution overlaps the target where the target puts mass, at a cost low enough to matter.

For one position, the probability that a proposal is accepted under the exact rule is

$$
\sum_v q(v)\min\left(1,\frac{p(v)}{q(v)}\right)
=\sum_v\min(p(v),q(v))
=1-\operatorname{TV}(p,q),
$$

where $\operatorname{TV}$ is total variation distance. The acceptance probability is the overlap of the two distributions. It is not top-1 accuracy, and it is not perplexity alone.

This gives three practical lessons.

1. **Calibration matters.** A drafter that overcommits to the wrong token can have a poor overlap even if its argmax is often reasonable.
2. **The workload matters.** Boilerplate, syntax, and copied spans tend to have high overlap; creative continuation and unfamiliar multilingual text may not.
3. **Temperature matters.** Increasing sampling temperature alters the distributions and often lowers agreement. Benchmark the exact production decoding policy, not a convenient greedy proxy.

## When speculative decoding disappoints

Speculative decoding is not a free multiplier. Common failures are usually explainable.

| Symptom | Likely cause | Useful response |
| --- | --- | --- |
| High acceptance, little wall-clock gain | Draft or verification overhead dominates | Profile draft, verify, tree build, and synchronization separately. |
| Gains vanish at batch size | Target is already efficiently batched; scheduling changes the bottleneck | Evaluate under the actual arrival process and SLO, not only batch size 1. |
| Great on code, weak on chat | Repetition/copy structure differs | Use workload-segmented benchmarks and adaptive routing. |
| Incorrect stochastic outputs | Logits processors or tokenization differ between $p$ and $q$ | Apply transformations identically and test distributional equivalence on small cases. |
| Memory pressure | Separate drafter and target caches compete | Consider self-speculation, smaller drafts, offloading, or cache-aware scheduling. |
| A deep tree slows the target | Verification has become compute-bound | Cap nodes by measured target cost; do not optimize accepted tokens alone. |
| A reported “lossless” result changes quality | The method is heuristic or changes the backbone | Name the guarantee precisely and compare with a matched baseline. |

Long contexts deserve special care. The target's decode cost grows with its KV cache; a draft model that sees the full context may no longer be cheap. A windowed or compressed drafter can regain speed, but may reduce overlap. The optimum is a systems tradeoff, not a universal draft/target parameter ratio.

Quantization changes the picture too. When weight loading is the bottleneck, speculation can help by amortizing a target pass. When low-bit kernels make target verification compute-heavy, a giant tree can hurt. [QSpec](https://aclanthology.org/2025.emnlp-main.240/) and work on speculative decoding with quantization make the interaction explicit. Benchmark the combined stack; do not multiply isolated speedup claims.

## A production decision procedure

Treat speculative decoding as an experiment with a narrow hypothesis: *for this model, traffic mix, hardware, and decoding policy, verified blocks reduce the relevant latency or cost without weakening the output contract.*

1. **Define the invariant.** Is the requirement identical greedy output, exact sampling from the target, or an allowed quality–latency tradeoff? Write it down before choosing a method.
2. **Establish a target-only baseline.** Match model revision, quantization, prompt distribution, temperature, maximum length, stopping rules, concurrency, and hardware.
3. **Measure the right segments.** Split interactive versus batch, short versus long context, code versus prose, and deterministic versus sampled workloads. A single average hides the decision.
4. **Choose the simplest candidate source.** Repetition-heavy workload: start with prompt lookup or retrieval. Existing compatible draft checkpoint: try independent SD. Training/control of the target: evaluate heads or early exit.
5. **Tune $\gamma$ or tree budget jointly with the runtime.** Optimize end-to-end p50/p95 latency, TPOT, throughput, and cost per completed request—not acceptance rate in isolation.
6. **Validate correctness.** For greedy mode, compare sequences exactly. For sampling, use controlled distribution tests on small vocabularies or repeated fixed-seed experiments and audit every logits transformation.
7. **Instrument and guardrail.** Log accepted length, fallback rate, memory, target/draft time, queue delay, stop-reason correctness, and errors by traffic segment. Fall back to target-only decoding when the drafter is unhealthy or a request class regresses.

The last step is essential. A production decoder should be able to stop speculating. The safe fallback is not a failure; it is the reason the optimization can be deployed responsibly.

## A compact paper map

The literature is moving quickly, but the following landmarks explain the field's major turns.

| Year | Paper | Contribution |
| --- | --- | --- |
| 2018 | [Blockwise Parallel Decoding](https://arxiv.org/abs/1811.03115) | Early block verification precursor. |
| 2023 | [Leviathan et al.](https://proceedings.mlr.press/v202/leviathan23a.html) | Exact speculative sampling with an external drafter. |
| 2023 | [Xia et al.](https://aclanthology.org/2023.findings-emnlp.257/) | Formal draft-and-verify study for seq2seq generation. |
| 2024 | [Medusa](https://arxiv.org/abs/2401.10774) | Parallel decoding heads and tree attention. |
| 2024 | [EAGLE](https://arxiv.org/abs/2401.15077) / [EAGLE-2](https://aclanthology.org/2024.emnlp-main.422/) | Feature-level drafting, then context-aware dynamic trees. |
| 2024 | [LayerSkip](https://aclanthology.org/2024.acl-long.681/) | Early-exit self-speculation. |
| 2024 | [REST](https://aclanthology.org/2024.naacl-long.88/) | Retrieval as a training-free drafter. |
| 2025 | [SAM Decoding](https://aclanthology.org/2025.acl-long.595/) | Suffix automata for retrieval-based drafts. |
| 2025 | [EAGLE-3](https://arxiv.org/abs/2503.01840) | Direct token prediction with multi-layer feature fusion. |
| 2025 | [Speculative Decoding and Beyond](https://arxiv.org/abs/2502.19732) | Broad survey of generation–refinement methods. |

For a wider bibliography and a comparative taxonomy, start with [*Unlocking Efficiency in Large Language Model Inference*](https://arxiv.org/abs/2401.07851) and the 2025 survey above. The papers are best read as a design space, not a leaderboard: their reported speedups differ in target model, drafter, task, decoding policy, length distribution, precision, GPU, batch size, and serving stack.

## The deeper pattern

Speculative decoding is an instance of a more general systems idea: **separate cheap prediction from expensive authority, then verify enough to preserve the contract.** CPUs speculate down branches and recover on a miss. Databases use optimistic concurrency and validate before commit. Here, a small model, a retrieved suffix, or an early representation predicts a future; the target model decides what becomes real.

The method works because language is uneven. A strong model is sometimes needed for a hard semantic choice, but much of a continuation can be predictable once that choice is made. The best speculative systems find those easy stretches without pretending that every stretch is easy.

That is why the target model's final say matters. It turns a fallible guess into a safe optimization. And it is why the best question is not, “How many tokens can my drafter predict?” It is:

> **How much expensive serial work can this system eliminate while preserving the behavior the user is actually paying for?**
