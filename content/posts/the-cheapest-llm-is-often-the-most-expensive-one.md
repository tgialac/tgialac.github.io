---
title: "The Cheapest LLM Is Often the Most Expensive One"
date: 2026-08-23
lastmod: 2026-08-23
description: "Why the right unit of optimisation is not the token, but the cost of a correct, timely, and reliable outcome."
summary: "A first-principles introduction to model routing, fallback, and the hidden cost of choosing a cheap LLM."
tags: ["LLM Gateway", "AI Infrastructure", "Model Routing", "Production AI"]
math: true
draft: false
---

## Why this title?

Because the cheapest model call is not necessarily the cheapest way to finish a task.

A small model may cost a fraction of a cent and still be the wrong choice if it cannot follow a tool schema, reason over a long context, or recognise that a request is high-risk. The system then pays for another model call, a retry, a failed tool execution, a longer agent trajectory, or a human review. The first call was cheap. The path to a correct outcome was not.

<mark>The unit I want to optimise is not the token. It is the cost of a correct, timely, and reliable outcome.</mark>

## 1. Introduction: The $0.001 Request That Costs $0.10

Let us start with a small request.

Suppose a customer asks an AI support agent a question that looks easy enough for a small model. The model call costs $0.001. It produces a plausible answer, but misses an important detail and emits an invalid tool call. The application retries with a stronger model, calls the transaction service again, runs a policy check, and eventually sends the case to a human reviewer.

The numbers below are illustrative, not a reported incident:

| Cost component | Illustrative cost |
| --- | ---: |
| Initial cheap model call | $0.001 |
| Router / verifier overhead | $0.004 |
| Retry with a stronger model | $0.015 |
| Extra tool and context steps | $0.030 |
| Human review / rework | $0.050 |
| **Total path to resolution** | **$0.100** |

The mistake is not that the small model was cheap. The mistake is treating the price of the first inference as the price of the task.

An application does not buy tokens for their own sake. It buys a result: a support case resolved, a transaction explained, a document classified, or a piece of code that passes review. A useful cost model is therefore:

<p class="concept-equation">\[\text{cost per successful outcome} = \frac{\text{initial calls} + \text{router} + \text{retries} + \text{tools} + \text{review}}{\Pr(\text{success})}\]</p>

This is why a more capable model can sometimes be cheaper in practice. It may cost more per call, but reach the quality bar in one attempt. Conversely, a cheaper model can be the expensive option when it creates enough uncertainty downstream.

This is not just a theoretical concern. Production traffic is already showing that the industry is moving away from the idea of one model per application.

### Production does not look like a leaderboard

In [Vercel's AI Gateway Production Index](https://vercel.com/blog/ai-gateway-production-index), Vercel analysed seven months of anonymised, aggregate traffic from more than 200,000 teams. The gateway carried tens of trillions of tokens across hundreds of models. At the highest-volume tier, teams used an average of 35 models in regular use.

That is not what a single-model architecture looks like.

It looks more like a routing graph, with each model occupying a role rather than competing for one universal leaderboard position:

| Role | Typical responsibility | Why it may be routed there |
| --- | --- | --- |
| Fast model | Intent classification, extraction, simple FAQ | Low latency and low cost |
| Strong model | Ambiguous reasoning, policy interpretation, high-risk cases | Higher quality floor |
| Specialist model | Embeddings, vision, summarisation, or tool selection | Capability fit |
| Verifier | Grounding, schema, policy, or confidence checks | Decide whether to accept or escalate |
| Fallback / human | Provider failure or unresolved uncertainty | Preserve availability and safety |

Different models win different layers of the same application. A fast model may handle intent classification. A stronger one may reason over evidence. Another may be used for embeddings, vision, summarisation, or tool selection. The useful question is not “Which model is best?” but “Which model is appropriate for this request, under this budget and this risk profile?”

Vercel's data also exposes a less obvious reliability problem. Roughly 3.5% of requests on its AI Gateway completed only after a fallback: the first route hit an error, rate limit, or timeout, and the gateway reissued the request to a healthy alternative. The rescued requests represented 5.1% of tokens and 4.9% of market cost. The cost-weighted rate was higher because long-context requests, multi-step agent runs, and heavy reasoning calls are both larger and more likely to fail under load.

<mark>A provider's request-level uptime is not the same thing as an application's cost-weighted uptime.</mark>

The distinction matters. If a cheap FAQ request fails, the user may try again. If a long-running agent fails after ten tool calls, the system may have already paid for a large amount of context and computation. The expensive end of the workload deserves a different reliability policy from the cheap end.

### The fallback that kept ten requests alive

There is a small but useful demonstration in [AWS's LLM gateway resilience patterns](https://aws.amazon.com/blogs/machine-learning/implementing-resilience-patterns-with-amazon-bedrock-and-llm-gateway/). AWS configured a primary model with a restrictive limit of three requests per minute and a fallback with capacity for 25 requests per minute. When the client sent ten concurrent requests, three went to the primary model and the remaining seven were diverted to the fallback. All ten completed without application-level retry logic.

This is a demonstration, not an anonymised production incident, and it should not be read as a universal availability guarantee. Its value is architectural: the client did not need to understand the provider quota. The gateway owned the decision about where the request should go next.

That same gateway can make the decision more intelligent than “retry anywhere”. A read-only request may move to a secondary provider, while a request containing restricted data may use only an approved deployment. A structured tool call may require schema compatibility before retrying. A payment mutation may stop rather than replay automatically, and a high-risk support request may require a stronger model and a human checkpoint.

Fallback is therefore not merely an availability feature. It is a policy decision.

### The hidden cost of being slightly wrong

The cost of a model error is rarely confined to the response itself. A low-quality first answer can create another model invocation, another retrieval pass with a longer context, an invalid tool call and repair loop, an unnecessary escalation to customer support, a delayed response that violates the product's latency target, or a business decision made from an answer that sounded confident but was not grounded.

This is why OpenAI's guidance for enterprise AI recommends evaluating the full cost of reaching an accepted outcome, including model and tool usage, retries, completion rate, latency, and human review. The same guidance treats model routing, observability, evaluations, and reusable agent patterns as shared capabilities rather than application-specific glue.

The practical consequence is uncomfortable but useful: **model selection is no longer a configuration detail**. It is part of the application's runtime behaviour.

### A request is already a distributed system

Consider a Vietnamese digital-wallet user asking:

> “Why is my transfer still pending?”

The user sees one sentence. A production system may need to authenticate the user and establish the correct tenant or product context, classify whether this is a general FAQ or a request about live account state, retrieve the relevant transaction through an authorised service, minimise or redact sensitive data before it enters a model context, select a model that can explain the evidence within the latency budget, call a provider with available quota and an approved data policy, retry or fall back if that route times out or is rate-limited, verify that the final answer is grounded in the transaction state, and record the provider, model, tokens, cost, policy version, and outcome.

The question is one line. The execution is not.

<figure class="article-figure article-figure-plain">
  <img src="/images/illustrations/gateway-tool-broker-architecture.png" width="1580" height="689" alt="Architecture showing autonomy policy, identity and attested intent, data labels and memory rules, and context trust segregation flowing through a gateway and tool broker into policy-based action, telemetry, human review, or a blocked and logged result.">
  <figcaption>The gateway and tool broker form a complete mediation boundary: no model call or tool call bypasses the broker.</figcaption>
</figure>

The architecture makes the missing control point explicit. Autonomy policy, identity, data labels, memory rules, and context trust are inputs to mediation. The broker then turns them into one of three outcomes: an allowed action with telemetry, a human review or override, or a blocked, rolled-back, and logged attempt.

The important invariant is not merely that the gateway can route to another provider. It is that **no model call or tool call bypasses the broker**. Without that invariant, a system can have an impressive policy layer on paper while a hidden direct call still escapes its data, authorization, or audit boundary.

Something has to translate between the application's semantic world and the provider's API world. If every application implements that translation independently, routing rules, provider credentials, quotas, retries, redaction, and cost accounting spread across the codebase.

That is the problem an LLM Gateway is meant to solve.

### A gateway is not just a reverse proxy

A traditional reverse proxy forwards traffic. An LLM Gateway must also understand the request well enough to decide which model is allowed to see it, which provider is healthy enough to receive it, how much it may cost, whether it can be retried safely, whether the result meets the quality bar, and what evidence must be retained afterward.

[Anthropic's work on effective agents](https://www.anthropic.com/engineering/building-effective-agents) makes a related point from the application side: start with the simplest workflow that solves the problem, and add agentic complexity only when its benefit justifies the added latency and cost. The gateway is where that discipline becomes operational. It can route a simple FAQ to a fast model, reserve stronger reasoning for ambiguous cases, and stop a high-risk request from being “optimised” into an unsafe cheap path.

The gateway earns its place when it makes decisions that individual applications should not have to duplicate.


**The thesis**

<mark>An LLM Gateway is the control plane for production AI applications.</mark>

It decides which model is allowed to see a request, which provider is healthy enough to receive it, how much the request may cost, what data may cross a boundary, when a failure can be recovered, and what evidence must be retained afterward.

The model generates the answer. The gateway determines the conditions under which that answer is allowed to exist.

In the next parts, I will build this idea from first principles: the difference between a model, provider, deployment, gateway, control plane, and data plane; routing across quality, latency, cost, capability, risk, and availability; fallbacks, retries, quotas, and failure containment; evaluation of routing decisions rather than only final answers; and eventually a concrete model router for Vietnamese fintech customer support.

For now, the first conclusion is enough:

> **The cheapest LLM call is often the most expensive path to a correct answer.**



## 2. The Model Is Not the Route

In the previous part, I argued that the price of the first model call is a poor proxy for the cost of reaching a correct outcome.

That argument creates a more difficult question:

> If the cheapest model is not always the cheapest path, what exactly is a path?

It is not just a model name.

A production request passes through an identity boundary, a data policy, a capability check, a routing decision, a provider endpoint, a retry policy, and an evaluation loop. The model is one component in that path. It is not the path itself.

<mark>The model generates the answer. The gateway decides the conditions under which that answer is allowed to exist.</mark>

### A model is not a provider

The distinction sounds obvious until it disappears into configuration.

| Thing | What it means | Why the gateway cares |
| --- | --- | --- |
| Model | A learned capability, such as reasoning, vision, embedding, or generation | Defines what the request may be able to do |
| Provider | The organisation or platform serving that capability | Defines API semantics, data terms, quotas, regions, and failure modes |
| Deployment | A concrete endpoint, account, region, or self-hosted replica | Defines capacity, latency, network boundary, and operational health |
| Route | The complete decision for this request | Combines model, provider, deployment, policy, budget, and fallback |

The same model family can behave differently when it is served through different deployments. A provider may expose different quotas by region. A self-hosted model may have better data locality but worse availability. A managed endpoint may have a stronger SLA but a higher price or a stricter context limit.

So the route cannot be represented safely by a model name alone. It is a decision about capability, data zone, quality floor, latency budget, cost ceiling, availability, and recovery. This is why a model catalog is not just a list of endpoints. It is a registry of capabilities, constraints, prices, regions, versions, evaluation results, and operational health.

The first router operation is therefore not scoring. It is filtering.

<mark>The cheapest model is irrelevant if it is not an admissible model.</mark>

Most early implementations start with a small map: a fast model, a strong model, and perhaps a local model. That map is useful for getting started, but it is not enough for production. A long-context task should not be routed to a model whose context window is too small. A tool-using agent should not be sent to a deployment that cannot guarantee the required schema. A request containing restricted data should not be routed to an endpoint that has not passed the organisation's data review.

Versioning matters for the same reason. A model alias that silently points to a new version can change structured output, tool selection, refusal behaviour, token usage, or latency. A catalog gives the organisation somewhere to record that change and somewhere to roll it back.

Uber learned that the integration problem comes before the routing problem. Its GenAI Gateway was built after teams began integrating external and internally hosted models in different ways. The platform exposed a consistent interface across OpenAI, Vertex AI, and Uber-hosted models. Uber reported that the gateway was used by close to 30 customer teams, served about 16 million queries per month, and reached a peak of 25 requests per second.

The important part is not the number. It is what the gateway had to centralise: PII redaction, authentication and authorisation, cost attribution, metrics, audit logs, and a standard security review before a use case could access the platform. [Uber's account of the GenAI Gateway](https://www.uber.com/en-CO/blog/genai-gateway/) describes a platform boundary, not merely a provider adapter.

Uber also chose an OpenAI-compatible HTTP/JSON interface. That reduced integration friction because existing clients and tools already understood the shape. But compatibility at the edge did not mean that every provider behaved identically behind the edge. The gateway still had to translate provider-specific clients, model capabilities, credentials, and response semantics.

This suggests a useful design principle:

> Keep the application-facing contract boring. Put the difficult differences behind the contract.

### A gateway is a control plane

Calling the gateway a reverse proxy understates its responsibility. A reverse proxy mostly forwards traffic. A model gateway decides whether traffic is admissible, where it should go, how it should be recovered, and what evidence should remain afterward.

The distinction becomes clearer when we separate two planes.

| Plane | Responsibility | Typical components |
| --- | --- | --- |
| Control plane | Define what is allowed and how the system should behave | Model registry, policy registry, pricing, quotas, evaluation results, routing configuration, rollout and rollback |
| Data plane | Execute one request under those rules | Authentication, redaction, routing, provider adapter, queue, retry, fallback, response validation, telemetry |

Without a control plane, routing logic spreads across application code. One service checks a hard-coded model name. Another has its own retry loop. A third stores provider keys in a deployment secret. A fourth records token usage in a different format. Each local decision may appear reasonable. Together, they create a system that cannot answer a basic question: why did this request use this model and expose this data to this provider?

The gateway is valuable because it turns those decisions into a shared, inspectable policy.

Routing should then happen in two stages. First, the gateway applies hard constraints: data policy, capability, identity, risk, availability, and budget. Only after those constraints have removed inadmissible candidates should the router optimise the remaining choices across quality, latency, cost, and reliability.

This order matters. A weighted score can hide a policy violation. A very cheap endpoint might receive a high score if cost has too much weight, even though the data should never have crossed that provider boundary. Policy is not just another feature in the objective function. It is the boundary around the objective function.

Microsoft's Model Router makes a similar distinction in a managed product. It provides Balanced, Cost, and Quality modes, but it also supports a constrained model subset so organisations can decide which models are eligible for routing. Microsoft recommends using the subset as a compliance gate, observing the selected model in responses, and keeping at least two models available for failover. Its guidance also describes a hybrid pattern: use dynamic routing for general traffic, while retaining direct deployments for specialised or compliance-mandated workloads. [Microsoft's model router guidance](https://learn.microsoft.com/en-us/azure/foundry/openai/concepts/model-router-how-it-works)

That hybrid pattern is more realistic than routing everything dynamically. Some requests should be optimised. Some requests should be pinned.

### Scale changes the meaning of routing

Scalability is not solved by adding more provider names to a dropdown.

LLM traffic has a shape. A short FAQ, a long document, a coding agent, and a burst of concurrent support requests stress different parts of the system. Input length affects prefill. Output length affects decode time. Shared prefixes affect cache reuse. Bursts affect queue depth. A model can be healthy while its deployment is already saturated.

Google's Vertex AI team documented this in a production case involving the GKE Inference Gateway. The gateway used load-aware and content-aware routing: it considered signals such as queue pressure and routed requests with reusable prefixes toward workers that already held the relevant cache. On the reported production workloads, TTFT improved by more than 35% for Qwen3-Coder, P95 TTFT improved by 52% for DeepSeek V3.1, and prefix-cache hit rate increased from 35% to 70%. [Google's production case study](https://cloud.google.com/blog/products/containers-kubernetes/how-gke-inference-gateway-improved-latency-for-vertex-ai)

For an external multi-provider gateway, the signals will look different, but the principle is the same. A provider health check that says “HTTP 200” is not a complete capacity signal. The gateway should observe time to first token, time between tokens, queue delay, timeout rate, rate-limit responses, and cost-weighted failure rate. The fastest route is sometimes the route with a cache hit. Sometimes it is the route with the shortest queue. Sometimes it is the route that avoids a retry.

There is another boundary that is easy to miss in a model-centric diagram. A model call usually produces information. A tool call can change the world.

An assistant may be allowed to read a transaction status but not initiate a refund. It may be allowed to draft a transfer explanation but not submit a transfer. It may be allowed to search a knowledge base but not query another customer's account. The model can propose an action. It should not be the final authority for the action.

Uber's more recent agent architecture makes this separation explicit. Its AI Gateway mediates calls from agents to external models. Its MCP Gateway mediates calls from agents to Uber's downstream systems. A security token service issues short-lived, scoped credentials for each hop. The AI Guard layer handles prompt injection, jailbreaks, content safety, and PII redaction. [Uber's agent identity architecture](https://www.uber.com/gb/en/blog/solving-the-agent-identity-crisis/) is a strong argument for showing model and tool mediation as two related but distinct paths.

For a fintech system, the tool path must carry its own authorisation model. Read and write permissions should be separate. Mutations need idempotency and transaction-state checks. A non-idempotent side effect must not be replayed automatically. High-risk actions need an approval path, and the audit record should include the caller, intent, tool, arguments, policy version, and result.

The model gateway protects the model boundary. The tool gateway protects the business boundary.

Observability must explain the decision, not only the duration. A trace should connect the application, tenant, user, task, risk class, eligible models, filtering constraints, selected route, queue delay, retries, fallback, token usage, cost, tool calls, and final outcome. If a new policy suddenly pushes most traffic to the most expensive model, the dashboard should show that before the invoice does.

AWS's multi-provider gateway reference architecture groups these concerns together: access management, budgets, rate limits, retries, fallback, prompt caching, cost allocation, security events, and performance metrics. Its lesson is practical: governance and observability are not paperwork added after routing. They are part of the routing system. [AWS reference architecture](https://aws.amazon.com/blogs/machine-learning/streamline-ai-operations-with-the-multi-provider-generative-ai-gateway-reference-architecture/)

Telemetry tells us what happened. Evaluation tells us whether it was good enough. A schema failure can lower a model's eligibility for structured-output routes. A grounding failure can raise the quality floor. Repeated retries can increase a route's effective cost. Human escalation can signal that a request class needs a stronger model. A latency regression can change capacity weights.

This does not mean letting a model rewrite routing policy by itself. High-impact changes should be evaluated, reviewed, and rolled out gradually. Anthropic's guidance is useful here: use routing when inputs have distinct categories and the classification decision can be made accurately. Do not add an autonomous loop simply because the framework makes it easy. Complexity should earn its place through measurable improvement. [Anthropic's guidance on effective agents](https://www.anthropic.com/engineering/building-effective-agents)

### The architecture I would build

Without drawing the diagram yet, the request path can be understood as five gates. The gateway first authenticates the caller and attaches tenant, application, user, and delegated intent. It then classifies task type, data sensitivity, risk, latency budget, and cost budget. A preflight safety step handles restricted data, prompt injection, jailbreaks, and disallowed requests.

Next, the gateway filters model candidates by data zone, capability, identity, budget, and availability. The router scores only the admissible candidates. Admission control then enforces quotas, queue limits, concurrency, and provider capacity. The provider adapter executes the request, while timeout, retry, circuit breaker, and safe fallback policies handle failure.

Finally, output checks validate schema, grounding, content policy, and tool-call safety. The gateway records the route, tokens, cost, latency, policy decisions, and outcome. Low-risk informational requests may degrade to a cached answer or a polite retry. A financial mutation should usually stop, preserve transaction state, and create an auditable review path.

Centralisation creates leverage, but it also creates a dependency. If every model call must pass through one gateway, the gateway becomes a critical path. It needs horizontal scaling, regional deployment, key rotation, health-aware routing, and a plan for degraded operation.

There is also a subtle failure question: should the gateway fail open or fail closed? For a low-risk FAQ, a temporary policy-service outage might allow a pre-approved read-only route to continue with a conservative model subset. For a payment mutation, allowing the request through when the policy service is unavailable may be unacceptable. The correct answer depends on the action's risk, not on a universal infrastructure preference.

A score can choose between acceptable routes. It should not decide whether an unacceptable route becomes acceptable because the system is busy.

**The thesis**

The first generation of LLM gateways solved provider fragmentation. The next generation must solve decision fragmentation.

Applications should not each implement their own redaction, retries, model allowlists, budget checks, provider adapters, and audit conventions. They should express intent and constraints. The gateway should turn those constraints into a route that is capable, permitted, observable, and recoverable.

<mark>A model gateway is not a place where requests go to find a model. It is a place where requests go to prove that a model call is allowed.</mark>

The model is part of the route.

It is not the route.

In the next part, I will make the scoring problem explicit: how should a router trade quality against latency, cost, reliability, and risk when no single model wins on every dimension?

## 3. Routing Is a Decision Problem, Not an If–Else Statement

The phrase “model router” can make the problem sound smaller than it is.

It suggests a switch:

If the prompt looks simple, use the cheap model. Otherwise, use the strong model.

That rule is useful as a first experiment. It is not a production policy. A real request can be simple in language but expensive in consequence. It can be difficult to classify, easy to answer incorrectly, or cheap to generate but costly to repair. The router is not merely predicting difficulty. It is deciding which path has the best expected outcome under a set of constraints.

<mark>Routing is constrained optimisation with incomplete information.</mark>

The router does not know the true quality of a response before the response exists. It has to estimate that quality from the request, the conversation, the available tools, the data boundary, the current system state, and the model history. Its decision is made under uncertainty, and the uncertainty is not evenly distributed across requests.

### A score is not a policy

It is tempting to put every concern into one weighted score: quality gets one weight, latency another, cost another, and reliability another. Then the router chooses the model with the highest number.

The problem is that not all dimensions are interchangeable.

A small reduction in cost should not compensate for sending restricted data to an unapproved provider. A five-point quality gain should not justify missing a payment deadline. A healthy provider should not be considered a safe fallback for a non-idempotent tool call just because it has spare capacity.

The router should therefore separate hard constraints from soft preferences. Hard constraints define the admissible set. Soft preferences rank the candidates left inside that set.

| Layer | Question | Typical decision |
| --- | --- | --- |
| Policy constraint | Is this route allowed at all? | Reject an unapproved data zone or tool capability |
| Capability constraint | Can the route perform the task? | Exclude a model without the required schema or context window |
| Service constraint | Can the route meet the operational contract? | Exclude a deployment that cannot meet the latency or availability target |
| Budget constraint | Can the request afford the route? | Enforce a tenant, workflow, or request cost ceiling |
| Preference objective | Which admissible route is best right now? | Trade quality, latency, cost, cache, and provider health |

This ordering is more than a clean diagram. It prevents a dangerous class of bugs in which a numerical objective quietly turns a forbidden action into an attractive action.

Once the admissible set is known, the router can choose a policy for the workload. A customer-support FAQ may prefer low latency and low cost. A fraud investigation may prefer quality and evidence preservation. A batch classification pipeline may accept higher latency in exchange for a lower cost. A transfer mutation may not be routed dynamically at all; it may be pinned to an approved deployment and require a separate approval path.

There is no universal best route because there is no universal definition of “best”.

### What the router can learn

There are several ways to build the decision layer, and each one has a different amount of data, complexity, and explainability.

The simplest router is rule-based. It maps task type, risk, language, context length, and tool requirements to a model pool. Rules are easy to audit and are often the right starting point. Their weakness is that they do not learn the boundary between a request that looks simple and one that only looks simple.

A classifier-based router learns that boundary. It can estimate task category, difficulty, or the probability that a weaker model will meet the quality bar. The classifier does not need to generate the final answer, but it adds its own latency and cost. For short requests, the router can become a material fraction of the entire bill.

A preference router learns from comparisons between model responses. RouteLLM, developed by researchers associated with LMSYS and UC Berkeley, trains routers using preference data to choose between a stronger and weaker model. The paper reports that its routers can reduce cost substantially on several benchmarks while preserving response quality, and it shows transfer across model pairs. [RouteLLM](https://arxiv.org/abs/2406.18665) is important because it treats routing as a learned preference problem rather than a static model leaderboard.

A cascade takes a different approach. It starts with a cheaper model and escalates only when the result appears unlikely to meet the quality bar. FrugalGPT describes this family of strategies as a learned combination of model calls that can reduce cost while maintaining or improving performance. [FrugalGPT](https://arxiv.org/abs/2305.05176) captures the appeal of a cascade: do not pay for the strongest model until the request gives you a reason to.

But a cascade has a hidden price. The first answer must be generated before the system knows whether to escalate. The verifier may itself be another model call. The final path can therefore cost more than a single strong call, especially when the quality estimator is poorly calibrated or when the request is already short and cheap.

The routing call is part of the route.

That sounds obvious, but it is easy to benchmark the downstream model prices while forgetting the classifier, verifier, tokenisation, queue time, and telemetry overhead introduced by the gateway. A router that saves 40 percent on model inference but adds 50 percent in routing and verification cost has optimised the wrong boundary.

### The cheapest admissible route

For a production router, “cheap” should mean the lowest expected cost of an accepted outcome, not the lowest price of a single call.

That expectation includes the probability of passing the quality check, the probability of retrying, the probability of using a tool, the probability of human review, and the operational cost of missing the latency target. A model with a low token price can lose this comparison if it fails often on the request class that matters.

This is also why the router should prefer a Pareto frontier over one fixed global ranking. One model may be cheaper and slower. Another may be faster and less accurate. A third may be expensive but stable for high-risk tasks. The useful question is not which model wins overall. It is which models remain non-dominated for this workload and policy.

Amazon Bedrock’s Intelligent Prompt Routing is a useful managed example. It predicts response quality for the available models and chooses a route that balances quality and cost. AWS says the feature can reduce cost by up to 30 percent without compromising accuracy, but its documentation also states that the router is optimised for English prompts and cannot adapt decisions to application-specific performance data. [Amazon Bedrock Intelligent Prompt Routing](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-routing.html)

That limitation is not a footnote for a Vietnamese fintech system. A router trained on broad English data may understand prompt difficulty while missing domain-specific failure modes: a polite but incorrect explanation of a pending transfer, a subtle distinction between a reversed and a failed transaction, or a response that sounds helpful but violates a local policy. The router must be evaluated on the workload it is actually serving.

This suggests a more honest operating model. Start with a deterministic policy and a small approved model pool. Add a learned router only after collecting representative prompts and outcome labels. Keep a strong baseline route so that the router can be compared against something stable. Log the selected model even when the router is managed by a cloud provider. Then promote a routing policy only when it improves the accepted outcome metric without breaking latency, risk, or observability requirements.

The decision should also be explainable after the fact. “The router chose Model B” is not enough. A useful record says that the request was classified as a low-risk Vietnamese FAQ, that two models passed the data and capability filters, that Model B was healthy, that its predicted quality cleared the floor, and that it fit the remaining latency and cost budget.

**The thesis**

The router should not ask, “Which model is best?”

It should ask, “Which admissible route is most likely to produce an accepted outcome for this request, under this policy and this system state?”

That question makes room for a cheap model, a strong model, a specialist, a cached answer, a human, or no model call at all. It also makes room for uncertainty. When the router cannot distinguish two routes confidently, it can choose a safe baseline, run a verifier, or escalate instead of pretending that a precise score is a precise truth.

<mark>A good router does not make every decision dynamic. It makes the right decisions dynamic.</mark>

The next problem is what happens after the chosen route fails, times out, is throttled, or returns an answer that is not good enough. That is where fallback, retry, quota, and failure containment stop being implementation details and become part of the economics of the system.

## 4. Resilience Is a Routing Decision

The word “fallback” often creates a comforting picture: the primary model fails, so the gateway quietly tries another one.

That picture is incomplete.

A fallback can duplicate a charge, repeat a tool call, exceed a user's deadline, cross a data boundary, or turn a temporary provider problem into a retry storm. Resilience is not the number of backup models in a configuration file. It is the discipline of deciding which failures are recoverable, under which deadline, with which side effects, and at what cost.

### A retry is not a recovery plan

The first question after an error is not “Should we retry?” It is “What kind of failure was this?”

A rate limit means the provider has asked the caller to slow down. A transient service error means the provider may recover. An invalid request means repeating the same request will not help. A context overflow means the request needs a different representation or a different model. A policy rejection means the request must not be replayed through a less restricted route.

These failures may all appear as unsuccessful HTTP calls, but they require different actions.

| Failure | Safe first response | What should not happen |
| --- | --- | --- |
| Rate limit or quota exhaustion | Wait within the deadline, use an isolated quota, or select an approved alternative | Repeated immediate retries against the same exhausted bucket |
| Transient provider error | Exponential backoff with jitter, then a health-aware fallback | Every client retrying at the same instant |
| Invalid request or unsupported capability | Reject or route to a compatible deployment | Replaying the same incompatible payload |
| Context overflow | Compress, retrieve selectively, or choose a larger approved context | Blindly sending the same oversized prompt again |
| Policy or data rejection | Stop and explain the blocked path | Sending the data to a weaker or less governed provider |
| Tool or business failure | Preserve state and follow the transaction policy | Replaying a non-idempotent mutation automatically |

AWS’s current guidance for production Bedrock workloads makes the same distinction between throttling and service unavailability. It recommends error-specific backoff, rate limiting, circuit breakers, monitoring, quota planning, and fallback rather than treating every error as a generic retry. [AWS on scaling and reliability for Bedrock](https://aws.amazon.com/blogs/machine-learning/optimize-your-applications-for-scale-and-reliability-on-amazon-bedrock/)

The retry needs a deadline, not just a maximum attempt count. If the user has a two-second response budget, a retry policy that waits two seconds before its second attempt is technically resilient and practically useless. The gateway should carry an end-to-end deadline and divide it among queueing, routing, provider execution, validation, and recovery. An attempt that cannot finish within the remaining budget should not start merely because the retry counter allows it.

The same principle applies to cost. A retry consumes tokens again. A fallback may use a more expensive model. A verifier may add another call after the response. The gateway should record the total cost of the recovery path, not just the cost of the successful final attempt.

### Admission control is a form of routing

Most systems think about routing after a request has entered the model server. By then, the damage may already be done. The queue is full, the GPU is saturated, and every additional request makes the tail slower for everyone.

Admission control moves the decision earlier. It decides whether the request should enter the queue, wait in a lower-priority class, move to an asynchronous path, or be rejected with a useful explanation.

Google’s production work on GKE Inference Gateway treats admission control as part of the inference router. The gateway manages queue depth upstream so a burst of requests does not starve individual model workers. In the reported workloads, the median remained stable while P95 TTFT improved by 52 percent. [Google’s GKE Inference Gateway case study](https://cloud.google.com/blog/products/containers-kubernetes/how-gke-inference-gateway-improved-latency-for-vertex-ai)

This is especially important in a multi-tenant gateway. A single team running a long-context agent should not be able to consume the capacity needed by a latency-sensitive support flow. Per-tenant quotas, token budgets, concurrency limits, and priority classes are not administrative decoration. They are routing inputs.

The gateway also needs a retry budget. Without one, every layer can retry independently: the application retries the gateway, the gateway retries the provider, the provider SDK retries the HTTP call, and the agent retries the tool step. One user request becomes a small traffic multiplier precisely when the dependency is least able to absorb it.

A retry budget gives the request a finite amount of recovery work. Once that budget is spent, the system degrades, queues, escalates, or stops. Reliability improves when the system knows when to stop trying.

### Fallback changes the contract

A fallback is not invisible. It may change the model family, system prompt, tool semantics, context window, response style, data residency, or output quality. If those differences matter, the fallback is a different product path and should be evaluated as one.

The gateway should therefore keep a route ledger for each attempt: the original route, the reason for failure, the fallback candidate, the policy checks that candidate passed, the remaining deadline, the additional cost, and the final outcome. This ledger is useful for operations, but it also prevents a misleading success metric. “Request succeeded” hides whether the primary route worked, whether the fallback worked after three retries, or whether the system returned a degraded cached answer.

Cross-region capacity mechanisms illustrate the same distinction. AWS describes cross-region inference as a way to increase throughput and reduce throttling, but it is not automatically a disaster-recovery mechanism for model or provider disruptions. Capacity distribution and failure recovery are related decisions, not interchangeable ones. [AWS resilience patterns for Bedrock and LLM gateways](https://aws.amazon.com/blogs/machine-learning/implementing-resilience-patterns-with-amazon-bedrock-and-llm-gateway/)

For low-risk read-only requests, a fallback may return a cached answer, a shorter response, or a slower asynchronous result. For a high-risk support answer, it may route to a stronger model or a human reviewer. For a payment mutation, the correct fallback may be no second model call at all: preserve the transaction state, return a pending status, and make the next action explicit.

### The failure policy depends on the action

The same provider outage should not produce the same behaviour for every request.

Consider two requests that both fail after the model proposes a tool call. The first asks for a read-only transaction lookup. The gateway may retry the read against an approved deployment if the caller's deadline allows it. The second asks to initiate a refund. Retrying it blindly could create a duplicate mutation if the first request reached the downstream service but the response was lost.

The gateway needs to know whether the operation is read-only, idempotent, conditionally idempotent, or non-idempotent. It needs a request identity that survives retries, and the downstream service needs to honour that identity. The model should not invent this protection in a prompt. It belongs in the protocol between the gateway and the business service.

This is where the tool gateway becomes more than a security feature. It is a transaction boundary. It can require a scoped credential, attach an idempotency key, check the current state, and refuse a replay that is no longer safe. A model gateway can choose a better model after a failure. A tool gateway must first determine whether another attempt is allowed to exist.

**The thesis**

Resilience is not “try another model”. It is a controlled transition from one state to another without losing the request’s policy, deadline, identity, cost, or side-effect guarantees.

<mark>The best fallback is the one that preserves the user’s contract, not merely the one that returns a token.</mark>

Once failure handling is explicit, the next question becomes harder: how do we know whether a route actually produced a good outcome? Availability dashboards can tell us that the provider responded. They cannot tell us whether the answer was grounded, useful, safe, or worth what we paid for.

## 5. Evaluation Is the Router’s Memory

A router without evaluation has no memory. It can record which model answered, but it cannot know whether the choice was good. Over time, it will confuse availability with quality, low price with efficiency, and user silence with satisfaction.

That is how routing systems become confidently wrong.

The evaluation problem is harder than comparing two answers side by side. A route can produce a fluent answer that is factually wrong. It can produce a correct answer after an unnecessary tool call. It can save tokens while increasing human escalations. It can improve average latency while making the P95 experience unusable during the traffic spikes that matter most.

<mark>The object being evaluated is not the response. It is the complete path to an accepted outcome.</mark>

### A benchmark is not an outcome

A benchmark prompt is useful because it is repeatable. A production outcome is useful because it is real. A serious router needs both.

The offline set should contain representative requests, not only clean examples. For a Vietnamese digital-wallet support flow, that means ordinary FAQs, ambiguous questions, code-switched language, transaction states, long conversation history, requests that contain sensitive data, and attempts to make the assistant take an unsafe action. It should contain the cases where a cheap model is likely to succeed and the cases where being slightly wrong is expensive.

The dataset also needs a baseline. Compare the router with an always-strong route, an always-cheap route, and the current production policy. Otherwise, “cost savings” can mean little more than choosing a weaker model and accepting a quality regression.

Microsoft’s guidance for evaluating its Model Router recommends at least 100 prompts from the actual workload, pairwise comparisons with response order swapped, separate quality scoring, and latency comparison through p50, p90, and p95 rather than averages alone. It also recommends examining cost savings by prompt category, because a router can look efficient in aggregate while failing a critical slice. [Microsoft’s model router evaluation guidance](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/model-router)

The number 100 is not a magical sample size for every system. It is a useful floor for a first directional benchmark. The more important principle is that the dataset should reflect the traffic and the quality bar that the product actually owns.

### Three graders and one system of record

No single grader is sufficient for a production router.

Code-based checks are excellent for properties with an objective answer: valid JSON, required fields, citation presence, tool name, argument schema, transaction state, or whether a forbidden tool was called. They are fast, cheap, and reproducible, but they cannot judge every valid variation of a natural-language answer.

Model-based graders can compare answers, score a rubric, or judge groundedness. They scale better than human review and capture nuance, but they are not automatically neutral. They need a clear rubric, calibration against experts, and tests for position bias. A judge that always prefers a longer answer can reward the very behaviour the router is supposed to control.

Human review remains the reference for ambiguous, high-risk, or domain-specific outcomes. It is expensive, so it should be used strategically: calibrate the model-based grader, review uncertain cases, sample production traffic, and adjudicate failures that affect policy or money.

Anthropic’s work on agent evaluations describes this combination as code-based, model-based, and human graders. It also makes a distinction between capability evaluations and regression evaluations. Capability tests ask what the system can learn to do; regression tests protect behaviours that already worked. Anthropic reports that Claude Code began with rapid feedback from employees and users, then added increasingly specific evals for concision, file edits, and over-engineering. The evaluation suite became a way to track latency, token usage, cost per task, and error rates on a stable task bank. [Anthropic’s evaluation practice](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)

That progression matters for a router. The first version does not need a perfect universal judge. It needs a small, trusted set of tasks that can tell the team whether a routing change improved or damaged the behaviours the product cares about.

### Evaluate the route, not just the answer

For every test case, the evaluation should preserve the full trace: classification, eligible model set, selected model, prompt and context shape, tool calls, retries, fallback, latency, tokens, cost, validation results, and final outcome.

This makes it possible to distinguish different failures. A correct answer from an inadmissible provider is a policy failure. A correct answer after three retries is a resilience warning. A wrong answer from the cheap model is a quality failure. A correct answer that misses the user’s deadline is a product failure. A human-reviewed answer may be acceptable, but its cost and time must still be visible.

The route metrics should therefore include more than accuracy:

| Metric | What it reveals |
| --- | --- |
| Accepted outcome rate | Whether the request met the product’s quality bar |
| False-cheap rate | How often the router chose a cheaper route that later needed repair or escalation |
| False-expensive rate | How often the router paid for a stronger route unnecessarily |
| Cost per accepted outcome | The actual economic efficiency of the route |
| p50 and p95 latency | The typical and tail user experience |
| Retry and fallback rate | Whether the chosen route is operationally stable |
| Policy violation rate | Whether any route crossed a forbidden boundary |
| Tool correctness | Whether the model selected the right tool with safe arguments |
| Calibration | Whether the router’s confidence predicts real success |

The terms false-cheap and false-expensive are particularly useful. A router that is too conservative sends almost everything to the strongest model. It may look safe, but it leaves cost and latency on the table. A router that is too aggressive sends difficult work to weak models. Its token bill looks attractive until retries, human review, and user dissatisfaction arrive.

Morgan Stanley’s deployment of GPT-4 in wealth-management workflows provides a useful financial-services example. The firm introduced an evaluation framework for AI use cases before deployment and later added translation evaluations and retrieval-focused testing as the document library and multilingual needs expanded. The important lesson is not the specific model. It is the decision to make evaluation a gate before production, not a post-launch reaction. [Morgan Stanley and OpenAI on AI evaluations](https://openai.com/index/morgan-stanley/)

Evaluation should also happen at three different speeds. Offline evaluation is slow enough to inspect and broad enough to compare candidate policies. Shadow traffic lets a new router observe production requests without changing user responses. A canary or A/B rollout tests the policy with real users and real latency, but only after the earlier layers show that it is safe to try.

The traffic split should not be judged only by aggregate averages. A policy can improve the overall cost while degrading high-risk requests, non-English requests, long-context requests, or users on a slower network. The dashboard needs slices that correspond to real product and policy boundaries.

**The thesis**

Evaluation is not a report generated after the router is built. It is the memory that lets the router improve without forgetting what “good” means.

<mark>A router earns the right to optimise only after it can measure what it must not sacrifice.</mark>

The next step is to turn these principles into a concrete workload. A Vietnamese fintech support router gives us a useful test because it combines language variation, sensitive data, tool calls, strict latency expectations, and outcomes whose cost is not visible in the first token.

## 6. A Vietnamese Fintech Router Does Not Start with a Model

It starts with a contract.

The user wants to know whether a transfer is pending, why a fee appeared, how to verify an account, or what to do after a failed payment. The system needs to know more than the sentence. It needs to know whether the request is asking for general information or live account state, whether the answer requires a tool, whether the data is sensitive, whether the user is asking the system to change something, and what must happen if the evidence is incomplete.

The model comes after those questions.

This is a proposed design for a Vietnamese fintech workload, not a claim about any internal MoMo architecture. The point is to make the gateway concrete enough to implement and evaluate.

### Start with intents and consequences

A useful first taxonomy is not “easy prompt versus hard prompt”. It is the combination of intent and consequence.

| Request class | Example | Default route posture |
| --- | --- | --- |
| General knowledge | “What is a virtual card?” | Fast model, no account tool, low latency target |
| Account explanation | “Why is my transfer still pending?” | Approved model plus read-only transaction tool |
| Dispute or recovery | “I was charged twice” | Stronger reasoning, evidence retrieval, possible human review |
| Fraud or security | “I don’t recognise this login” | Safety-first flow, identity verification, conservative escalation |
| Financial mutation | “Refund this transaction” | Tool gateway, idempotency, policy check, explicit approval path |
| Ambiguous or high-risk | “Move the money back” | Clarification or human review before any mutation |

The same words can map to different risk classes depending on the authenticated user, transaction state, and tool capabilities. “Refund this” from an unauthenticated session is not the same request as “refund this transaction” after identity verification and a policy-approved dispute flow.

The router should receive an attested intent, not merely a free-form sentence. The application or identity layer can attach the user, tenant, session, product, and allowed action scope. The model may help interpret the request, but it should not be allowed to manufacture authority that the caller did not possess.

### Use a small, explicit model pool

The first production pool should be deliberately small. A fast model can classify intent, extract structured fields, and answer low-risk FAQs. A general support model can explain evidence and handle ordinary multi-turn conversations. A stronger reasoning model can handle ambiguity, policy interpretation, and difficult disputes. A specialist can process documents or images when the task requires it. A human reviewer remains a route, not an exception outside the architecture.

The model names will change. The capabilities and constraints should not.

For each model, the registry should record Vietnamese quality on the relevant task slices, tool-call correctness, context limit, structured-output reliability, latency distribution, cost, data-zone approval, and failure history. A model that performs well on a general Vietnamese benchmark may still be poor at explaining transaction states or preserving the distinction between “reversed”, “failed”, and “pending”.

ViLLM-Eval provides a useful external reference for Vietnamese language-model evaluation, but it is not a substitute for a fintech dataset. [ViLLM-Eval](https://arxiv.org/abs/2404.11086) was designed as a comprehensive Vietnamese LLM evaluation suite in the VLSP 2023 shared task. It can help expose general language weaknesses, while the product-owned suite must measure the vocabulary, policies, tools, and failure costs of the actual wallet workflow.

This is the difference between language competence and product competence.

### Make routing decisions legible

For a request such as “Why is my transfer still pending?”, the gateway should be able to produce an explanation like this after the fact:

The request was authenticated as a read-only account inquiry. The conversation contained transaction-related data, so the candidate set was restricted to approved deployments. The task required a transaction lookup and Vietnamese explanation, so models without the read-only tool schema were removed. The selected support model met the quality floor and latency budget. The transaction tool returned a pending state. The response was grounded in that state, and no mutation was permitted.

That explanation is more valuable than a model label. It makes the route debuggable, auditable, and improvable.

The same request should produce a different path when the user says, “Refund the transfer now.” The gateway should classify the request as a mutation, check identity and policy, require the refund tool’s constraints, and decide whether a human approval is needed. If the tool is unavailable, the correct result is not an automatic retry through another model. It is a preserved transaction state and an explicit next step.

### Build the evaluation set around failure cost

I would begin with a few hundred anonymised, consented, or synthetic-but-reviewed cases divided across the request classes. The exact number is less important than the coverage. Each class should have normal examples, borderline examples, adversarial examples, and cases where the safest answer is to ask a question or refuse an action.

The set should contain Vietnamese formal language, colloquial Vietnamese, code-switching, missing diacritics, shorthand, multi-turn references, and names or transaction descriptions that look similar. It should include requests where the model can answer from policy, requests that require a read-only tool, and requests where a tool result must override the model’s prior assumption.

The evaluation should not stop at “the answer sounds good”. A transaction-support trial should check whether the right tool was called, whether the arguments were safe, whether the answer matched the returned state, whether sensitive values were minimised, whether the user’s question was actually resolved, and whether the route stayed inside its time and cost budget.

The hardest cases should be weighted by consequence. A slightly awkward FAQ response is not equivalent to a confident explanation of the wrong transaction state. A longer answer is not a failure if it prevents an unsafe refund. A low-cost response is not a success if it causes a second contact or human rework.

### The route policy

The policy can remain simple while the system is young. Low-risk general questions go to the fast route. Account explanations go to an approved support route with read-only retrieval. Ambiguous cases go to a stronger model or clarification flow. Fraud and security requests use a conservative policy with identity checks. Mutations are never decided by model quality alone; they require tool authorization, transaction state, idempotency, and the approval policy.

Over time, the router can learn within those boundaries. It can discover that short fee questions are safe on the fast route, that long code-switched dispute histories need the stronger route, or that a particular provider degrades during a traffic window. It should not learn that a restricted provider becomes acceptable because it is cheaper, or that a mutation becomes replayable because the first response timed out.

The model pool can evolve. The policy boundary should evolve through review.

**The thesis**

The best Vietnamese fintech router is not the one that sends the most requests to the cheapest model. It is the one that knows which requests are cheap to answer, which are cheap to get wrong, and which must never be decided by a model alone.

<mark>In financial AI, routing is part language understanding, part distributed systems, and part institutional responsibility.</mark>

That combination is what makes the problem interesting. It is also what makes a small benchmark result insufficient. A production gateway must show not only that it can save money, but that it can explain every saving, preserve every boundary, and stop when the cost of being wrong is higher than the cost of asking for help.

The final step is to turn the thesis into a buildable project: a thin gateway, a policy-aware router, a reproducible evaluation harness, and a dashboard that makes quality, latency, cost, and failure visible together.

## 7. The Project Is the Proof

An architecture diagram can explain where the boxes are. A working gateway has to explain why a request took a particular path.

That is the difference between describing production AI and building a small piece of it.

The project I would build starts with a stable, OpenAI-compatible request surface. The application sends a request once and receives a normal response or stream. It does not know whether the request went to a managed provider, an internal deployment, a fallback, or a human review queue. The gateway owns that decision and returns enough route metadata for the developer to understand it.

Behind that surface is a deliberately small control plane. A model registry records capabilities, data zones, price, context limits, tool support, evaluation results, and health. A policy registry records which applications and tenants can use which capabilities. A routing policy records the hard constraints and the soft preferences. A versioned configuration makes it possible to compare a new policy with the previous one and roll back without changing application code.

The data plane should be equally explicit. It authenticates the caller, classifies the request, filters inadmissible candidates, scores the remaining routes, enforces the deadline and quota, calls a provider adapter, applies a bounded recovery policy, validates the result, and emits a trace. The first version does not need every provider in the market. It needs enough different failure and capability profiles to prove that the gateway is making real decisions.

The evaluation harness is the centre of the project, not a README appendix. It should run the same task set against an always-strong baseline, an always-cheap baseline, a deterministic router, and any learned router introduced later. It should preserve the full trace and produce a report that shows accepted outcome rate, cost per accepted outcome, p50 and p95 latency, retry rate, fallback rate, policy violations, tool correctness, and false-cheap versus false-expensive decisions.

The benchmark should be able to answer a difficult question honestly: did the router create value, or did it simply lower the quality bar?

### A project with a falsifiable claim

The project should not begin with a promise that it will save a particular percentage. That number would be invented before the workload existed. It should begin with a falsifiable claim:

> On a representative Vietnamese fintech support workload, the policy-aware router can reduce cost per accepted outcome while preserving a defined quality floor and latency SLO.

The claim becomes meaningful only after the quality floor, SLO, and workload are written down. For example, the system might require grounded transaction explanations, zero unauthorised mutation attempts, a target P95 for read-only support, and a maximum cost per accepted case. Those are design decisions, not universal industry constants, and they should be exposed as configuration rather than hidden in the code.

The strongest demo is not a chart where the cheapest model handles the most traffic. It is a trace where the gateway explains why a cheap model was safe for one request, why a stronger model was necessary for another, why a provider was removed from the candidate set, and why a mutation stopped instead of retrying.

Balyasny Asset Management describes a similar operating idea in financial research: models were evaluated on internal benchmarks and proprietary financial data, then selected task-by-task based on empirical performance. The organisation centralised core AI components while allowing teams to customise locally. [Balyasny’s AI research engine](https://openai.com/index/balyasny-asset-management/) is a useful reminder that a shared platform does not require one universal model or one universal workflow.

AWS’s multi-provider gateway guidance reaches the same conclusion from an infrastructure angle. Its reference implementation includes infrastructure as code, deployment options, centralized usage management, observability, and support for models hosted both inside and outside the provider’s own platform. The gateway is valuable because it turns model access into an operable platform capability. [AWS multi-provider gateway reference architecture](https://aws.amazon.com/blogs/machine-learning/streamline-ai-operations-with-the-multi-provider-generative-ai-gateway-reference-architecture/)

### The project’s boundary

The project should not pretend to solve every problem in AI infrastructure. It does not need to train a foundation model, replace a bank’s transaction system, or invent a universal safety classifier. It needs to demonstrate the seam between application intent and model execution.

That seam is already rich enough. It includes provider abstraction, model capability negotiation, policy enforcement, routing, resilience, cost accounting, evaluation, and traceability. A small, coherent implementation that makes those boundaries explicit is more convincing than a sprawling platform that hides them behind five frameworks.

The first release can use mock providers or OpenAI-compatible endpoints so the benchmark is reproducible. The provider adapter should be able to inject controlled latency, throttling, malformed structured output, context overflow, and partial failure. A router that only works when every provider is healthy has not demonstrated routing; it has demonstrated a happy-path proxy.

The second release can connect real model providers and compare their behaviour on the Vietnamese task set. The third can add shadow routing, canary policy rollout, and feedback from reviewed outcomes. Each release should preserve the same request contract and trace schema, so improvements are visible rather than anecdotal.

**The final thesis**

The price of a token is easy to see. The cost of a decision is not.

It appears in a retry that nobody planned, a tool call that should have been blocked, a long context sent to the wrong deployment, a fallback that crosses a data boundary, a human reviewer repairing a confident answer, and a user who asks the same question again because the first answer sounded right but did not resolve the problem.

The job of an LLM Gateway is to make those hidden paths visible and governable.

It should know what the request is allowed to do, which models are capable of doing it, which provider can do it now, how much recovery is still affordable, and what evidence will remain when the request is over. It should optimise within policy, not around it. It should treat a tool mutation differently from an informational answer. It should learn from outcomes without silently rewriting the rules that protect them.

<mark>The cheapest LLM is often the most expensive one because the token is only the beginning of the route.</mark>

The real optimisation target is not the model.

It is the path to a correct, timely, safe, and accepted outcome.


**Sources and scope**

- [Vercel — AI Gateway Production Index](https://vercel.com/blog/ai-gateway-production-index), based on anonymised aggregate AI Gateway traffic through April 2026.
- [AWS — Implementing resilience patterns with Amazon Bedrock and LLM gateway](https://aws.amazon.com/blogs/machine-learning/implementing-resilience-patterns-with-amazon-bedrock-and-llm-gateway/), including the explicitly documented ten-request fallback demonstration.
- [OpenAI — How to manage AI investments in the agentic era](https://openai.com/index/managing-ai-investments-in-agentic-era/), for cost-per-outcome and shared production capabilities.
- [Anthropic — Building effective agents](https://www.anthropic.com/engineering/building-effective-agents), for the workflow-versus-agent trade-off.
- [Uber — Navigating the LLM Landscape: Uber’s Innovation with GenAI Gateway](https://www.uber.com/en-CO/blog/genai-gateway/), describing Uber’s multi-provider gateway, security review, PII redaction, cost attribution, and production usage.
- [Uber — Solving the Identity Crisis for AI Agents](https://www.uber.com/gb/en/blog/solving-the-agent-identity-crisis/), describing the separation between AI Gateway, MCP Gateway, AI Guard, and scoped agent identity.
- [Google Cloud — How GKE Inference Gateway improved latency for Vertex AI](https://cloud.google.com/blog/products/containers-kubernetes/how-gke-inference-gateway-improved-latency-for-vertex-ai), reporting production results for load-aware and prefix-aware routing.
- [Microsoft — How model router works in Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/openai/concepts/model-router-how-it-works), describing routing modes, model subsets, failover, and hybrid direct deployments.
- [AWS — Multi-Provider Generative AI Gateway reference architecture](https://aws.amazon.com/blogs/machine-learning/streamline-ai-operations-with-the-multi-provider-generative-ai-gateway-reference-architecture/), describing governance, resilience, cost, and observability capabilities.
- [Amazon Bedrock — Intelligent Prompt Routing](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-routing.html), for quality prediction, cost-aware routing, and the documented English-only and application-specific limitations.
- [RouteLLM — Learning to Route LLMs with Preference Data](https://arxiv.org/abs/2406.18665), for learned routing from preference data.
- [FrugalGPT — How to Use Large Language Models While Reducing Cost and Improving Performance](https://arxiv.org/abs/2305.05176), for cascade-based model selection.
- [AWS — Optimize your applications for scale and reliability on Amazon Bedrock](https://aws.amazon.com/blogs/machine-learning/optimize-your-applications-for-scale-and-reliability-on-amazon-bedrock/), for error-specific retries, circuit breakers, throttling, and fallback.
- [Anthropic — Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents), for capability and regression evals, grader types, and route-level signals such as latency, tokens, cost, and errors.
- [Microsoft — How to use model router](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/model-router), for representative-prompt methodology and quality, cost, and latency benchmarking.
- [OpenAI — Morgan Stanley uses AI evals to shape the future of financial services](https://openai.com/index/morgan-stanley/), for a real financial-services deployment using pre-deployment and multilingual evaluations.
- [ViLLM-Eval — A Comprehensive Evaluation Suite for Vietnamese Large Language Models](https://arxiv.org/abs/2404.11086), for a Vietnamese-language evaluation reference.
- [Anthropic — Gradient Labs transforms financial services customer support with Claude](https://www.anthropic.com/customers/gradient-labs), for a real regulated-financial-services customer-support deployment example.
- [OpenAI — How Balyasny Asset Management built an AI research engine](https://openai.com/index/balyasny-asset-management/), for task-level model selection, internal financial evaluations, feedback loops, and centralized platform capabilities.

The production figures above are reported by the cited organisations. The architecture sequence and failure-policy recommendations are my synthesis for a production LLM Gateway, not claims that any one company implements every step exactly this way.
