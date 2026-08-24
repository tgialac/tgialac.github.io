---
title: "The Cheapest LLM Is Often the Most Expensive One"
date: 2026-08-23
lastmod: 2026-08-24
description: "Why production AI should optimise the path to an accepted outcome, not the price of the first model call."
summary: "A production essay on model routing, failure containment, tool governance, and the economics of quality, latency, and cost."
tags: ["LLM Gateway", "AI Infrastructure", "Model Routing", "Production AI"]
math: true
draft: false
---

A $0.001 model call can become a $0.10 support case.

The model may answer plausibly, miss a critical detail, emit an invalid tool call, trigger a retry, and eventually send the case to a human. The first inference was cheap. The path to an accepted outcome was not.

That is the mistake this article is about: pricing the first call instead of pricing the path.

The unit of optimisation is the cost of an accepted outcome: a result that is correct enough, fast enough, safe enough, and complete enough for the product to accept.

## The Cheap Call Is Not the Cheap Path

### The cost of being slightly wrong

Consider a support request about a delayed transfer. The user sees one sentence. The system may see an identity boundary, a data policy, a read-only tool, a latency deadline, a provider quota, and a possible escalation.

Suppose a small model receives the request because its list price is only $0.001. It produces a fluent answer, but the answer is not grounded in the current transaction state. The application calls a verifier, retries with a stronger model, queries the transaction service again, and sends the case to a reviewer.

The numbers below are illustrative, not a reported incident:

| Cost component | Illustrative cost |
| --- | ---: |
| Initial model call | $0.001 |
| Router and verifier overhead | $0.004 |
| Stronger-model retry | $0.015 |
| Extra tool and context steps | $0.030 |
| Human review or rework | $0.050 |
| **Total path to resolution** | **$0.100** |

The error is not that the small model was cheap. The error is treating the price of the first inference as the price of the task.

A more defensible economic model is:

<p class="concept-equation">\[\text{expected cost per accepted outcome} = \frac{\mathbb{E}[\text{total path cost}]}{\Pr(\text{accepted outcome before deadline})}\]</p>

The numerator is monetary. It includes model input and output tokens, reasoning tokens where applicable, router and verifier calls, retrieval, tools, databases, network and gateway infrastructure, retries, fallbacks, and human review. The denominator is the probability that the whole policy produces an accepted result before the deadline.

An observed cohort metric is simpler:

> Cost per accepted outcome = total dollars spent by the policy divided by the number of accepted outcomes.

<figure class="article-figure article-figure-plain">
  <img src="/images/illustrations/llm-gateway-value-chain.png" width="1580" height="773" loading="lazy" decoding="async" alt="A value chain from problem to input, solution, output, outcome, and value, with economy, efficiency, effectiveness, cost-efficiency, and cost-effectiveness marked at different stages.">
  <figcaption>The gateway is useful only when it connects spending on inputs to the effectiveness and value of the final outcome. This is an illustrative value-chain sketch, not a benchmark result.</figcaption>
</figure>

The expected formulation helps compare a new policy before it has processed much traffic. The cohort formulation is what finance and operations can reconcile after deployment. They answer different questions and should not be casually mixed.

**A worked cohort calculation.** Imagine 100,000 requests from the same workload. An always-cheap policy spends $100 on its first calls, but 18,000 requests need an additional $0.024 of repair, retry, or tool work and 5,000 reach a reviewer at an average marginal cost of $0.050. Its illustrative total is therefore $782 for 82,000 accepted outcomes, or about $0.0095 per accepted outcome.

An always-strong policy might spend $1,500 on its first calls, accept 96,000 outcomes, and land at about $0.0156 per accepted outcome. A deterministic router might spend $70 on cheap first calls, $375 on stronger routes, $100 on routing and verification, and $200 on recovery, while accepting 94,000 outcomes. That is $745 in total, or about $0.0079 per accepted outcome.

The router wins in this deliberately simplified example even though it uses more infrastructure than the always-cheap policy. The result depends on the workload mix, the quality threshold, the price schedule, and the cost assigned to review. Change any of those assumptions and the winner can change. That is precisely why a router should be measured against baselines rather than advertised with a universal savings percentage.

The accounting boundary also matters. A model vendor may report token spend; the product owner may need to account for retrieval, API calls, gateway compute, reviewer time, and the cost of a second customer contact. If those costs live in separate systems, the route trace needs a shared request identifier so finance can reconcile them after the fact.

This is why a more capable model can be cheaper in practice. It may cost more per call but reach the quality bar in one attempt. Conversely, a cheap model can be the expensive choice when it creates enough uncertainty downstream.

OpenAI makes a similar outcome-based argument in its [guidance on enterprise AI investment](https://openai.com/index/managing-ai-investments-in-agentic-era/): the meaningful unit is not isolated token spend, but the cost of reaching a useful result, including completion, latency, tool use, retries, and human involvement. That is enterprise guidance from one vendor, not an independent industry law, but it gives the right accounting direction.

### Production is a routing graph

Production traffic does not resemble a leaderboard where one model wins every task. In its July 2026 [AI Gateway Production Index](https://vercel.com/blog/ai-gateway-production-index-july-2026), Vercel reports anonymised aggregate traffic from more than 200,000 teams and regular use of many models across its highest-volume workloads. The data describes Vercel's gateway population, not the whole industry, but it is a useful view of what multi-model production looks like.

It looks like a routing graph. A fast model may classify intent or extract fields. A stronger model may handle ambiguity. A specialist may process images, embeddings, documents, or tool arguments. A verifier may check grounding or schema. A human may be the safest route when evidence is incomplete.

The right question is therefore not “Which model is best?” It is:

> Which admissible path gives this request an acceptable result under its quality floor, latency budget, data boundary, risk class, and cost ceiling?

Vercel also reports that a small percentage of requests completed only after a fallback. The rescued requests represented a somewhat different share of tokens and market cost. That is better described as a fallback rescue rate, with cost-weighted analysis, not “cost-weighted uptime”. Request availability and the cost of rescuing expensive requests are separate metrics.

The distinction matters because the workload has a long tail. A short FAQ can be retried by the user. A long-running agent may have already paid for a large context, several tool calls, and a long response when its provider connection fails. The expensive end of the workload deserves a different reliability policy from the cheap end.

An AWS demonstration of [resilience patterns with Bedrock and an LLM gateway](https://aws.amazon.com/blogs/machine-learning/implementing-resilience-patterns-with-amazon-bedrock-and-llm-gateway/) makes the idea concrete. In the documented configuration, a primary route has a restrictive quota and a fallback has more capacity. Ten concurrent requests are distributed so that all can complete without application-level retry logic. This is a controlled demonstration, not a production incident and not a universal availability guarantee. Its architectural lesson is that the client does not need to understand every provider quota. A gateway can own the next admissible path.

But “try another provider” is not a complete fallback policy. A read-only question may move to an approved secondary deployment. A request with restricted data may have only one legal route. A structured tool call may require compatible schema semantics. A state-changing action may need to stop rather than replay. A high-risk support case may need a stronger model and a human checkpoint.

<mark>The first call is an event. The path to an accepted outcome is the product.</mark>

## The Model Is Not the Route

### Model, provider, deployment, route

The word “model” hides several different decisions.

| Thing | What it means | Why the gateway cares |
| --- | --- | --- |
| Model | A learned capability: generation, reasoning, vision, embedding, or classification | Establishes what the request may be able to do |
| Provider | The organisation or platform serving that capability | Establishes API semantics, quotas, data terms, regions, and failure modes |
| Deployment | A concrete endpoint, account, region, or self-hosted replica | Establishes capacity, network boundary, version, and operational health |
| Route | The complete decision for this request | Combines capability, policy, budget, health, and recovery |

The same model family can behave differently across deployments. A provider can expose different quotas by region. A self-hosted endpoint can offer stronger data locality and weaker availability. A managed endpoint can offer operational simplicity while imposing a stricter context limit or a higher price.

So a route cannot safely be represented by a model name alone. It is a decision about capability, data zone, quality floor, latency budget, cost ceiling, availability, and recovery. The model registry should contain immutable model and deployment versions, capability declarations, context limits, structured-output behaviour, tool support, approved data zones, price schedules, health, and evaluation results.

A catalog is not a proxy table. It is the vocabulary with which the router can reason.

The first router operation is not scoring. It is filtering.

<mark>The cheapest model is irrelevant if it is not an admissible model.</mark>

### The control plane behind the proxy

There is an important terminology correction here. The AI control plane defines policies, registries, quotas, evaluation results, and rollout state. The request-serving gateway is the runtime enforcement boundary of that control plane. It executes on the data plane under a versioned control-plane configuration.

That distinction matters when the control plane is unavailable. A data-plane gateway may be able to continue serving low-risk traffic from a signed, bounded policy snapshot. It should not silently invent a new policy because the registry is down. Control-plane propagation, stale configuration, and request-level availability have different failure semantics.

The data plane for one request should be understandable: authenticate the caller, establish identity and data labels, validate the request, filter candidates, score the admissible routes, enforce deadline and budget, call a provider adapter, apply bounded recovery, validate the result, and emit a trace.

The asynchronous control plane should do different work: evaluate candidate models, analyse sampled outcomes, update prices and health, promote or roll back policy versions, and compare route performance. Offline evaluation is not a normal synchronous stage in every request path. It is a control-plane activity that changes what the data plane is allowed to do later.

<figure class="article-figure article-figure-plain">
  <img src="/images/illustrations/ai-gateway-control-boundary.png" width="1359" height="626" loading="lazy" decoding="async" alt="Applications and a user enter an AI Gateway whose safety, scalability, governance, FinOps, and observability capabilities connect to multiple model providers.">
  <figcaption>A gateway is a shared boundary for safety, scalability, governance, FinOps, and observability. The provider logos are illustrative; the architectural boundary is the point.</figcaption>
</figure>

Uber's [GenAI Gateway](https://www.uber.com/co/en/blog/genai-gateway/) is a useful real-world example of this platform boundary. Uber reports that its gateway unified access across OpenAI, Vertex AI, and internally hosted models, while centralising authentication, authorisation, PII redaction, metrics, audit logs, cost attribution, and security review. The published snapshot describes a platform used by many teams and millions of queries per month; those are Uber-reported figures, not a universal reference architecture.

Uber's later work on [agent identity](https://www.uber.com/gb/en/blog/solving-the-agent-identity-crisis/) adds a related insight: an agent request needs an attributable actor chain and short-lived, scoped credentials. A gateway can preserve that chain, but it should not pretend that a model-generated name or tool argument is an identity assertion.

The application-facing contract can remain boring and OpenAI-compatible. That is valuable because it reduces integration friction. Compatibility at the edge does not mean provider behaviour is identical behind the edge. Adapters still need to translate authentication, streaming, tool semantics, error classes, context limits, and structured-output guarantees.

A provider may support structured output without guaranteeing that every response is valid for every schema. Reliability depends on constrained decoding, API semantics, schema complexity, model version, validation, and repair behaviour. The gateway must validate the actual response.

**One gateway can hide two different routing problems.** The outer gateway chooses among logical routes: a model, provider, deployment, data zone, policy, budget, and fallback contract. The inner inference gateway chooses the concrete serving endpoint that should handle the request right now. These decisions should not be collapsed into one generic “model router”.

The [Kubernetes Gateway API Inference Extension](https://gateway-api-inference-extension.sigs.k8s.io/) makes the inner problem explicit. Its Inference Gateway couples a proxy with an Endpoint Picker, and its InferencePool represents a group of inference-serving endpoints. The picker can use model-serving metrics and capabilities such as queue depth, KV-cache status, prefix locality, or available LoRA adapters. It also supports model-aware routing, serving priority, and incremental model rollouts. This is a Kubernetes ecosystem specification and implementation pattern, not a requirement that every multi-provider gateway run on Kubernetes.

| Routing layer | Decision | Signals it can see |
| --- | --- | --- |
| Policy and provider gateway | Which model/provider/deployment is admissible for the request | Data policy, identity, capability, quota, price, provider health, historical latency |
| Inference gateway | Which replica or inference worker should serve it now | Queue depth, KV-cache utilisation, prefix match, accelerator capacity, adapter availability |

The distinction changes the architecture. A cross-provider gateway usually cannot observe the KV-cache state of a provider-managed endpoint. A self-hosted inference gateway can, but it may not know the organisation’s tenant policy or provider contract. A robust system composes the layers: the outer router produces an eligible pool, and the inner scheduler picks an endpoint within that pool. Neither layer should silently override the other’s boundary.

**Scheduling is not ordinary load balancing.** LLM serving has at least two user-visible latency components: time to first token and time to produce the remaining tokens. Prompt length, expected output length, queue depth, shared-prefix reuse, and prefill/decode contention all affect them. A round-robin balancer can distribute requests evenly while making p95 latency worse because it sends a long prompt to a busy worker with a cold cache.

The scheduler should therefore separate admission from endpoint selection. Admission checks whether the request can meet its deadline and budget at all. Endpoint selection then ranks eligible workers using the signals available in that serving layer. Interactive traffic may receive priority over batch summarisation; a tenant quota may cap concurrent work; a long-context request may be routed to a larger-context pool; and a request with a reusable system prompt may prefer the worker with a matching prefix. Every priority rule needs a fairness story, otherwise a small number of expensive requests can monopolise the fleet.

The [GKE Inference Gateway documentation](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/about-gke-inference-gateway) describes load-aware routing using KV-cache utilisation and pending queue depth, while newer predicted-latency routing uses request features and live serving signals to estimate TTFT and TPOT. Those are model-server signals. The article’s broader claim is narrower: if the platform owns the serving layer, it can make better endpoint decisions than an external gateway that sees only HTTP status codes and coarse provider health.

**Observability must be a contract, not a pile of logs.** The root trace should connect the application request to the policy decision, provider attempt, inference endpoint, tool call, and final outcome. At minimum, the gateway should record a stable request ID, policy and price versions, logical route, provider and deployment, queue delay, time to first token, time to last token, input and output token counts, retries, fallback transitions, tool latency, and accepted-outcome status.

The [OpenTelemetry GenAI observability work](https://opentelemetry.io/blog/2026/genai-observability/) is useful here because it standardises a vocabulary for model identity and token usage, including input and output token attributes. Standard names make it possible to compare providers and export traces to different backends. They do not make it safe to log everything. Prompt content, completion content, tool arguments, and tool results may contain secrets or personal data; content capture should be opt-in, redacted, sampled, and access-controlled, with a separate retention policy for compliance records.

The metric model should distinguish queue time, gateway time, provider time, and downstream tool time. Otherwise a p95 regression can be blamed on the model when the actual cause is admission control or a slow policy service. Cost should be derived from the same trace using effective-dated price versions, not from a dashboard that happens to know the current list price.

**The route contract.** A production gateway should be able to serialise the decision it made for one request. Not every internal feature needs to be exposed to the application, but the contract must be rich enough for an operator to answer five questions: who asked, what was allowed, which candidates were rejected, why this route won, and what happened when the route failed.

| Contract field | What it records |
| --- | --- |
| Request context | Tenant class, actor class, task class, deadline, data zone, and whether the request can mutate state |
| Policy decision | Policy version, model-subset version, allowed tools, budget reservation, and degraded-mode status |
| Candidate reasoning | Eligible routes, rejected reasons, health snapshot, and the quality or capability floor |
| Execution ledger | Provider, deployment, model version, attempts, stream state, tool calls, and request correlation ID |
| Outcome ledger | Validation, groundedness, policy result, accepted or rejected status, reviewer action, and final state |

This is not a request to expose hidden chain-of-thought. A useful trace can record decision features, policy outcomes, tool arguments after redaction, and route transitions without storing private reasoning traces. The goal is operational explainability: enough evidence to reproduce the decision and diagnose a failure.

The contract should be versioned like an API. Adding a new model should not silently change the meaning of an old trace. A model alias should resolve to an immutable version in the ledger. A policy rollout should carry a configuration digest. When a new routing policy looks better in a dashboard, an engineer should be able to compare it with the exact previous policy rather than with whatever happens to be deployed today.

The same record also creates a natural boundary between debugging and compliance. A redacted, short-retention trace can help an on-call engineer understand latency. A tamper-resistant audit record can prove which identity and policy authorised a mutation. They may share a correlation ID without sharing the same payload, retention period, or access permissions.

### Model calls and tool calls have different blast radii

A model call can produce a bad answer. A tool call can change state.

That difference should be visible in the architecture. Retrieved documents, tool outputs, and model-produced arguments are untrusted input. The model may propose an action. Only the tool gateway, using independently authenticated identity and server-side policy, may authorise or execute it. Secrets should not enter model context. Tools should receive least-privilege scoped credentials. Mutations should validate current server-side state and carry an idempotency key.

The gateway can carry policy and identity to a tool. It is not automatically the transaction coordinator. The downstream business service remains responsible for atomic state transitions, duplicate suppression, and commit semantics.

Telemetry is part of this trust boundary too. A useful route trace may contain tenant, actor, task, candidates, rejected reasons, tool names, arguments, model output metadata, cost, latency, policy version, and outcome. That trace can be more sensitive than the original prompt. Pseudonymous identifiers, redaction, retention periods, role-based access, encryption, and tamper-resistant audit storage are not optional decorations.

Identity should be more than a user ID copied into a prompt. [RFC 8693](https://www.rfc-editor.org/rfc/rfc8693.html) distinguishes delegation from impersonation and gives token exchange a place to represent both the subject and the acting party. A gateway can use that distinction to carry an actor chain to the tool service: the user or service on whose behalf the action is requested, the agent or application acting for them, the target resource, the requested scope, and the expiry. The authorization server still decides whether the delegation is valid; the model never does.

This is also why prompt-injection detection cannot be the only security control. The [OWASP Top 10 for LLM Applications](https://genai.owasp.org/resource/owasp-top-10-for-llm-applications-2025/) treats excessive agency as a combination of excessive functionality, excessive permissions, and excessive autonomy. The practical response is architectural: expose the smallest tool surface, validate arguments at the tool boundary, use least-privilege credentials, separate read from write capabilities, require current-state checks for mutations, and make sensitive actions idempotent and reviewable. A model can be manipulated even when the prompt classifier looks healthy; the tool gateway must remain safe under that assumption.

The invariant is simple:

> No model call or tool call bypasses the broker.

That is stronger than having a policy document. It is a structural property that can be tested.

<figure class="article-figure article-figure-plain">
  <img src="/images/illustrations/gateway-broker-mediation.png" width="1580" height="689" loading="lazy" decoding="async" alt="Autonomy policy, identity and attested intent, data labels and memory rules, and context trust converge on a broker that mediates whether an action is allowed, reviewed, or blocked.">
  <figcaption>The broker is the invariant: autonomy, identity, data labels, and context trust must converge before either a model call or a tool call is allowed to proceed.</figcaption>
</figure>

## Filter First, Then Optimise

### Hard constraints before soft preferences

Once model, provider, deployment, and route are separated, routing becomes a constrained decision problem.

For a request x, let C(x) be the candidate routes, and let A(x) be the subset that passes hard constraints. The router should choose from A(x), not from the full catalog:

<p class="concept-equation">\[A(x)=\{r\in C(x):\text{policy}(r,x)\land\text{capability}(r,x)\land\text{budget}(r,x)\land\text{health}(r)\}\]</p>

Only then should a preference function rank quality, latency, cost, and reliability:

<p class="concept-equation">\[r^*=\arg\min_{r\in A(x)}\;\text{expected path cost}(r,x)\]</p>

The exact objective can be more elaborate. The ordering cannot be reversed.

| Constraint or preference | Example | Failure if handled incorrectly |
| --- | --- | --- |
| Data policy | Restricted content may use only approved deployments | Data crosses an unauthorised boundary |
| Capability | Long context, vision, tool schema, structured output | The request cannot be completed reliably |
| Identity and risk | Mutation requires an authenticated actor and approval | The model is allowed to manufacture authority |
| Deadline and budget | Read-only answer must meet a p95 target | The route is technically correct but unusable |
| Health and quota | Provider is throttled or deployment is saturated | Retry storms and cascading failure |
| Preference | Among admissible routes, favour quality, latency, or cost | A legitimate trade-off is made explicit |

Policy defines the admissible set. Optimisation ranks what remains. A score may choose between acceptable routes; it must never make an unacceptable route attractive.

Microsoft's Model Router is a useful managed example of this separation. It offers Balanced, Cost, and Quality modes, but also lets an organisation constrain the model subset. Microsoft recommends observing which model was selected, retaining direct deployments for specialised or compliance-mandated workloads, and maintaining more than one eligible model for failover. These are product-specific controls, but they reinforce the general pattern: some requests should be optimised, and some should be pinned.

### The router is part of the route

Routing itself has cost and latency. A classifier, preference model, verifier, policy service, or extra embedding call may improve the final outcome while making the path slower or more expensive. The router is not free because it lives inside the gateway.

Research such as [RouteLLM](https://arxiv.org/abs/2406.18665) and [FrugalGPT](https://arxiv.org/abs/2305.05176) explores learned routing, preference-based selection, and cascades that use a cheaper model before escalating. Their reported savings are results under particular benchmark distributions, model pairs, labels, and calibration assumptions. They are important techniques, not production guarantees.

The same rule applies to managed routing claims. AWS reports that [Intelligent Prompt Routing](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-routing.html) can improve cost or quality in its supported configuration, while documenting limitations around language optimisation and the absence of application-specific performance feedback. The number should be treated as a vendor-reported result for a particular setup, not as a default expectation.

A router should start with rules because rules are inspectable. A deterministic classifier can send tool-using requests only to tool-capable deployments. A policy table can constrain a data zone. A simple cascade can escalate when a schema validator fails or a confidence threshold is not met. A learned router becomes worthwhile when it can beat those baselines on the real workload, including its own overhead.

Google's [GKE Inference Gateway case study](https://cloud.google.com/blog/products/containers-kubernetes/how-gke-inference-gateway-improved-latency-for-vertex-ai) demonstrates a different level of routing. Its gateway can observe queue pressure, model-server capacity, and prefix or KV-cache locality because it controls the inference-serving layer. Google reports improvements in time to first token and cache hit rate for specific workloads. An external multi-provider gateway usually cannot see those internal signals. It should not claim prefix-aware optimisation unless it owns or receives equivalent serving telemetry.

A production router therefore needs a confidence story. When the route decision is uncertain, the system can ask for clarification, choose a safer model, restrict tools, or escalate. It should not hide uncertainty inside a weighted score that no operator can explain.

<figure class="article-figure article-figure-plain">
  <img src="/images/illustrations/query-router-cost-quality-frontier.png" width="2048" height="874" loading="lazy" decoding="async" alt="A query router chooses among several language models, with an ideal router shown near the cost-quality frontier.">
  <figcaption>An illustrative cost–quality frontier: the router’s job is to approach the best admissible trade-off, not to pretend that one model dominates every workload. The model labels are illustrative, not current benchmark data.</figcaption>
</figure>

## Failure Is Part of the Route

### Retry budgets and admission control

Retries are useful only when the next attempt has a reasonable chance of changing the outcome.

| Failure | Safe response | Unsafe response |
| --- | --- | --- |
| Transient network error before provider acceptance | Retry within a shared budget, possibly on another eligible deployment | Retry independently at every application and agent layer |
| Rate limit or quota exhaustion | Select a healthy eligible route or shed load | Hammer the same provider until the queue collapses |
| Timeout before any streamed output | Retry if the deadline and request semantics allow it | Start an unbounded cascade |
| Malformed structured output | Validate, repair once if safe, then escalate | Treat provider support as a guarantee |
| Context overflow | Compress, retrieve less, or select a larger-context route | Repeat the same request unchanged |
| Tool timeout with unknown commit state | Reconcile by idempotency key before deciding | Blindly replay a mutation |
| Connection loss after streaming begins | Mark the response incomplete and surface a clear state | Transparently splice a second answer into the first |

<figure class="article-figure article-figure-plain">
  <img src="/images/illustrations/idempotency-retry-trap.png" width="1076" height="850" loading="lazy" decoding="async" alt="A retry after a payment timeout creates a double charge, while an idempotency key lets the server return the saved result and keep a single charge.">
  <figcaption>A timeout does not tell the client whether a mutation happened. An idempotency key turns a blind replay into a reconciliation check.</figcaption>
</figure>

The retry budget should be shared across the application, SDK, gateway, provider adapter, and agent loop. Otherwise five layers can each believe they are making one reasonable retry while the user experiences thirty attempts.

Admission control is part of reliability. If the deadline is already impossible, rejecting, degrading, or queueing a request can be better than accepting work that will consume capacity and still fail. Circuit breakers should be keyed to error class and route, not just to a single global percentage. A context overflow is not the same failure as provider throttling. A malformed tool argument is not repaired by adding more concurrency.

The gateway also needs a degraded-mode matrix. A signed policy snapshot may allow a low-risk, read-only request to continue when the control plane is temporarily unavailable. Sensitive data may be restricted to an already approved local route. External mutations should fail closed, with no replay. The exact policy is product-specific, but the decision must be explicit rather than hidden in a generic “fail open” switch.

### Fallbacks that preserve contracts

A fallback is not invisible. It may change the model, context limit, tool semantics, data residency, refusal behaviour, response quality, or price. Therefore it is not merely a recovery mechanism; it is another product path.

The fallback contract should say what is preserved and what may change. For an informational answer, the gateway may preserve the response schema and latency deadline while changing the provider. For a tool call, it may preserve the tool name and argument schema but require a different adapter. For a state-changing request, it may preserve only the transaction identifier and move the case to review rather than execute a second attempt.

Streaming makes transparent fallback especially dangerous. Once partial tokens have reached the client, a second provider cannot safely continue the same answer without risking duplication or contradiction. A connection loss after provider acceptance can also leave the commit state unknown. The gateway should use correlation IDs, mark partial responses explicitly, and let the downstream service reconcile mutations by idempotency key.

The gateway should not promise atomicity that only the business service can provide.

The strongest failure policy is often a graceful stop: preserve the state, explain what is known, and expose the next safe action. Availability is not the same as pretending that an incomplete operation succeeded.

Failure injection should be part of the benchmark rather than a last-minute chaos exercise. The gateway needs to distinguish a provider that rejected a request from one that accepted it and then disappeared, because the recovery contract is different. It also needs to distinguish a model error from a policy-service error, and a tool timeout from an unknown transaction state.

| Injected condition | Expected decision | What the trace must prove |
| --- | --- | --- |
| Provider throttling before acceptance | Choose another eligible route or shed load | No unbounded retry loop and no policy bypass |
| Timeout before first token | Retry only within the deadline and shared budget | The second attempt had a different admissibility or health reason |
| Connection loss after streaming begins | Mark the response incomplete | The gateway did not splice two answers together |
| Malformed tool arguments | Reject or repair once, then escalate | The tool never received unauthorised arguments |
| Mutation timeout after provider acceptance | Reconcile by idempotency key | No blind replay and no false success |
| Stale or unavailable policy snapshot | Use only the approved degraded path | The policy version and freshness were recorded |

The canary should measure more than error rate. A new policy may reduce provider errors while increasing false-cheap decisions, or lower token spend while increasing human review. Compare the route distribution, accepted outcomes, p95 latency, fallback cost, policy violations, and reviewer rework against the previous policy. Roll back on a hard safety regression immediately; do not wait for a weighted score to average it away.

## Evaluation Gives the Router a Memory

### Accepted outcomes, not fluent answers

A router cannot improve what it cannot observe. But observing final text is not enough.

An accepted outcome is a product-level contract. For a support response it may mean that the answer is grounded in the current record, uses an approved data path, follows the response schema, meets the latency target, and resolves the user’s question. For a coding agent it may mean tests pass, changes are reviewable, and no forbidden file or secret was touched. For a document workflow it may mean the extracted fields are correct enough for downstream processing and uncertainty is escalated when required.

Safety gates are not soft quality features. A policy violation should be a hard failure, not something a cheaper route can compensate for. Quality, cost, and latency can be compared among admissible routes. An unauthorised tool call cannot earn its way back into the accepted set by being inexpensive.

The evaluation stack should combine code-based checks, model-based graders, and human labels. Code can validate schemas, tool arguments, state transitions, policy versions, latency, token accounting, and cost. A model grader can help assess groundedness or helpfulness, but it needs calibration against expert labels. Human review remains important for high-consequence slices and for measuring grader drift.

The benchmark should be stratified by traffic slice: short and long context, simple and ambiguous intent, read-only and state-changing tools, normal and adversarial input, cold and warm cache, healthy and degraded provider. Aggregate quality can hide a dangerous regression in one slice.

Two tempting metrics need special care. A “false-cheap” decision means the router selected a cheap route that failed when a stronger route would have produced an accepted outcome. A “false-expensive” decision means the router paid for a stronger route when a cheaper route would also have succeeded. Neither is directly observable from one production request because the counterfactual route was not run.

They require paired offline runs, safe shadow execution, stratified replay, or reviewed labels. Side-effecting tools must not be freely replayed. Use read-only replay, sandboxed tools, deterministic transaction simulation, or production shadowing with tools disabled. Report uncertainty and confidence intervals rather than turning counterfactual estimates into exact online counters.

### From offline benchmark to canary

The route trace is the router’s memory. One useful trace records:

| Field | Example |
| --- | --- |
| Request context | tenant class, actor class, task class, deadline, data zone |
| Candidate set | eligible routes and rejected reasons |
| Decision | policy version, selected route, score inputs, fallback budget |
| Execution | provider, deployment, attempts, tool calls, stream state |
| Outcome | validation, groundedness, policy result, human review, accepted or rejected |
| Economics | input/output tokens, cached tokens, tool cost, gateway cost, total path cost |

The trace must be privacy-aware. Debugging payloads should be redacted and access-controlled; compliance records may need stronger retention and immutability. A trace is useful only if it can explain a decision without becoming a second uncontrolled data leak.

The evaluation loop should move through increasing levels of exposure. Begin with offline benchmark runs against an always-cheap baseline, an always-strong baseline, and a deterministic router. Add a learned router only after it beats the deterministic baseline on accepted outcomes, cost, latency, and safety slices. Then run read-only shadow evaluation. Next, canary a new policy on a bounded slice. Finally, promote it only when rollback thresholds are explicit.

Anthropic's [guidance on agent evaluations](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) emphasises capability and regression evals, transcript-level checks, and multiple grader types. Microsoft's model-router evaluation guidance similarly recommends representative prompts, pairwise comparisons, and latency and cost measurements across repeated runs. These sources support a method, not a universal benchmark threshold.

The details matter. Microsoft's [model-router evaluation protocol](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/model-router) recommends using at least 100 prompts from the actual workload, swapping the order of pairwise responses to reduce position bias, scoring accuracy and completeness independently, and comparing p50, p90, and p95 latency rather than relying on an average. It also asks teams to include router markup and the underlying model price in the cost calculation. Those recommendations are specific to Microsoft's toolkit, but they are a good minimum discipline for any router benchmark.

Anthropic's work adds a different warning: the final sentence of an agent transcript is not necessarily the outcome. A flight-booking agent can say that a booking succeeded while the environment contains no reservation. Their evaluation model separates a task, multiple trials, graders, the full transcript, and the final environment state. That separation is essential for a gateway with tools. The response text can be fluent while the database state, authorization path, or tool argument is wrong.

The [tau-bench research benchmark](https://arxiv.org/abs/2406.12045) makes this concrete by evaluating tool-agent-user interaction against an annotated database goal state and by introducing pass-to-the-k reliability across repeated trials. Its reported results found that even strong function-calling agents succeeded on fewer than half of the tasks in some settings, with reliability falling sharply across repeated attempts. The number is a benchmark result from retail and airline-style environments, not a prediction for every production system. The lesson is more durable: one successful transcript is weak evidence for a state-changing workflow.

For the gateway project, I would turn this into a release protocol. Split the task set by intent, consequence, context length, and provider condition. Keep a private test slice so the router is not tuned against every example it will later be judged on. Run several trials per task because stochastic output makes one pass/fail result noisy. Use deterministic graders for schema, policy, tool parameters, state transitions, latency, and cost. Use model graders only for dimensions that genuinely require semantic judgement, and calibrate them against a reviewed sample. Read failed transcripts before changing the router; otherwise a bad grader can teach the system to optimise the wrong behaviour.

The release gate should be conjunctive rather than a single weighted score. A policy violation, unauthorised mutation, or incorrect final state is a hard failure. Among policies that clear those gates, compare accepted-outcome rate, cost per accepted outcome, p95 latency, fallback rate, and route-distribution drift. This preserves the central asymmetry: quality and cost are trade-offs only after safety and contract compliance have passed.

Real financial-services deployments reinforce the need for domain-owned evaluation. OpenAI's [Morgan Stanley case study](https://openai.com/index/morgan-stanley/) describes pre-deployment evaluation and ongoing testing for a high-stakes financial workflow. OpenAI's [Balyasny case study](https://openai.com/index/balyasny-asset-management/) describes task-level model selection using internal benchmarks and proprietary data. Both are customer stories from one vendor, not independent proof, but they show why a platform needs empirical task-level evidence rather than a single global model ranking.

The core dashboard should stay small:

| Metric | Why it matters |
| --- | --- |
| Accepted outcome rate | Whether the policy clears the product contract |
| Cost per accepted outcome | Whether the policy creates economic value |
| p95 latency | Whether users receive the result in time |
| Policy violation rate | Whether hard boundaries remain hard |

Track fallback, retry, tool correctness, and route-distribution drift alongside them. Do not collapse everything into one weighted score. A single score is useful for a chart and dangerous as a release gate.

## A High-Stakes Router Starts With a Contract

### Intent, identity, and consequence

A high-stakes router does not start with a model. It starts with a contract.

Before selecting a model, the system should know whether the request is informational or state-changing, whether it needs a tool, what data boundary applies, what deadline must be met, who is acting, and what happens if evidence is incomplete.

The useful taxonomy is the combination of intent and consequence, not “easy prompt” versus “hard prompt”.

| Request class | Typical route posture |
| --- | --- |
| Informational | Fast eligible model, no account mutation, low-latency target |
| Evidence-backed | Approved model plus read-only retrieval or tool, grounded response required |
| Ambiguous or sensitive | Stronger reasoning, clarification, restricted context, or human review |
| State-changing | Independently authorised tool, server-side validation, idempotency, explicit approval policy |

The same words can map to different risk classes depending on authenticated identity, current state, tenant, product, and tool scope. “Refund this” from an unauthenticated session is not the same request as a refund after identity verification and a policy-approved dispute flow.

The model may interpret intent. It should not manufacture authority. The application and identity layers should attach an attested actor, tenant, session, purpose, data labels, and allowed action scope. The broker should enforce them independently of the model’s prose.

When policy is unavailable, the contract should specify the degraded path. A low-risk read-only request may use a signed local snapshot. Sensitive data may be limited to a pre-approved deployment. A mutation may stop and preserve the transaction identifier for review. These are product decisions, but making them explicit is what turns safety from a principle into an executable boundary.

### Build the smallest system that can prove you wrong

The project should be small enough to run and rigorous enough to falsify its own thesis.

Start with an OpenAI-compatible request surface. The application sends one request and receives a response or stream. It does not know whether the request reached a managed provider, an internal endpoint, a fallback, or a review queue. The gateway owns that decision and returns route metadata suitable for debugging.

The control plane needs only a few versioned records: a model and deployment registry, a policy registry, pricing and quota data, evaluation results, and rollout state. The data plane authenticates the caller, classifies the request, filters inadmissible routes, scores the remaining candidates, enforces deadline and budget, calls an adapter, applies bounded recovery, validates the result, and emits a trace.

The first release should use mock or OpenAI-compatible providers with controlled failure injection. Inject latency, throttling, malformed structured output, context overflow, provider errors, and connection loss after streaming begins. A router that only works when every provider is healthy has demonstrated a happy-path proxy, not a routing system.

The budget should be enforced before execution, not reconstructed after the invoice arrives. At admission time, the gateway can reserve a maximum path budget based on the request class: the initial call, one allowed retry, expected tool work, and any review path. It can release the unused reservation when the request finishes and charge the actual ledger to the product and tenant. A state-changing request may have a higher ceiling but a stricter approval path; a low-risk informational request may be rejected or degraded when the remaining budget cannot meet its deadline.

The [AWS multi-provider gateway reference architecture](https://aws.amazon.com/blogs/machine-learning/streamline-ai-operations-with-the-multi-provider-generative-ai-gateway-reference-architecture/) treats cost tracking, budget controls, rate limiting, access management, failover, and observability as one operating surface. That combination is more useful than a token dashboard alone. The ledger should attribute input tokens, output tokens, cached context, router and verifier calls, retrieval and tool charges, gateway compute, provider fallback, and human review to the same request, product, and tenant dimensions.

Price changes and route-distribution changes should be observable events. If a provider changes its price, the registry needs a new effective-dated price version. If a policy suddenly sends twice as many requests to a reasoning model, the alert should show whether traffic changed, the classifier drifted, a cheaper route degraded, or a provider became unavailable. Otherwise FinOps becomes a monthly report rather than a control loop.

The benchmark should compare three baselines before any learned policy:

| Baseline | What it reveals |
| --- | --- |
| Always cheap | The lower bound on first-call spend and the cost of under-capability |
| Always strong | The quality and latency ceiling with less routing complexity |
| Deterministic router | Whether explicit policy and simple cascades create value |

The claim should be falsifiable:

> On a representative high-stakes support and operations workload, a policy-aware router reduces cost per accepted outcome while preserving a defined quality floor, safety boundary, and latency SLO.

The claim is meaningful only after the quality floor, SLO, and workload slices are written down. Do not promise a particular percentage before the workload exists. A credible result might show that the deterministic router sends routine questions to a fast model, reserves the stronger route for ambiguity, blocks unauthorised tools, and spends more when the cost of being wrong is higher.

The strongest demo is not a chart where the cheapest model handles the most traffic. It is one trace that explains why a cheap route was safe for one request, why a stronger route was necessary for another, why a provider was removed from the candidate set, why a fallback did not preserve a mutation, and how the system knew the result was accepted.

That is what turns an architecture into evidence.

## Case Study: Red Hat's Semantic Router

The most useful case study for this design is not a claim that a router saves a universal percentage. It is a concrete example of where semantic routing meets the inference-serving layer.

In [Red Hat's LLM Semantic Router article](https://developers.redhat.com/articles/2025/05/20/llm-semantic-router-intelligent-request-routing), the router sits behind Envoy as an External Processor. A client sends an OpenAI-compatible request to Envoy. The semantic processor extracts the prompt, generates an embedding with a configured BERT model, compares it with task vectors, and selects the model associated with the detected task. Red Hat describes several use cases: sending mathematics to a specialised model, routing simpler requests to cheaper models, serving semantically similar requests from a cache, and detecting PII so a request can be redirected, redacted, or rejected before it reaches a public model.

<figure class="article-figure article-figure-plain">
  <img src="/images/illustrations/red-hat-envoy-semantic-router-flow.png" width="1413" height="741" loading="lazy" decoding="async" alt="A client sends an OpenAI-compatible request through an Envoy proxy to a semantic LLM router, which analyzes the request and routes a modified request to an LLM backend.">
  <figcaption>A simplified Envoy–semantic-router flow based on Red Hat's engineering case study. The diagram is an explanatory visual supplied for this article, not Red Hat's official figure.</figcaption>
</figure>

The implementation detail is important. Red Hat describes a hybrid Rust and Go design: Rust's Candle library handles BERT embedding generation, similarity matching, and classification; Go bindings expose that work to a Go-based ExtProc server; Envoy handles interception and request/response modification. This is a different trade-off from putting a large classifier or another LLM directly in the request path. The router tries to keep semantic classification fast enough to sit in front of the actual inference call while preserving a familiar client contract.

The case study also makes the cache path explicit. If a new request is semantically similar to a prior request, the router may return a cached response and avoid backend inference. That can be powerful for repeated informational workloads, but it is not safe as a generic answer cache. A production cache key should include the policy version, tenant or authorization boundary, data zone, model and prompt versions, locale, tool state, and freshness requirement. Personalized answers, live account state, and mutations should bypass semantic response caching or use a domain-specific cache with explicit invalidation and authorization checks.

Red Hat reports Prometheus metrics for request counts by model, routing decisions and modifications, cache hit rates, response latency, and token usage. Those metrics connect directly to the route ledger proposed earlier in this article. They also expose the real cost of semantic routing: embedding generation, similarity search, cache validation, and the risk of a false semantic match must be counted as part of the path. A router that saves model tokens but returns a stale or policy-incompatible answer has not reduced cost per accepted outcome.

The most interesting part of the case study is its planned relationship with the Kubernetes Gateway API Inference Extension. Red Hat describes semantic understanding as a complement to the extension's queue and KV-cache awareness: semantic routing can choose the model or task pool, while the inference scheduler can choose the best endpoint inside that pool. This is the two-layer architecture in concrete form. Content semantics and serving state are different signals, and a production system should compose them rather than force one router to own both.

The article does not present Red Hat's work as an anonymised production benchmark with a public latency or cost number. It is an engineering implementation and integration direction. That makes it more useful here: it gives the project a credible technical seam to reproduce and test. A strong implementation would compare a rule-only router, a semantic router, and a semantic-plus-serving-aware router under repeated prompts, cache hits and misses, PII cases, long contexts, provider degradation, and policy changes.

The price of a token is easy to see. The cost of a decision is not. It appears in a retry nobody planned, a tool call that should have been blocked, a long context sent to the wrong deployment, a fallback that crosses a data boundary, a reviewer repairing a confident answer, and a user who asks the same question again because the first response sounded right but did not resolve the problem.

The job of an LLM Gateway is to make those hidden paths visible and governable.

It should know what the request is allowed to do, which routes are capable of doing it, which provider can do it now, how much recovery is still affordable, and what evidence will remain when the request is over. It should optimise within policy, not around it. It should treat a tool mutation differently from an informational answer. It should learn from outcomes without silently rewriting the rules that protect them.

<mark>The cheapest LLM is often the most expensive one because the token is only the beginning of the route.</mark>

The real optimisation target is not the model. It is the path to a correct, timely, safe, and accepted outcome.

**Sources and scope**

The production evidence in this essay comes from reported case studies and product documentation: [Vercel's AI Gateway Production Index](https://vercel.com/blog/ai-gateway-production-index-july-2026), [Uber's GenAI Gateway](https://www.uber.com/co/en/blog/genai-gateway/), [Uber's agent identity work](https://www.uber.com/gb/en/blog/solving-the-agent-identity-crisis/), [Google Cloud's GKE Inference Gateway case study](https://cloud.google.com/blog/products/containers-kubernetes/how-gke-inference-gateway-improved-latency-for-vertex-ai), [Microsoft's model router guidance](https://learn.microsoft.com/en-us/azure/foundry/openai/concepts/model-router-how-it-works), [AWS's resilience demonstration](https://aws.amazon.com/blogs/machine-learning/implementing-resilience-patterns-with-amazon-bedrock-and-llm-gateway/), [AWS's multi-provider gateway reference architecture](https://aws.amazon.com/blogs/machine-learning/streamline-ai-operations-with-the-multi-provider-generative-ai-gateway-reference-architecture/), and [Red Hat's LLM Semantic Router case study](https://developers.redhat.com/articles/2025/05/20/llm-semantic-router-intelligent-request-routing). Their figures are reported by the cited organisations and should be read within each source's workload and measurement window. Red Hat's architecture is cited here as an engineering reference; no public production KPI is inferred from it.

The research and evaluation discussion draws on [RouteLLM](https://arxiv.org/abs/2406.18665), [FrugalGPT](https://arxiv.org/abs/2305.05176), [Anthropic's agent-building guidance](https://www.anthropic.com/engineering/building-effective-agents), [Anthropic's evaluation guidance](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents), [Microsoft's model-router evaluation protocol](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/model-router), and [tau-bench](https://arxiv.org/abs/2406.12045). The architecture, formulas, failure policy, and project boundary are my synthesis for a production LLM Gateway, not a claim that any one company implements every step exactly this way.

The technical boundary draws on the [Kubernetes Gateway API Inference Extension](https://gateway-api-inference-extension.sigs.k8s.io/), [OpenTelemetry's GenAI semantic-convention work](https://opentelemetry.io/blog/2026/genai-observability/), [OAuth 2.0 Token Exchange RFC 8693](https://www.rfc-editor.org/rfc/rfc8693.html), and the [OWASP Top 10 for LLM Applications](https://genai.owasp.org/resource/owasp-top-10-for-llm-applications-2025/). These sources define technical patterns and risks; they do not validate the complete architecture proposed here.
