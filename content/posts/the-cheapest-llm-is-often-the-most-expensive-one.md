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

That same gateway can make the decision more intelligent than “retry anywhere”:

- a read-only request may move to a secondary provider;
- a request containing restricted data may use only an approved deployment;
- a structured tool call may require schema compatibility before retrying;
- a payment mutation may stop rather than replay automatically;
- a high-risk support request may require a stronger model and a human checkpoint.

Fallback is therefore not merely an availability feature. It is a policy decision.

### The hidden cost of being slightly wrong

The cost of a model error is rarely confined to the response itself. A low-quality first answer can create:

- another model invocation;
- another retrieval pass with a longer context;
- an invalid tool call and a repair loop;
- an unnecessary escalation to customer support;
- a delayed response that violates the product's latency target;
- a business decision made from an answer that sounded confident but was not grounded.

This is why OpenAI's guidance for enterprise AI recommends evaluating the full cost of reaching an accepted outcome, including model and tool usage, retries, completion rate, latency, and human review. The same guidance treats model routing, observability, evaluations, and reusable agent patterns as shared capabilities rather than application-specific glue.

The practical consequence is uncomfortable but useful: **model selection is no longer a configuration detail**. It is part of the application's runtime behaviour.

### A request is already a distributed system

Consider a Vietnamese digital-wallet user asking:

> “Why is my transfer still pending?”

The user sees one sentence. A production system may need to:

1. authenticate the user and establish the correct tenant or product context;
2. classify whether this is a general FAQ or a request about live account state;
3. retrieve the relevant transaction through an authorised service;
4. minimise or redact sensitive data before it enters a model context;
5. select a model that can explain the evidence within the latency budget;
6. call a provider with available quota and an approved data policy;
7. retry or fall back if that route times out or is rate-limited;
8. verify that the final answer is grounded in the transaction state;
9. record the provider, model, tokens, cost, policy version, and outcome.

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

A traditional reverse proxy forwards traffic. An LLM Gateway must also understand the request well enough to decide:

- which model is allowed to see it;
- which provider is healthy enough to receive it;
- how much it may cost;
- whether it can be retried safely;
- whether the result meets the quality bar;
- what evidence must be retained afterward.

[Anthropic's work on effective agents](https://www.anthropic.com/engineering/building-effective-agents) makes a related point from the application side: start with the simplest workflow that solves the problem, and add agentic complexity only when its benefit justifies the added latency and cost. The gateway is where that discipline becomes operational. It can route a simple FAQ to a fast model, reserve stronger reasoning for ambiguous cases, and stop a high-risk request from being “optimised” into an unsafe cheap path.

The gateway earns its place when it makes decisions that individual applications should not have to duplicate.

### The thesis

<mark>An LLM Gateway is the control plane for production AI applications.</mark>

It decides which model is allowed to see a request, which provider is healthy enough to receive it, how much the request may cost, what data may cross a boundary, when a failure can be recovered, and what evidence must be retained afterward.

The model generates the answer. The gateway determines the conditions under which that answer is allowed to exist.

In the next parts, I will build this idea from first principles:

- the difference between a model, provider, deployment, gateway, control plane, and data plane;
- routing across quality, latency, cost, capability, risk, and availability;
- fallbacks, retries, quotas, and failure containment;
- evaluation of routing decisions rather than only final answers;
- and a concrete model router for Vietnamese fintech customer support.

For now, the first conclusion is enough:

> **The cheapest LLM call is often the most expensive path to a correct answer.**

### Sources and scope

- [Vercel — AI Gateway Production Index](https://vercel.com/blog/ai-gateway-production-index), based on anonymised aggregate AI Gateway traffic through April 2026.
- [AWS — Implementing resilience patterns with Amazon Bedrock and LLM gateway](https://aws.amazon.com/blogs/machine-learning/implementing-resilience-patterns-with-amazon-bedrock-and-llm-gateway/), including the explicitly documented ten-request fallback demonstration.
- [OpenAI — How to manage AI investments in the agentic era](https://openai.com/index/managing-ai-investments-in-agentic-era/), for cost-per-outcome and shared production capabilities.
- [Anthropic — Building effective agents](https://www.anthropic.com/engineering/building-effective-agents), for the workflow-versus-agent trade-off.
