---
title: "The Model Is Not the Route"
date: 2026-08-23
lastmod: 2026-08-23
description: "A model gateway is not an API wrapper. It is the control plane that decides which intelligence, data boundary, and failure policy a request is allowed to use."
summary: "Why production AI needs a control plane between applications and models — and why the most important routing decision is often deciding what cannot happen."
tags: ["LLM Gateway", "AI Infrastructure", "Model Routing", "Production AI"]
math: false
draft: false
---

## 2. The Model Is Not the Route

This is [Part 2 of the LLM Gateway series](/posts/the-cheapest-llm-is-often-the-most-expensive-one/).

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

So the route cannot be represented safely by `model = x`.

A useful route record looks more like this:

| Route field | Example question |
| --- | --- |
| Capability | Can this endpoint produce structured tool calls? |
| Data zone | Is this deployment approved for personal or financial data? |
| Quality floor | Has this model passed the evaluation for Vietnamese support? |
| Latency budget | Can it meet the request's time-to-first-token target? |
| Cost ceiling | Is it allowed to spend more than the request budget? |
| Availability | Is the provider healthy and below its quota? |
| Recovery | If it fails, can the request be retried safely? |

This is why a model catalog is not just a list of model names. It is a registry of capabilities, constraints, prices, regions, versions, evaluation results, and operational health.

### The gateway begins with a catalog, not a proxy table

Most early implementations start with a map:

| Friendly name | Provider endpoint |
| --- | --- |
| `fast` | Provider A model |
| `strong` | Provider B model |
| `local` | Internal model |

That map is useful for getting started. It is not enough for production.

The gateway needs to know whether a model can handle the request before it optimises the request. A long-context task should not be routed to a model whose context window is too small. A tool-using agent should not be sent to a deployment that cannot guarantee the required schema. A request containing restricted data should not be routed to an endpoint that has not passed the organisation's data review.

The first router operation is therefore not scoring. It is filtering.

<mark>The cheapest model is irrelevant if it is not an admissible model.</mark>

This is also where versioning matters. Model behaviour changes even when an application does not. A model alias that silently points to a new version can change structured output, tool selection, refusal behaviour, token usage, or latency. A catalog gives the organisation somewhere to record that change and somewhere to roll it back.

### Uber learned that the integration problem comes before the routing problem

Uber built a GenAI Gateway after teams began integrating external and internally hosted models in different ways. The platform exposed a consistent interface across providers including OpenAI, Vertex AI, and Uber-hosted models. Uber reported that the gateway was used by close to 30 customer teams, served about 16 million queries per month, and reached a peak of 25 requests per second.

The important part is not the number. It is what the gateway had to centralise: PII redaction, authentication and authorisation, cost attribution, metrics, audit logs, and a standard security review before a use case could access the platform. [Uber’s account of the GenAI Gateway](https://www.uber.com/en-CO/blog/genai-gateway/) describes a platform boundary, not merely a provider adapter.

Uber also chose an OpenAI-compatible HTTP/JSON interface. That decision reduced integration friction because existing clients and tools already understood the shape. But compatibility at the edge did not mean that every provider behaved identically behind the edge. The gateway still had to translate provider-specific clients, model capabilities, credentials, and response semantics.

This is a useful design principle:

> Keep the application-facing contract boring. Put the difficult differences behind the contract.

The API should be stable enough for application teams to use. The platform behind it should remain replaceable enough for the infrastructure team to change providers, deployments, and policies without asking every application team to migrate at once.

### A gateway is a control plane with a data plane attached

Calling the gateway a reverse proxy understates its responsibility. A reverse proxy mostly forwards traffic. A model gateway decides whether traffic is admissible, where it should go, how it should be recovered, and what evidence should remain afterward.

The distinction becomes clearer when we separate two planes.

| Plane | Responsibility | Typical components |
| --- | --- | --- |
| Control plane | Define what is allowed and how the system should behave | Model registry, policy registry, pricing, quotas, evaluation results, routing configuration, rollout and rollback |
| Data plane | Execute one request under those rules | Authentication, redaction, routing, provider adapter, queue, retry, fallback, response validation, telemetry |

The control plane changes less frequently than the request path. It stores the rules, while the data plane applies them at runtime.

Without a control plane, routing logic spreads across application code. One service checks a hard-coded model name. Another has its own retry loop. A third stores provider keys in a deployment secret. A fourth records token usage in a different format. Each local decision may appear reasonable. Together, they create a system that cannot answer a basic question: why did this request use this model and expose this data to this provider?

The gateway is valuable because it turns those decisions into a shared, inspectable policy.

### The router’s first job is to eliminate impossible choices

Once the request enters the gateway, routing should happen in two stages.

The first stage applies hard constraints:

| Constraint | Example result |
| --- | --- |
| Data policy | Remove providers that cannot receive restricted data |
| Capability | Remove models without the required context, modality, or tool schema |
| Identity | Remove models not approved for this team, tenant, or application |
| Risk | Remove autonomous mutation paths for a high-risk transaction |
| Availability | Remove providers that are unhealthy, throttled, or out of quota |
| Budget | Remove routes that exceed the request or tenant cost ceiling |

Only after this filtering should the router optimise the remaining choices across quality, latency, cost, and availability.

This order matters. A weighted score can hide a policy violation. A very cheap endpoint might receive a high score if cost has too much weight, even though the data should never have crossed that provider boundary. Policy is not just another feature in the objective function. It is the boundary around the objective function.

Microsoft’s Model Router makes a similar distinction in a managed product. It provides Balanced, Cost, and Quality modes, but it also supports a constrained model subset so organisations can decide which models are eligible for routing. Microsoft recommends using the subset as a compliance gate, observing the selected model in responses, and keeping at least two models available for failover. [Microsoft’s model router guidance](https://learn.microsoft.com/en-us/azure/foundry/openai/concepts/model-router-how-it-works) also describes a hybrid pattern: use dynamic routing for general traffic, while retaining direct deployments for specialised or compliance-mandated workloads.

That hybrid pattern is more realistic than routing everything dynamically.

Some requests should be optimised. Some requests should be pinned.

### The provider is part of the latency model

Scalability is not solved by adding more provider names to a dropdown.

LLM traffic has a shape. A short FAQ, a long document, a coding agent, and a burst of concurrent support requests stress different parts of the system. Input length affects prefill. Output length affects decode time. Shared prefixes affect cache reuse. Bursts affect queue depth. A model can be healthy while its deployment is already saturated.

Google’s Vertex AI team documented this in a production case involving the GKE Inference Gateway. The gateway used both load-aware routing and content-aware routing: it considered signals such as queue pressure and routed requests with reusable prefixes toward workers that already held the relevant cache. On the reported production workloads, TTFT improved by more than 35% for Qwen3-Coder, P95 TTFT improved by 52% for DeepSeek V3.1, and prefix-cache hit rate increased from 35% to 70%. [Google’s production case study](https://cloud.google.com/blog/products/containers-kubernetes/how-gke-inference-gateway-improved-latency-for-vertex-ai) is a reminder that an inference gateway may need to understand workload shape, not only provider health.

For an external multi-provider gateway, the signals will look different, but the principle is the same. A provider health check that says “HTTP 200” is not a complete capacity signal. The gateway should observe time to first token, time between tokens, queue delay, timeout rate, rate-limit responses, and cost-weighted failure rate.

The fastest route is sometimes the route with a cache hit. Sometimes it is the route with the shortest queue. Sometimes it is the route that avoids a retry.

### A model call and a tool call have different blast radii

There is another boundary that is easy to miss in a model-centric diagram.

A model call usually produces information. A tool call can change the world.

The difference is not philosophical. It changes the authorization model.

An assistant may be allowed to read a transaction status but not initiate a refund. It may be allowed to draft a transfer explanation but not submit a transfer. It may be allowed to search a knowledge base but not query another customer’s account.

The model can propose an action. It should not be the final authority for the action.

Uber’s more recent agent architecture makes this separation explicit. Its AI Gateway mediates calls from agents to external models. Its MCP Gateway mediates calls from agents to Uber’s downstream systems. A security token service issues short-lived, scoped credentials for each hop. The AI Guard layer handles concerns such as prompt injection, jailbreaks, content safety, and PII redaction. [Uber’s agent identity architecture](https://www.uber.com/gb/en/blog/solving-the-agent-identity-crisis/) is a strong argument for showing model and tool mediation as two related but distinct paths.

For a fintech system, the tool path deserves at least:

- scoped identity and authorisation;
- explicit read versus write permissions;
- idempotency keys for mutations;
- transaction state checks;
- human approval where the risk policy requires it;
- no automatic replay of a non-idempotent side effect;
- an audit record that includes the caller, intent, tool, arguments, policy version, and result.

The model gateway protects the model boundary. The tool gateway protects the business boundary.

### Observability must explain the decision, not only the duration

A dashboard that says “the request took 4.2 seconds” is not enough to operate a router.

The trace should be able to answer:

1. Which application, tenant, and user initiated the request?
2. What task and risk class did the gateway infer?
3. Which models were eligible, and which constraints removed the others?
4. Which route was selected, and why?
5. Did the request wait in a queue?
6. Did it retry or fall back?
7. How many input and output tokens were used?
8. What did the model call cost?
9. Did a tool call occur?
10. Was the outcome accepted, reviewed, blocked, or retried?

The selected model should be a first-class telemetry field, not a value hidden in a provider adapter. Routing distributions are operational data. If a new policy suddenly pushes 80% of traffic to the most expensive model, the dashboard should show that before the invoice does.

AWS’s multi-provider gateway reference architecture groups these concerns together: access management, budgets, rate limits, retries, fallback, prompt caching, cost allocation, security events, and performance metrics. Its lesson is practical: governance and observability are not paperwork added after routing. They are part of the routing system. [AWS reference architecture](https://aws.amazon.com/blogs/machine-learning/streamline-ai-operations-with-the-multi-provider-generative-ai-gateway-reference-architecture/)

### The gateway needs a feedback loop

Telemetry tells us what happened. Evaluation tells us whether it was good enough.

The router therefore needs a loop connecting production outcomes back to policy and model selection.

| Signal | What it can change |
| --- | --- |
| Schema validation failure | Remove a model from structured-output routes |
| Grounding failure | Raise the quality floor or require verification |
| Retry rate | Penalise a route’s effective cost |
| Human escalation | Increase the model tier for that request class |
| P95 latency regression | Reduce traffic or change the capacity policy |
| Provider throttling | Adjust quota and fallback weights |
| Accepted outcome rate | Promote a route for similar requests |

This is not an argument for letting a model rewrite routing policy by itself. High-impact changes should be evaluated, reviewed, and rolled out gradually. A router can learn from feedback while policy remains governed.

Anthropic’s guidance is useful here: use routing when inputs have distinct categories and the classification decision can be made accurately. Do not add an autonomous loop simply because the framework makes it easy. More steps can improve a task, but they also add latency, cost, and more places to fail. [Anthropic’s guidance on effective agents](https://www.anthropic.com/engineering/building-effective-agents) supports a conservative rule: complexity should earn its place through measurable improvement.

### The architecture I would build

Without drawing the diagram yet, the request path is best understood as a sequence of gates:

| Stage | Responsibility |
| --- | --- |
| 1. Identity | Authenticate the caller and attach tenant, application, user, and delegated intent |
| 2. Classification | Determine task type, data sensitivity, risk level, latency budget, and cost budget |
| 3. Preflight safety | Detect restricted data, prompt injection, jailbreaks, and disallowed requests |
| 4. Candidate filtering | Apply data zone, capability, identity, budget, and availability constraints |
| 5. Route scoring | Optimise quality, latency, cost, and reliability among admissible candidates |
| 6. Admission control | Enforce quotas, queue limits, concurrency, and provider capacity |
| 7. Provider execution | Translate the request through a provider-specific adapter |
| 8. Recovery | Apply timeout, retry, circuit breaker, and a safe fallback policy |
| 9. Output checks | Validate schema, grounding, content policy, and tool-call safety |
| 10. Evidence | Record route, tokens, cost, latency, policy decisions, and outcome |

For low-risk informational requests, a failure may degrade to a cached answer or a polite retry. For a financial mutation, the same failure should usually stop the action, preserve the transaction state, and create an auditable review path.

The gateway should not have one universal failure mode. It should have failure policies by risk class.

That is the point at which an AI Gateway becomes more than a convenient abstraction. It becomes a decision system.

### The gateway has a cost of its own

Centralisation creates leverage, but it also creates a dependency.

If every model call must pass through one gateway, the gateway becomes a critical path. It needs horizontal scaling, regional deployment, key rotation, health-aware routing, and a plan for degraded operation.

There is also a subtle failure question: should the gateway fail open or fail closed?

For a low-risk FAQ, a temporary policy-service outage might allow a pre-approved, read-only route to continue with a conservative model subset. For a payment mutation, allowing the request through when the policy service is unavailable may be unacceptable. The correct answer depends on the action's risk, not on a universal infrastructure preference.

This is another reason to separate policy from scoring. A score can choose between acceptable routes. It should not decide whether an unacceptable route becomes acceptable because the system is busy.

### The thesis

The first generation of LLM gateways solved provider fragmentation. The next generation must solve decision fragmentation.

Applications should not each implement their own redaction, retries, model allowlists, budget checks, provider adapters, and audit conventions. They should express intent and constraints. The gateway should turn those constraints into a route that is capable, permitted, observable, and recoverable.

<mark>A model gateway is not a place where requests go to find a model. It is a place where requests go to prove that a model call is allowed.</mark>

The model is part of the route.

It is not the route.

In the next part, I will make the scoring problem explicit: how should a router trade quality against latency, cost, reliability, and risk when no single model wins on every dimension?

### Sources and scope

- [Uber — Navigating the LLM Landscape: Uber’s Innovation with GenAI Gateway](https://www.uber.com/en-CO/blog/genai-gateway/), describing Uber’s multi-provider gateway, security review, PII redaction, cost attribution, and production usage.
- [Uber — Solving the Identity Crisis for AI Agents](https://www.uber.com/gb/en/blog/solving-the-agent-identity-crisis/), describing the separation between AI Gateway, MCP Gateway, AI Guard, and scoped agent identity.
- [Google Cloud — How GKE Inference Gateway improved latency for Vertex AI](https://cloud.google.com/blog/products/containers-kubernetes/how-gke-inference-gateway-improved-latency-for-vertex-ai), reporting production results for load-aware and prefix-aware routing.
- [Vercel — AI Gateway Production Index](https://vercel.com/blog/ai-gateway-production-index), reporting aggregate production traffic across more than 200,000 teams.
- [Microsoft — How model router works in Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/openai/concepts/model-router-how-it-works), describing routing modes, model subsets, failover, and hybrid direct deployments.
- [AWS — Multi-Provider Generative AI Gateway reference architecture](https://aws.amazon.com/blogs/machine-learning/streamline-ai-operations-with-the-multi-provider-generative-ai-gateway-reference-architecture/), describing governance, resilience, cost, and observability capabilities.
- [Anthropic — Building effective agents](https://www.anthropic.com/engineering/building-effective-agents), for the workflow, routing, and complexity trade-offs.

The production figures above are reported by the cited organisations. The architecture sequence and failure-policy recommendations are my synthesis for a production LLM Gateway, not claims that any one company implements every step exactly this way.
