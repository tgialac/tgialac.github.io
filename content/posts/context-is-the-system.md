---
title: "Context Is the System: From RAG Pipeline to Production Agent"
date: 2026-07-31T10:00:00+07:00
tags: ["ai", "agents", "rag", "architecture", "context-engineering", "production"]
math: true
summary: "A production agent is not a model with more tools. It is a context system: an architecture that assembles the right evidence, authority, state, and feedback for every decision."
cover:
  image: "/images/covers/context-is-the-system.png"
  alt: "A luminous reasoning loop connected through a context fabric to data, tools, messages, human approval, and evidence"
---

A language model can know more than anyone in your company and still understand less about the decision in front of it.

It does not know which policy is current, which customer record it may access, which failed attempt has already been tried, whether the user wants a draft or an irreversible action, or why the same request was rejected yesterday. None of that is primarily a model problem. It is a context problem.

This is the uncomfortable lesson behind the move from retrieval-augmented generation to agents. A prototype can retrieve a few passages, call a tool, and produce an impressive answer. A production system must continuously assemble the right evidence, memory, permissions, tools, state, and human judgment around every step of a task.

> **The model supplies reasoning. The context system makes that reasoning situationally correct, operationally useful, and safe to act on.**

Seen this way, RAG is not an accessory bolted onto an agent. It is the beginning of the agent's context plane. And an agent is not simply “RAG plus tool use.” It is a controlled loop that acquires context, decides, acts, observes the environment, and revises its state until it reaches a stopping condition.

That distinction connects five useful perspectives: OpenAI's foundations for models, tools, instructions, orchestration, and guardrails; Anthropic's progression from composable workflows to autonomous loops; Jerry Liu's account of making RAG measurable and production-ready; Douwe Kiela's argument that enterprise value comes from systems and specialized context rather than models alone; and LinkedIn's description of what happens when agents meet messaging, memory, observability, identity, and distributed systems at scale.

## Begin with an architecture decision, not an agent

The word *agent* hides several different systems.

Anthropic draws the cleanest boundary: in a **workflow**, code determines the path through models and tools; in an **agent**, the model dynamically decides how the task should proceed. OpenAI uses a compatible operational definition: an agent uses a model to manage workflow execution, selects tools according to the current state, recognizes completion, corrects itself, and can return control to a person. [Anthropic](https://www.anthropic.com/engineering/building-effective-agents) · [OpenAI](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/)

The distinction matters because autonomy is not free. It buys flexibility by spending predictability, latency, tokens, and a larger failure surface.

| Pattern | Who chooses the next step? | Best fit | Primary failure mode |
| --- | --- | --- | --- |
| **One model call** | Application code | A bounded transformation with enough context | Prompt or context is insufficient |
| **Prompt chain** | Fixed sequence | A task that decomposes into stable stages | An early error contaminates later stages |
| **Router** | Classifier, then fixed branch | Inputs fall into distinct, testable categories | Misrouting |
| **Parallel workers** | Fixed fan-out and aggregation | Independent analysis or multiple votes | Inconsistent aggregation or wasted work |
| **Evaluator–optimizer** | Generator and critic loop | Quality criteria are explicit and iteration helps | Endless refinement or judge bias |
| **Agent loop** | Model, within bounded controls | Steps cannot be known in advance | Compounding decisions and unsafe actions |
| **Multi-agent system** | Manager or peer handoffs | Roles and context boundaries are genuinely distinct | Coordination, state, and ownership failures |

This is an escalation ladder, not a maturity model. A router is not less advanced than an agent when routing solves the problem more reliably. A single model call with excellent retrieval can be a better product than a network of agents with vague roles.

Anthropic's advice is to add complexity only when measurement shows that it improves the result. OpenAI similarly recommends maximizing a single agent before splitting it, and points to two practical signals for considering multiple agents: instructions have become dominated by conditional branches, or overlapping tools remain confusing even after their names, parameters, and descriptions have been improved. [Anthropic](https://www.anthropic.com/engineering/building-effective-agents) · [OpenAI](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/)

The first design question is therefore not “Which agent framework?” It is:

> **Where does uncertainty actually require model judgment, and where can software keep control?**

Keep deterministic structure everywhere it remains valid. Spend autonomy only on the parts of the work that cannot be usefully hard-coded.

## RAG is a context compiler, not a vector search feature

Naive RAG is usually described as a three-step pipeline: ingest documents, retrieve the top-*k* chunks, and give them to a model for synthesis. It is a powerful baseline—and an incomplete production architecture.

In his talk on production-ready RAG, Jerry Liu separates two failure domains that teams often blur together:

1. **Retrieval can fail.** Relevant evidence is missing, irrelevant chunks dominate the result, information is stale, or the useful passage is buried in too much context.
2. **Synthesis can fail.** The evidence is present, but the model produces an unsupported, incomplete, irrelevant, or poorly structured answer.

Those failures require different measurements and different remedies. Retrieval should be evaluated as retrieval, using a query set with relevant document or chunk identifiers and ranking metrics such as hit rate, mean reciprocal rank, or NDCG. End-to-end responses need their own groundedness, correctness, relevance, and task-specific evaluation. You cannot tune the pipeline intelligently until you know which component is failing. [Jerry Liu, 5:21–8:25](https://www.youtube.com/watch?v=TRjq7t2Ms5I&t=321s)

This reframes retrieval as compilation. The system is compiling a large, messy world into a small decision context for the current step:

$$
C_t = f(G, S_t, I, P, M, E, T)
$$

where $G$ is the goal, $S_t$ is current task state, $I$ is identity, $P$ is policy, $M$ is memory, $E$ is retrieved evidence, and $T$ is the available tool surface. The output $C_t$ is not “all potentially useful text.” It is the smallest context that lets the system make the next decision correctly.

That compiler has several levers.

### Retrieve narrowly, synthesize broadly

Liu describes a useful **small-to-big** pattern: index and retrieve compact, discriminative units, then expand to a parent passage or surrounding window for synthesis. Small units can improve retrieval precision; larger parent context gives the model enough information to reason without flooding the prompt with unrelated chunks. He also describes embedding summaries or questions that a chunk can answer, while returning the underlying source content for generation. [Jerry Liu, 12:31–14:33](https://www.youtube.com/watch?v=TRjq7t2Ms5I&t=751s)

This is a general context-engineering principle: the best representation for *finding* evidence does not have to be the representation used for *reasoning* over it.

### Combine semantic and structured constraints

If a user asks for risk factors in a 2021 filing, semantic similarity alone may retrieve the right topic from the wrong year. Metadata turns implicit constraints into explicit filters: year, document type, account, region, access class, version, owner, or freshness. Retrieval can then combine a structured predicate with semantic ranking. [Jerry Liu, 10:27–12:30](https://www.youtube.com/watch?v=TRjq7t2Ms5I&t=627s)

The same idea becomes essential for agents. Identity and authorization are not passages that should compete for attention inside a prompt. They are executable constraints on what may enter context and which tools may run.

### Retrieve capabilities, not only passages

For questions that require comparing documents, building an analysis, or performing a sequence of lookups, top-*k* text injection eventually reaches its limit. Liu describes modeling a document as a set of capabilities—such as summary and question answering—and retrieving the relevant document tools before the model uses them. [Jerry Liu, 14:34–16:03](https://www.youtube.com/watch?v=TRjq7t2Ms5I&t=874s)

That is the bridge from RAG to an agent: retrieval no longer returns only *what the model should read*. It can also return *what the model is allowed and equipped to do*.

## The production agent has six planes

OpenAI's minimal agent has three foundations: a model, tools, and instructions. That is the correct conceptual core. In production, the surrounding system needs a little more resolution.

| Plane | Responsibility | The question it must answer |
| --- | --- | --- |
| **Experience** | Intent capture, progress, clarification, streaming, review, cancellation | Does the user understand and control what is happening? |
| **Control** | Planning, routing, loop state, budgets, stopping conditions, handoffs | Who chooses the next step, and when must the run stop? |
| **Context** | Retrieval, memory, identity, policy, provenance, context assembly | What does this decision need to know now? |
| **Action** | Tool contracts, credentials, validation, idempotency, reversibility | What may the system do, and how can it do it safely? |
| **Runtime** | Queues, messages, retries, concurrency, persistence, sync/async execution | Can a long-running task survive real infrastructure? |
| **Evidence** | Traces, evaluations, audit logs, quality and business telemetry | Can we explain, measure, and improve the run? |

The model participates mainly in the control plane, but the quality of its decisions is bounded by all six.

LinkedIn's agent platform makes this concrete. Agents expose standardized gRPC contracts and register their capabilities. Long-lived tasks move through an existing messaging system, gaining FIFO delivery, history, parallel threads, persistent retries, and cross-region resilience. An agent lifecycle service hides the message-to-RPC adaptation. Client libraries handle push notifications, cross-device synchronization, streaming, and fallback behavior for work that can outlive a user session. [LinkedIn Engineering](https://www.linkedin.com/blog/engineering/generative-ai/the-linkedin-generative-ai-application-tech-stack-extending-to-build-ai-agents)

This is not incidental plumbing. Once a task lasts minutes or hours, “the conversation” is no longer a string of chat messages. It is a durable distributed process. It needs an identity, state transitions, delivery semantics, retry policy, cancellation behavior, and a way to resume after failure.

A useful run state might contain:

```text
RunState
  goal
  current_plan
  completed_steps
  evidence_with_provenance
  tool_results
  pending_approvals
  budgets { turns, tokens, time, money }
  permissions
  retry_history
  final_status
```

The prompt is only a temporary projection of that state. Durable truth belongs outside the context window.

## Tools are the real agent-computer interface

A tool description is not API documentation pasted into a prompt. It is an interface designed for a stochastic caller.

Anthropic argues that agent-computer interfaces deserve the same care that product teams give human-computer interfaces. Tool names and parameters should make the correct action obvious, examples and edge cases should be explicit, formats should minimize unnecessary escaping or bookkeeping, and argument design should make mistakes difficult. In its coding-agent work, Anthropic found that requiring absolute file paths eliminated a recurring class of tool error. [Anthropic](https://www.anthropic.com/engineering/building-effective-agents)

OpenAI divides tools into three useful classes:

- **Data tools** retrieve context.
- **Action tools** change an external system.
- **Orchestration tools** expose other agents as callable capabilities.

The classification should affect the contract. A search tool may tolerate a retry. A payment tool needs idempotency, a bounded amount, explicit account identity, a confirmation artifact, and a human approval path. A handoff tool needs a clear transfer of ownership and state.

For every action tool, specify at least:

- preconditions and authorization scope;
- a typed input and output schema;
- side effects and whether they are reversible;
- idempotency or duplicate-call behavior;
- expected errors and retry rules;
- evidence returned after execution;
- the conditions that require approval.

This is where safety becomes architectural. A prompt can request caution; a tool contract can prevent an unauthorized transfer.

## Guardrails should scale with authority

Content filters are necessary, but the most consequential agent risk begins when a model can act.

OpenAI recommends layered guardrails: relevance and safety classifiers, PII filtering, moderation, rules-based checks, output validation, and tool safeguards. It also suggests rating tools by properties such as read versus write access, reversibility, permissions, and financial impact, then pausing or escalating before high-risk actions. Guardrails must sit beside—not replace—authentication, authorization, strict access control, and ordinary software security. [OpenAI](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/)

A practical authority ladder looks like this:

| Authority | Example | Default control |
| --- | --- | --- |
| **Read** | Search a permitted knowledge base | Scoped identity, provenance, privacy filtering |
| **Draft** | Prepare an email or record update | Human review before publication |
| **Reversible write** | Update a ticket status | Validation, audit log, undo path |
| **External communication** | Send a customer message | Policy checks and approval by risk tier |
| **Irreversible or financial action** | Cancel an order or issue a large refund | Explicit human authorization and transaction limits |

Human intervention is not one generic “approve” button. It should occur when the run exceeds a failure or retry threshold, when requested authority crosses a risk boundary, or when the system lacks required information. The handoff should include the goal, evidence, attempted actions, uncertainty, and proposed next step. That makes the person a decision-maker, not a cleanup crew.

LinkedIn applies the same principle to context itself: conversation memory, experiential memory, and other stores are siloed, and cross-component access occurs through explicit policy-governed interfaces with authentication, authorization, logging, and auditability. [LinkedIn Engineering](https://www.linkedin.com/blog/engineering/generative-ai/the-linkedin-generative-ai-application-tech-stack-extending-to-build-ai-agents)

## Observability is how an agent learns to become a product

An agent combines two difficult things: a distributed system and a non-deterministic decision-maker. Traditional service metrics are necessary but insufficient. “Request succeeded” does not tell you whether the agent used the wrong source, called the right tool with the wrong parameter, retried wastefully, or reached a correct answer through an unsafe path.

LinkedIn uses different trace strategies for different environments. In pre-production, developers capture rich execution context—model calls, tools, and control flow—for diagnosis and iteration. In production, privacy-safe structured spans capture key lifecycle events and correlate the agent with upstream and downstream services. Those traces are then aggregated into datasets for offline evaluation, regression testing, and prompt experiments. [LinkedIn Engineering](https://www.linkedin.com/blog/engineering/generative-ai/the-linkedin-generative-ai-application-tech-stack-extending-to-build-ai-agents)

That closes a vital loop:

```text
production run
  -> structured trace
  -> classified failure
  -> evaluation case
  -> targeted change
  -> regression test
  -> safer deployment
```

The evaluation stack should mirror the architecture:

1. **Retrieval:** Did the right evidence appear, with sufficient precision and recall?
2. **Context:** Was identity, policy, memory, and provenance assembled correctly?
3. **Trajectory:** Were the plan, tool choice, arguments, retries, and handoffs appropriate?
4. **Outcome:** Was the final answer or action correct, complete, and useful?
5. **Operations:** Did the run meet latency, cost, reliability, and privacy constraints?
6. **Adoption:** Did it fit the real workflow closely enough that people used it?

Douwe Kiela's production lessons sharpen the last two points. He argues that models may be only a small part of the enterprise system; specialized institutional context is the fuel for differentiated value. Pilots are easy, while production introduces scale, noisy data, security, compliance, and many use cases. Teams should design for production from the start, iterate with real users early, integrate into workflows people already use, and treat attribution and audit trails as the way to manage the inevitable residual error. [Douwe Kiela, 4:23–10:28](https://www.youtube.com/watch?v=kPL-6-9MVyA&t=263s) · [13:01–14:30](https://www.youtube.com/watch?v=kPL-6-9MVyA&t=781s)

Accuracy is the entry ticket. The product question is what the system does when it is inaccurate.

## A production sequence that preserves learning

The sources converge on an incremental path, but “start small” should describe architectural complexity—not business ambition. Kiela warns that low-value assistants can succeed technically and still produce no meaningful return. Choose an important problem, then solve it with the smallest system that exposes the real constraints. [Douwe Kiela, 14:31–16:34](https://www.youtube.com/watch?v=kPL-6-9MVyA&t=871s)

### 1. Write the decision boundary

Define the user, goal, source of truth, permitted actions, success condition, and escalation rule. Separate what software can decide deterministically from what genuinely needs model judgment.

### 2. Establish the simplest measurable baseline

Try a strong model with retrieval and clear instructions before adding loops. Build an evaluation set from real tasks, expert labels, user feedback, or carefully reviewed synthetic cases. Measure retrieval and final answers separately.

### 3. Design context as a product surface

Version sources, attach provenance, enforce identity before retrieval, combine semantic search with structured filters, and keep durable state outside the prompt. Give users a way to inspect sources and correct memory.

### 4. Engineer the tools

Make names and parameters unambiguous. Separate read, draft, and action capabilities. Add validation, idempotency, scoped credentials, risk tiers, and evidence-bearing results.

### 5. Add only the orchestration the task earns

Use chaining, routing, parallelization, or an evaluator loop when the path is structurally known. Add an agent loop when the number and order of steps cannot be known in advance. Split into multiple agents only when distinct roles improve measured performance or reduce context and tool confusion.

### 6. Make the run durable and interruptible

Persist task state. Define turn, time, token, and action budgets. Support cancellation, resume, retries, and human checkpoints. Choose synchronous execution for interactive latency and asynchronous execution for long-running work rather than forcing every task through chat request-response semantics.

### 7. Turn traces into the learning system

Inspect early runs in full. Convert failures into named categories and regression cases. In production, collect privacy-safe operational spans and preserve enough provenance to reconstruct consequential actions.

### 8. Integrate where work already happens

An accurate agent in an unused interface creates no value. Put it inside the ticketing, recruiting, engineering, or research workflow; notify people when background work completes; and make review faster than doing the work again.

## The deeper transition

The journey from RAG to agents is often presented as an increase in autonomy:

```text
retrieve -> generate -> use tools -> plan -> act
```

The more important transition is an increase in **context discipline**:

```text
documents
  -> evidence with provenance
  -> state with memory
  -> capabilities with authority
  -> actions with feedback
  -> traces that improve the next run
```

This explains why the best production advice sounds less like prompt engineering and more like systems engineering. Context must be retrieved and filtered. State must be durable. Messages must be delivered. Tools must be typed and permissioned. Actions must be reversible or approved. Traces must become evaluations. Users must retain control.

A better model can improve reasoning inside the loop. It cannot decide which company policy is current, create an audit trail that was never captured, recover task state that was never persisted, or undo authority that was granted too broadly.

The production agent is therefore not the model at the center of the diagram. It is the complete context system around it.

Build that system well, and autonomy becomes useful. Build only the model loop, and autonomy merely makes the prototype fail on its own.

## Sources

- [OpenAI: *A practical guide to building agents*](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/)
- [Anthropic: *Building effective agents*](https://www.anthropic.com/engineering/building-effective-agents)
- [Jerry Liu: *Building Production-Ready RAG Applications* (video)](https://www.youtube.com/watch?v=TRjq7t2Ms5I)
- [Douwe Kiela: *RAG Agents in Prod: 10 Lessons We Learned* (video)](https://www.youtube.com/watch?v=kPL-6-9MVyA)
- [LinkedIn Engineering: *The LinkedIn Generative AI Application Tech Stack: Extending to Build AI Agents*](https://www.linkedin.com/blog/engineering/generative-ai/the-linkedin-generative-ai-application-tech-stack-extending-to-build-ai-agents)

*The two video sources were reviewed from their complete English auto-generated transcripts; timestamps link back to the relevant passages for verification.*
