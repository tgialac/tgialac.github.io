---
title: "The LLM Gateway Problem, Part I: When One Model Call Becomes a Production System"
date: 2026-08-23
lastmod: 2026-08-23
description: "Why production AI applications need an LLM gateway: lessons from Vercel's production traffic and the first principles behind a fintech control plane."
summary: "A production-grounded introduction to LLM gateways, routing, fallback, cost, and why the model should not be the system boundary."
tags: ["LLM Gateway", "AI Infrastructure", "Production AI", "Fintech"]
draft: false
---

## 1. Introduction

The usual way to start an AI feature is deceptively simple:

```python
response = client.responses.create(
    model="the-best-model",
    input=user_message,
)
```

This is a reasonable starting point. It is also a dangerous place to stop.

The first production signal that changed how I think about this problem came from [Vercel's AI Gateway Production Index](https://vercel.com/blog/ai-gateway-production-index), published in May 2026. The report is based on seven months of anonymized, aggregate traffic through Vercel's gateway: **more than 200,000 teams, tens of trillions of tokens, and hundreds of models used by real applications and agents**.

The most important number is not a leaderboard position. It is this:

> **“Roughly 3.5% of requests on AI Gateway complete after a fallback.”**

That sentence describes requests whose first route encountered an error, rate limit, or timeout, then succeeded after the gateway sent them to a healthy alternative. The result is not a benchmark score. It is production traffic that would otherwise have failed, timed out, or surfaced a provider error to the user.

The number becomes more serious when weighted by what those requests contain. Vercel reports that fallback rescued **5.1% of tokens** and **4.9% of market cost**, higher than the 3.5% request rate. Long-context requests, multi-step agent runs, and heavy reasoning calls are more likely to be large, expensive, and exposed to rate limits or timeouts. The requests that need rescuing are not evenly distributed across the workload; they cluster around the calls that already carry the most computational and financial weight.

This is the real case I want to begin with. A gateway did not merely make the API nicer to call. It changed the outcome of a non-trivial fraction of production work by deciding **what to do after the first model path stopped being reliable**.

### The first assumption to discard

When a team says, “We use model X,” it often means one of three different things:

- the application has a model name in its code;
- the model is available through one provider endpoint;
- the model is the best choice for every request.

These are not the same statement.

Vercel's data shows why. At the highest traffic tier, teams used **an average of 35 models in regular use**. Tool-using requests represented only 22.2% of requests in April 2026, but accounted for 58.9% of token volume. Production AI is therefore not shaped like a chat box making one isolated completion. It is shaped like a routing graph: a fast model for classification, a stronger model for reasoning, an embedding model for retrieval, a vision model for images, and one or more fallbacks when capacity or availability changes.

The model is becoming a replaceable compute primitive. **The policy that decides which primitive to use is becoming the durable part of the application.**

### A request is already a small distributed system

Consider a design scenario rather than a reported MoMo incident: a customer asks a Vietnamese digital wallet, “Why is my transfer still pending?”

The user sees one question. The application may need to:

1. authenticate the user and establish the tenant or product context;
2. classify whether the request is a general FAQ or a request about live account state;
3. retrieve the relevant transaction record through an authorized service;
4. redact or minimize sensitive data before it enters a model context;
5. select a model that can reason over the evidence within the latency budget;
6. call a provider with the right quota, region, and data policy;
7. retry or fall back if that route returns a timeout or rate limit;
8. check that the final answer is grounded in the transaction data;
9. record the model, provider, token usage, cost, policy version, and outcome.

The question is one line. The execution is not.

```text
User request
     |
     v
Identity -> Data policy -> Intent / capability routing
                                  |
                                  v
                       Model + provider selection
                                  |
                                  v
                         Inference + tool calls
                                  |
                    +-------------+-------------+
                    |                           |
                 Success                    Failure
                    |                           |
                    v                           v
              Validate output          Retry / fallback
                    |                           |
                    +-------------+-------------+
                                  |
                                  v
                         Response + audit trace
```

This diagram hides an important asymmetry: the application wants to think in terms of user intent, while the provider thinks in terms of an API request. Something must translate between those two levels. If every application implements that translation independently, routing logic, secrets, quotas, retries, redaction, and audit behavior become scattered across the organization.

That is the problem an LLM Gateway is meant to solve.

### A gateway is not just a reverse proxy

A traditional reverse proxy can forward traffic, terminate TLS, and perhaps perform load balancing. Those capabilities are still useful. They are not enough for model-serving systems where the request itself carries semantic, financial, and compliance consequences.

An LLM Gateway needs to answer questions that a normal HTTP proxy does not understand:

| Question | Direct provider call | Gateway-mediated call |
| --- | --- | --- |
| Which model should handle this request? | Hard-coded by the application | Selected by capability and policy |
| What happens after a timeout? | Usually application-specific | Centralized retry and fallback policy |
| Who is consuming the quota? | Difficult to aggregate | Tracked by user, team, tenant, and model |
| Can this data leave the approved boundary? | Depends on every caller | Enforced at a common control point |
| How much did this workflow cost? | Scattered provider logs | Unified usage and cost trace |
| Can we change providers without rewriting clients? | Often no | Yes, through an abstraction layer |

AWS described the same architectural pressure in its [Generative AI Gateway guidance](https://aws.amazon.com/blogs/machine-learning/create-a-generative-ai-gateway-to-allow-secure-and-compliant-consumption-of-foundation-models/): enterprises need a model abstraction layer, centralized governance, observability, and a way to keep application clients loosely coupled from rapidly changing inference endpoints. In other words, the gateway is both an **execution path** and a **management boundary**.

The distinction matters because a gateway can create a new failure mode if it becomes only another hop. Adding a proxy that forwards requests without owning policy, routing, or observability adds latency and operational complexity without adding control. The gateway earns its place when it makes decisions that applications should not have to duplicate.

### What production traffic says about “the best model”

The phrase “best model” hides the objective function.

For a low-stakes FAQ, the best model may be the cheapest model that answers within 300 milliseconds. For a transaction explanation, the best model may be the one with stronger instruction following and a stricter grounding check. For a sensitive workflow, the best model may be the only approved deployment in a particular region. For an agent that performs several tool calls, the best model may be the one that reduces the probability of an invalid action, even if its per-token price is higher.

Vercel's production data makes this fragmentation visible. In April 2026, Google led token volume while Anthropic led spend. The same customer base appeared on both sides because different models occupied different layers of the workload: cheap, fast models carried high-volume calls, while expensive models handled quality-critical work.

This suggests a more useful formulation:

<p class="concept-equation">best_model(request) = argmax quality(request) - cost(request) - latency(request) - risk(request)</p>

The exact formula will differ by product. The architectural consequence does not: **model choice is a runtime decision under constraints, not a permanent import statement.**

### Fallback is not an edge case

It is tempting to treat fallback as an emergency branch that can be added after the first version. The production evidence argues for the opposite order.

Vercel's 3.5% request-level rescue rate does not mean that every gateway should blindly retry every failure. A fallback can change model behavior, tool-call formatting, latency, cost, and even the interpretation of a prompt. Retrying a financial action without idempotency can be worse than returning an error. Sending sensitive data to an unapproved provider can turn an availability fix into a compliance incident.

The correct lesson is narrower and more useful:

**If the application depends on one provider path, provider availability is part of the application's availability.**

The gateway is where a team can encode the conditions under which a fallback is safe:

- a read-only request may move to a secondary provider;
- a request containing restricted data may use only an approved private deployment;
- a structured tool call may require schema compatibility before retrying;
- a payment mutation may stop and request human approval rather than replaying automatically;
- a quality-sensitive task may fall back only to a model that passes a minimum capability threshold.

AWS's [resilience patterns for Amazon Bedrock and LLM gateways](https://aws.amazon.com/blogs/machine-learning/implementing-resilience-patterns-with-amazon-bedrock-and-llm-gateway/) demonstrate this mechanics explicitly: a primary model with a restrictive three-requests-per-minute limit serves the first three of ten concurrent requests, while a fallback with higher capacity handles the remaining seven. The demonstration is a reproducible capacity test rather than an anonymized production statistic, but it captures the operational boundary clearly: **the client does not need to know that the first route ran out of capacity.**

### The thesis

The central claim of this series is simple:

> **An LLM Gateway is the control plane for production AI applications.**

It decides which model is allowed to see a request, which provider is healthy enough to receive it, how much the request may cost, what data may cross a boundary, when a failure can be recovered, and what evidence must be retained afterward.

The model generates the answer. **The gateway determines the conditions under which that answer is allowed to exist.**

That is especially important in fintech. A gateway for a consumer writing assistant and a gateway for a financial support agent may share protocol adapters, metrics, and routing primitives. They should not share the same data policy, authorization boundary, fallback rules, or definition of an acceptable failure.

The project I will build around this thesis is **FinGuard Gateway**: a policy-aware, cost-aware, and failure-resilient control plane for Vietnamese fintech agents. It will not begin by pretending that one model can solve every task. It will begin by making the system's decisions explicit and measurable.

The next parts will derive that system from first principles:

- the mental model of model, provider, deployment, gateway, control plane, and data plane;
- the lifecycle of a request from authentication to audit;
- routing across quality, latency, cost, capability, and risk;
- failure handling, security, governance, and evaluation;
- a concrete implementation and benchmark for FinGuard.

For now, the most important conclusion is the one production traffic has already made difficult to ignore:

**The moment an AI application has more than one meaningful model path, the routing layer stops being infrastructure around the application. It becomes part of the application itself.**

### Sources and scope

- [Vercel — AI Gateway production index](https://vercel.com/blog/ai-gateway-production-index), published May 12, 2026. Its figures are anonymized aggregate routing data through April 2026; they do not identify any individual team or workload.
- [AWS — Create a Generative AI Gateway](https://aws.amazon.com/blogs/machine-learning/create-a-generative-ai-gateway-to-allow-secure-and-compliant-consumption-of-foundation-models/), an enterprise architecture and governance reference.
- [AWS — Implementing resilience patterns with Amazon Bedrock and LLM gateway](https://aws.amazon.com/blogs/machine-learning/implementing-resilience-patterns-with-amazon-bedrock-and-llm-gateway/), including a reproducible model-fallback demonstration.
