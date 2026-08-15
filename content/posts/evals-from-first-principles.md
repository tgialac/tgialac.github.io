---
title: "Evals from First Principles: How to Measure, Debug, and Improve AI Agents"
date: 2026-08-10
lastmod: 2026-08-15
description: "A practical guide to evaluating AI agents through traces, targeted graders, repeated trials, release gates, and production feedback loops."
summary: "How to turn agent failures into observable contracts, reliable evals, regression tests, and safer release decisions."
tags: ["AI Agents", "LLM Evals", "Evaluation", "Observability"]
cover:
  image: "/images/illustrations/eval-task-dataset-scorer.png"
  alt: "An evaluation decomposed into a task, dataset, and scorer."
  hiddenInSingle: true
draft: false
---

## 1. Introduction

In July 2025, SaaStr founder Jason Lemkin was nine days into building an application with Replit Agent when its batch processing stopped working. He asked the agent what had happened. The answer eventually exposed something much worse than a broken job: during an explicit **code freeze**, the agent had deleted the application's production database. The deleted data included records for 1,206 executives and more than 1,196 companies. Lemkin [documented the incident publicly](https://x.com/jasonlk/status/1946069562723897802) as it unfolded.

The agent had not merely missed an implied preference. Lemkin had repeatedly instructed it not to make changes without permission. In the account it produced after the deletion, the agent acknowledged that it had ignored those instructions and destroyed live data anyway. It had enough context to restate the boundary after crossing it, but not enough control to respect the boundary before acting. [Contemporary reporting preserved the exchange](https://www.tomshardware.com/tech-industry/artificial-intelligence/ai-coding-platform-goes-rogue-during-code-freeze-and-deletes-entire-company-database-replit-ceo-apologizes-after-ai-engine-says-it-made-a-catastrophic-error-in-judgment-and-destroyed-all-production-data), including the agent's own description of the sequence.

That description is not a **root-cause analysis**. A model explaining its previous behavior is still generating text, not exposing a reliable record of its internal causes. The confession sounds precise and self-aware, but it does not tell me which instruction lost priority, why the destructive action remained available, what context the agent saw at that step, or whether another run would make the same decision.

Replit CEO Amjad Masad later confirmed that an agent in development had deleted production data and called the outcome unacceptable. Replit responded with product changes, including automatic separation between development and production databases, staging environments, and restore capabilities. The response matters because it moved beyond asking the model to be more careful. It changed the system around the model so that the same class of failure would become harder to repeat. Masad [described those measures publicly](https://x.com/amasad/status/1946986468586721478), and Replit later [documented the development/production database separation](https://replit.com/blog/introducing-a-safer-way-to-vibe-code-with-replit-databases) in its product.

The database deletion is dramatic, but it is only one point in a much larger **failure surface**:

- **Tool selection:** the wrong tool, or the right tool with the wrong arguments.
- **Retrieval and synthesis:** good evidence retrieved, then synthesized badly.
- **Cost and latency:** the correct answer delivered at an absurd cost or after an unreasonable delay.
- **Skill routing:** a relevant skill never triggers, or an irrelevant one triggers instead.
- **Control flow:** the agent ignores a tool result, trusts stale memory, stops too early, or retries forever.
- **Safety:** a policy is violated midway through an otherwise valid trajectory, or a good final answer is reached through a process that should never be allowed in production.

Rerunning the same task may not reproduce the same mistake. Another trajectory might respect the code freeze. It might inspect the database without modifying it, ask for permission, stop when it encounters uncertainty, or fail in an entirely different way. The behavior I need to understand is not a single output. It is a **distribution of possible trajectories**, including the rare ones that are costly, unsafe, or simply strange.

I would never ship a database migration because I tried it a few times and it looked fine. I would want tests for the expected path, the boundary conditions, and the failures I had already encountered. I would want logs that tell me which branch ran, inputs that let me reproduce the bug, and regression tests that prevent it from quietly returning. **Deterministic software** can still surprise me, but software engineering has spent decades building practices that turn those surprises into inspectable problems.

With LLM applications, that standard often disappears. I type a few prompts into a chat window, read the responses, change a sentence in the system prompt, and try again. If the outputs feel better, I call the change an improvement. If five examples pass, I start to believe the system works. This is **vibe checking**, and for many LLM applications it is still the entire test suite.

Vibe checking is useful while exploring an idea. It is fast, intuitive, and often the first way I notice that something is wrong. But it cannot tell me how often the agent violates a constraint, which step caused the violation, whether a prompt change fixed the underlying failure or merely moved it, or what else regressed while I was watching my favorite examples. A convincing demo is evidence that a system *can* work. It is not evidence that the system works reliably.

An AI agent should be treated as a **nondeterministic software system**. The goal of **evals** is not merely to assign it a score, but to turn failures into observable, reproducible, and eventually preventable behavior.

That changes the question. I am no longer asking only whether the final answer looks good. I am asking what the agent saw, which decisions it made, which tools it called, what those calls cost, where the trajectory diverged, and whether the same class of failure will be caught the next time it appears.

**If an agent can look competent across a long session and then erase production data in seconds, what evidence would justify saying that it “works”?**

## 2. What Exactly Are We Evaluating?

Before I define an evaluation, I need to define the thing being evaluated.

At the simplest level, an **application** is a system that receives an input and produces an output. For a traditional function, that description may be nearly complete. The input determines which code runs, and the code determines the output. If the function is pure and the environment is fixed, the same input should produce the same result.

An AI application has a larger boundary. The model is one component inside it, not the application itself. Between input and output, the system may make several model calls, retrieve documents, select skills, invoke tools, read and update state, apply policies, retry failed steps, and decide when to stop. The orchestration code that connects those operations is part of the behavior too.

<figure class="article-figure article-figure-plain">
  <img src="/images/illustrations/agent-application-boundary.png" width="1176" height="426" alt="An input flows into an agent and then to an output, while the agent interacts with model calls, retrieval, tools, skills, and state.">
</figure>

This boundary matters. Suppose I replace the model but keep the prompt, tools, and retrieval pipeline fixed. That is one application change. Suppose I keep the model but change a tool schema, add a skill, update the search index, or shorten the context window. Those are application changes too. Any of them can alter the final answer, the path taken to reach it, or both.

For an agent, I can observe behavior at three levels:

- **Outcome:** What answer or action did the application ultimately produce?
- **Trajectory:** Which intermediate decisions, model calls, retrieval results, tool calls, and state transitions led to that outcome?
- **Operational behavior:** How much time, money, and computation did the run consume, and how many retries or external actions did it require?

These levels are related, but they are not interchangeable. Two runs can return the same correct answer while following very different trajectories. One may use a single retrieval call and cite the right evidence. The other may call five unnecessary tools, expose sensitive data along the way, and cost one hundred times more. An output-only view would call them equivalent. A system-level view would not.

The reverse is also possible. An agent may follow a reasonable trajectory and still produce a weak final answer. Good retrieval does not guarantee good synthesis. The correct tool does not guarantee correct arguments. A policy-compliant process does not guarantee a useful response. Evaluating the application therefore begins by deciding which behavior I care about and at which level that behavior becomes observable.

### From Desired Behavior to a Test

With the system boundary in place, the core idea becomes simple:

**Evaluation is simply the process of turning some desired behavior of this system into a measurable test.**

Each part of that sentence carries weight.

**Desired behavior** is a claim about how the application should act. It might be "answer using only the supplied evidence," "never mutate production data during a code freeze," "select the refund tool only after verifying the order," or "complete the task for less than ten cents."

**Measurable** means that the behavior leaves an observable signal. If I care about whether the agent respected a code freeze, the final message is not enough. I need to observe attempted writes, tool calls, database operations, or some other trace of action. If the system records only the final answer, then a confident sentence such as "No changes were made" can hide a destructive trajectory.

A **test** combines a situation with a decision rule. Given this input and system state, run the application, collect the relevant observations, and decide whether the desired behavior occurred. The decision may be binary, graded, or multidimensional, but it must connect an observed run to an explicit definition of success.

Consider the Replit incident from the introduction. A useful test would not ask whether the agent apologized correctly after deleting the database. It would encode the actual requirement:

- **Desired behavior:** no production mutation during an active code freeze.
- **Observable signal:** any attempted or completed mutating operation against production.
- **Decision rule:** fail the run if such an operation occurs, regardless of the final response.

That test turns a vague instruction into a behavior I can detect. Once the failure is observable, I can preserve the case, rerun it against a changed system, and prevent a regression. This is the first important move in eval design: from preference to specification, and from specification to evidence.

### A Useful Mental Model

[Braintrust describes an evaluation](https://www.braintrust.dev/docs/evaluation-quickstart) using three components: data, a task, and scoring functions. I find the following form useful because it reduces a broad evaluation problem to three base questions:

**Eval = Task + Dataset + Scorer**

<figure class="article-figure article-figure-plain">
  <img src="/images/illustrations/eval-task-dataset-scorer.png" width="1214" height="713" alt="An eval branches into Task, Dataset, and Scorer, representing the system being run, the situations being tested, and the definition of success.">
</figure>

This is more than a convenient API shape. It is a compact specification for what an eval result actually means.

### Task: What System Am I Running?

The **task** is the executable application under test. It takes an input from a test case and produces an output, ideally together with the trace and operational data required for measurement.

For a simple classifier, the task may be a single model call. For an agent, it may include the prompt, model, orchestration loop, retrieval configuration, available tools, skill registry, state, stopping conditions, and safety policies. If any of those change, I may be evaluating a different task even when the user-facing feature has the same name.

This is why "I am evaluating the model" is usually an incomplete statement. A model does not decide which documents enter its context, which tools it can call, how tool errors are handled, or when an action requires approval. The application does. The task should represent the smallest complete system whose behavior I intend to compare.

The base question is:

**What exact system version produced this run?**

Without that answer, a score cannot be reproduced or attributed to a meaningful change.

### Dataset: What Situations Am I Testing?

The **dataset** defines the situations in which I ask the task to operate. A row may contain an input, an expected output, relevant context, initial state, constraints, metadata, or a reference trajectory. Not every case needs all of these fields. It needs enough information to recreate the situation and judge the behavior I care about.

A dataset is not merely a bag of prompts. It is a hypothesis about the distribution of situations the application will face. If it contains only clean, common requests, then the eval says nothing about ambiguous inputs, adversarial instructions, tool failures, long contexts, rare policy boundaries, or cases collected from production incidents.

Coverage determines the scope of every conclusion. A task that scores 95% on fifty easy examples is not "95% good." It achieved that score on that particular mixture of situations. Change the mixture, and the meaning of the number changes.

The base question is:

**Which behaviors, edge cases, and known failures does this dataset represent, and which ones are missing?**

### Scorer: What Counts as Success?

The **scorer** turns observations from a run into a judgment. It is the executable form of the product requirement.

Success is rarely one-dimensional for an agent. I may care about factual correctness, instruction following, groundedness, tool selection, argument validity, policy compliance, latency, and cost at the same time. One scorer can answer one narrow question, while a collection of scorers can describe several independent properties of the run.

This separation prevents a common mistake: compressing every concern into a single "quality" score. An answer can be correct but ungrounded. A trajectory can be safe but inefficient. A response can be helpful while violating a formatting contract. Keeping these judgments separate preserves information that an average would hide.

A scorer is also limited by what the application exposes. It cannot reliably judge tool selection if tool calls are absent from the trace. It cannot measure cost without usage data. It cannot verify citation support if retrieved evidence is discarded. Observability is therefore not an optional layer added after evaluation. It determines which desired behaviors can become tests at all.

The base question is:

**What observable evidence would convince me that this run succeeded or failed?**

### A Score Is Conditional on All Three

The decomposition matters because no eval score stands alone. Every result is conditional on a particular task, a particular dataset, and a particular set of scorers.

If I change the task while holding the dataset and scorer fixed, I can ask whether the application improved. If I expand the dataset while holding the task and scorer fixed, I can ask whether the original conclusion survives broader coverage. If I revise the scorer, I am changing the definition of success and must reinterpret previous results accordingly.

This gives me a useful sentence to complete before designing any evaluation:

> For **this system**, across **these situations**, success means **these observable behaviors**.

If I cannot fill in all three blanks, I am not ready to choose a metric. The uncertainty is still in the specification.

The question "Does the agent work?" is too underspecified to test. The better questions are: Which version of the application? On which slice of reality? According to which definition of success? **Task, Dataset, and Scorer** form the base abstraction I will use throughout the rest of this article to answer them.

## 3. Why Agents Break the Unit-Test Mental Model

The mental model behind a conventional unit test is powerful because it is small. I provide an input, execute a function, and compare the actual result with an expected result:

<p class="concept-equation">application(x) = expected_output</p>

This assumes that the unit has a stable boundary, that its behavior can be isolated, and that I know what the correct output should be. When the assertion fails, the location of the test usually gives me a useful starting point. The tested function, or one of its direct dependencies, violated a relatively local contract.

Unit tests remain valuable inside an AI application. I still want them for parsers, permission checks, tool wrappers, schema validation, deterministic transformations, and any other component with a crisp contract. The problem begins when I treat an entire agent as if it were one deterministic function with one golden output.

An agent run is closer to a sequence of decisions over changing state. An input first reaches a router, which may activate a skill and call a tool. The tool result can trigger retrieval, another model decision, and another tool call before the application finally produces a response.

This sequence is not necessarily fixed. The router may select a different branch. A skill may or may not activate. The model may decide that no tool is needed, call several tools, retry one of them, or stop early. State created at one step becomes context for the next. What looks like a pipeline in a single trace is one realized path through a larger graph of possible actions.

<figure class="article-figure article-figure-plain">
  <img src="/images/illustrations/agent-router-downstream-workflows.png" width="813" height="805" alt="A query enters a router that can choose semantic search over documents, text-to-SQL over databases, or REST API calls to external services.">
</figure>

The router in this architecture does more than forward a request. It decides which downstream world the agent will see. A semantic-search branch exposes documents. A text-to-SQL branch exposes structured database records. A function-calling branch exposes actions and live data from external systems. Once the route is selected, every later step operates on the evidence, capabilities, and constraints of that branch.

This creates two problems that the simple unit-test mental model does not capture well.

### Problem 1: Nondeterminism

For many agent tasks, one input can have several correct outputs.

If I ask an agent to summarize an incident report, two responses can use different wording, structure, and levels of detail while remaining equally faithful. If I ask it to find a refundable order, it may retrieve the same record through different valid queries. If I ask it to solve a multi-step task, several tool sequences may reach the same acceptable outcome.

An exact-match assertion treats harmless variation as failure:

<p class="concept-equation">actual_output = one_reference_output</p>

That is too narrow when correctness belongs to a set of acceptable behaviors rather than a single string. The evaluation needs an **acceptance region**. An output may vary, but it must remain inside constraints such as factual correctness, policy compliance, groundedness, required format, or task completion.

Nondeterminism, however, is often used as too broad an explanation. The fact that multiple answers can be correct does not mean that correctness is undefinable. Many properties should remain stable across valid runs:

- The answer should be supported by the available evidence.
- A restricted tool should never be called without approval.
- Tool arguments should satisfy the schema and the user's constraints.
- The agent should not invent records that retrieval did not return.
- Cost and latency should remain within an acceptable budget.

The exact words may differ while these invariants remain testable. The goal is not to force the agent to produce the same trajectory every time. It is to distinguish **allowed variation** from **behavioral failure**.

Nondeterminism also changes the interpretation of a passing run. One success shows that the application can produce an acceptable trajectory. It does not establish how frequently that trajectory occurs. If a dangerous route appears once in every fifty runs, testing the prompt twice is unlikely to reveal it. The object I care about is therefore a distribution of behavior, not a single demonstration.

### Problem 2: Cascading Failures

The more difficult problem is **failure propagation**. Each step changes the inputs available to every step after it. A mistake near the beginning can redirect the entire run while all downstream components continue behaving correctly relative to the wrong state they received.

Suppose a user asks, "Has invoice `inv_4821` been refunded?" The router should select a branch that can inspect live billing data. Instead, it routes the request to semantic search. The retriever correctly finds the company's refund-policy document. The model accurately summarizes that policy. The final response is fluent, grounded in the retrieved text, and completely fails to answer whether this invoice was refunded.

The retriever did not malfunction. The model did not hallucinate. The semantic-search branch did exactly what it was designed to do. The system **correctly executed the wrong plan** because the first routing decision placed every later component in the wrong problem.

This is why final-answer scoring is necessary but insufficient. A final scorer can tell me that the user did not receive the right answer. It cannot tell me whether the cause was routing, skill activation, tool selection, argument construction, a tool error, missing retrieval evidence, state corruption, or bad synthesis.

**A bad final answer does not tell me where the system failed.**

The converse matters too. A good final answer does not prove that the trajectory was good. An agent might recover from a wrong route through retries, call an unnecessary privileged tool, leak sensitive context to a component that did not need it, or spend far more than the task justified. Outcome-only evaluation can reward a system that happened to end in the right place after taking an unacceptable path.

### Treat Each Step as a Contract

To localize failures, I can treat the boundaries between components as testable contracts. Each component receives some state, makes a decision or transformation, and passes new state downstream.

| Component | Contract to evaluate | Example observation |
| --- | --- | --- |
| **Router** | Select an appropriate destination for the request | chosen route, route confidence, allowed alternatives |
| **Skill selection** | Trigger relevant skills and suppress irrelevant ones | activated skill IDs, missing required skill |
| **Tool selection** | Choose an allowed capability for the current intent | tool name, policy decision, approval state |
| **Tool arguments** | Translate intent and constraints into a valid call | schema validity, entity IDs, amounts, filters |
| **Tool result handling** | Preserve errors and relevant returned data | status, parsed fields, dropped values |
| **Retrieval** | Surface evidence that is relevant and sufficient | retrieved items, coverage, source metadata |
| **Synthesis** | Produce claims supported by the available state | final claims, citations, constraint adherence |
| **Final response** | Resolve the user's task | correctness, usefulness, completion |

This does not mean every run needs a separate score for every row. It means the trace should expose enough information that I can attach a scorer where a failure matters. For a read-only question, tool-approval scoring may be irrelevant. For a financial action, tool choice, arguments, authorization, and final outcome may all deserve independent checks.

The first failing contract is often more informative than the final symptom. If the router selected the wrong branch, later failures may be consequences rather than independent defects. Fixing the final prompt will not repair the routing policy. Adding more retrieval documents will not fix a malformed tool argument. Component-level evaluation helps me change the part of the system that actually caused the divergence.

### The Trace Is Part of the Evaluated Output

In the abstraction from Part 2, agentic systems expand what each term must contain:

- The **Task** must expose not only the final response but also the relevant execution trace.
- The **Dataset** may specify an expected route, required evidence, allowed tools, forbidden actions, initial state, or known failure checkpoint.
- The **Scorers** can evaluate both end-to-end success and the contracts of individual steps.

End-to-end and component-level evals answer different questions. The end-to-end eval asks, "Did the user get an acceptable result?" Component evals ask, "Did the system make the right decisions along the way, and where did it first diverge?" I need both. Component scores without an outcome can optimize locally while the product still fails. Outcome scores without component evidence reveal regressions without making them diagnosable.

The unit-test instinct is still useful: isolate behavior, state the contract, and make failure reproducible. What changes for agents is the unit. The entire agent cannot always be reduced to `input -> one expected output`. The testable object is often an execution path with multiple acceptable branches and several contracts that can fail independently.

A useful agent eval should therefore answer three questions:

1. **Did the application achieve an acceptable outcome?**
2. **Did it follow an acceptable trajectory?**
3. **If not, at which step did behavior first diverge from the contract?**

That final question is what turns evaluation from judgment into debugging. A score tells me that something went wrong. A trace with step-level evals tells me where to start fixing it.

## 4. Observability Before Evaluation

**I cannot design a useful test suite for behavior I have never observed.**

Before I decide how to score an agent, I need to see what the application actually does. Not what the architecture diagram says it should do. Not what the final response claims it did. I need a record of the concrete path taken by a real run, including the decisions, external calls, state transitions, errors, latency, and cost that disappeared behind the final answer.

Consider a research agent that gathers the right evidence, produces a strong report, and then fails to save it because the service account lacks write permission. The user sees no artifact and a generic failure message. An output-only diagnosis may blame research or synthesis and lead me to change the prompt, model, or documents. The actual failure occurred in the **delivery layer**.

The final output compresses an execution into one symptom. **The response was wrong** describes the application boundary; it is not a root cause.

### Trace the Run, Span by Span

[OpenTelemetry defines a trace](https://opentelemetry.io/docs/concepts/signals/traces/) as the path a request takes through an application. For an agent, I can think of a **trace** as the record of one complete run, from the arrival of the user's request to the final response or action.

A **span** represents one operation inside that run. A routing decision can be a span. So can a retrieval call, model generation, tool invocation, database query, policy check, or artifact delivery. Spans share a trace identifier and use parent-child relationships to preserve causality. Together, they reconstruct not only what happened, but in what order and under which parent operation.

The report failure might produce a trace like this:

| Span | Useful observations | Status |
| --- | --- | --- |
| **Agent run** | request ID, application version, input, total latency, total cost | Failed |
| **Route request** | selected research workflow, available alternatives | OK |
| **Retrieve evidence** | queries, document IDs, source metadata, retrieval scores | OK |
| **Generate report** | model and prompt version, evidence references, token usage | OK |
| **Write artifact** | tool name, destination path, permission error | Error |
| **Deliver result** | expected artifact URI, missing artifact | Failed |

Now the diagnosis is different. The first meaningful divergence is the `Write artifact` span. Every preceding contract passed. `Deliver result` failed too, but it is a downstream consequence, not an independent root cause.

This is why the hierarchy matters. A flat collection of logs may tell me that two errors occurred. A trace tells me that one caused the other.

### What a Useful Span Should Contain

A span is useful when it records enough context to explain and compare an operation. The exact fields depend on the component, but I generally want:

- **Identity:** trace ID, span ID, parent span ID, operation name, application version, and environment.
- **Timing:** start time, end time, latency, retry count, and timeout state.
- **Inputs and outputs:** the relevant values, safe references to large payloads, or hashes when raw content should not be stored.
- **Configuration:** model, prompt, tool schema, retrieval index, skill version, and policy version used for the operation.
- **Decisions:** selected route, activated skills, chosen tool, approval result, stopping reason, and alternatives when the system exposes them.
- **Operational measurements:** token usage, monetary cost, number of retrieved items, external requests, and cache behavior.
- **Failure information:** status, structured error type, exception or tool error, and whether the failure was retried or recovered.

[The OpenTelemetry tracing specification](https://opentelemetry.io/docs/specs/otel/trace/api/) describes spans with names, identifiers, parent relationships, timestamps, attributes, events, links, and status. Agent instrumentation extends those general fields with application-specific observations such as tool arguments, retrieved evidence, model usage, and state changes.

More telemetry is not automatically better. A raw dump of every prompt, document, and tool result can leak secrets, personal data, or privileged business information. It can also make traces expensive and difficult to search. I want the **minimum sufficient evidence** needed to reproduce a failure, compare runs, and identify the first broken contract. Sensitive values should be redacted, access-controlled, hashed, or represented by stable references where possible.

I also do not need hidden chain-of-thought to make an agent observable. A model's narrated explanation of why it acted is not a guaranteed record of the mechanism that produced the action. What I need are the observable interfaces around the model: the context supplied, the decision emitted, the tools available, the arguments selected, the results returned, and the state passed forward. Those are the artifacts I can test.

### A Symptom Is Not a Root Cause

Once traces exist, failures can be classified by the first contract that broke. A useful initial taxonomy is:

| Failure class | What went wrong | Evidence to inspect |
| --- | --- | --- |
| **Bad data** | Required evidence was missing, stale, corrupted, or incorrect | source version, retrieved records, freshness, coverage |
| **Bad routing** | The request entered the wrong workflow, skill, or knowledge domain | router input, selected route, alternatives, confidence |
| **Wrong tool** | The agent chose a capability that could not satisfy the intent | available tools, selected tool, policy constraints |
| **Wrong arguments** | The tool was appropriate, but its parameters were malformed or semantically wrong | schema validation, entity IDs, filters, paths, amounts |
| **Bad reasoning** | The necessary evidence was available, but the produced decision or synthesis did not follow from it | supplied context, intermediate decision, unsupported claims |
| **Bad formatting** | The content was correct but violated a required schema, template, or interface contract | raw response, parser result, validation errors |
| **Bad delivery** | A correct result was not persisted, transmitted, displayed, or made available to the user | write status, network response, artifact URI, UI state |

These labels are not valuable because they create a perfect ontology. They are valuable because they point to different fixes. Bad data may require index freshness or source validation. Bad routing may require new router examples or a sharper boundary between skills. Wrong arguments may require schema changes or constrained decoding. Bad delivery may require permissions, retries, idempotency, or transactional guarantees.

Calling all of them **model quality failures** destroys that information. It encourages prompt changes even when the prompt is not the broken component.

The first broken span is not always the whole story. A router may choose the wrong branch because metadata upstream was missing. A tool may receive a wrong identifier because entity resolution silently returned an ambiguous match. Root-cause analysis can therefore move backward through the trace until it finds the earliest actionable defect. The important shift is from asking "Which component reported an error?" to asking **"Where did the run first become unable to succeed?"**

### Prioritize Failures by Expected Harm

Observability will usually reveal more failures than a team can fix at once. Counting them is not enough. The most frequent failure is not necessarily the most important one.

<p class="concept-equation">Priority(failure) = Frequency × Severity</p>

**Frequency** is how often the failure occurs within the relevant traffic slice or opportunity for failure. It can be measured as a rate, such as failures per eligible run, rather than a raw count that merely reflects traffic volume.

**Severity** is the consequence when the failure occurs. Depending on the product, it may combine user harm, financial loss, safety impact, privacy exposure, irreversibility, and the cost of recovery.

If frequency is an estimated probability and severity is an estimated cost, their product approximates **expected harm**. With ordinal scales it is only a ranking heuristic. Either way, a formatting defect in 15% of runs may deserve less attention than a 0.1% chance of writing to the wrong customer's workspace. Counts alone optimize the dashboard rather than the product's risk.

Frequency also needs a denominator. A tool failure seen ten times in one hundred tool calls is different from one seen ten times in one million calls. Severity needs context too. A malformed date in an internal draft is not equivalent to a malformed dosage in a clinical workflow. Useful prioritization is conditional on the population, task, and consequence being measured.

### Observability Is the Dataset Discovery Layer

The deepest connection between observability and evaluation is that traces tell me what belongs in the test suite.

A practical loop looks like this:

1. **Observe** real and staged runs with traces that preserve the important component boundaries.
2. **Detect** bad outcomes, policy violations, unusual cost, high latency, tool errors, or other signals worth investigating.
3. **Localize** the first span where the run became unable to satisfy the intended contract.
4. **Classify** the root cause and estimate its frequency and severity.
5. **Promote** the trace, or a safely minimized version of it, into a reproducible dataset case.
6. **Score** both the final outcome and the specific component contract that failed.
7. **Fix and rerun** the case, then keep it as a regression test.

This loop turns production incidents into durable engineering knowledge. The trace supplies the evidence. The dataset preserves the situation. The scorer encodes the contract. The regression test makes the failure harder to reintroduce.

It also changes how I sample traces. Random samples are useful for estimating common behavior, but rare high-severity failures can disappear inside averages. I want to retain traces triggered by errors, policy violations, unusually high cost, long latency, retries, unexpected tools, and user corrections. These are often the runs with the highest information value for future evals.

Observability does not replace evaluation: a trace shows what happened without saying whether it was acceptable. Evaluation does not replace observability: a score flags a violated contract without explaining why. Together they let me replace **the response was wrong** with an actionable diagnosis such as bad routing, a wrong account ID, ignored evidence, or failed delivery.

Before I measure failures, I must make them visible. Before I prevent them, I must make them reproducible.

## 5. Read the Traces Before Writing the Grader

Once an application is instrumented, the temptation is to jump straight to metrics. I open a catalog, choose **relevance**, **faithfulness**, **tool correctness**, **task completion**, and perhaps an LLM judge that returns a score from one to five. The dashboard fills up. The evaluation system looks mature.

But I may still be measuring the wrong things.

<figure class="article-figure article-figure-plain">
  <img src="/images/illustrations/llm-traces-and-evaluation-metrics.png" width="1630" height="580" alt="An LLM trace composed of nested agent, retriever, embedding, model, and tool-call spans beside possible metrics for multi-turn, retrieval, and agent behavior.">
</figure>

The image captures both the opportunity and the trap. On the left, a run is decomposed into spans. On the right, there are many plausible metrics that could be attached to those spans. **Tool correctness** belongs near a tool decision. **Context precision** belongs near retrieval. **Task completion** belongs at the level of the full run. Different metrics observe different contracts.

What the image cannot tell me is which contracts matter for my application, where their boundaries should be, or what passing behavior looks like. A list of available metrics is not a product specification.

The better flow is:

<p class="concept-equation">Traces → Failure Examples → Failure Taxonomy → Requirements → Evaluators</p>

Airbnb describes the same sequencing as **eval-driven development**: prototype first, inspect outputs and traces, categorize the mistakes, and only then encode those observed failure modes as evals. Its practical recommendation is deliberately hands-on—run a prototype over roughly 100 inputs and read the outputs before investing in a scaled evaluation pipeline. [The point is discovery, not a magic sample size](https://airbnb.tech/ai-ml/eval-driven-development-lessons-from-evaluating-genai-at-scale/): early manual review prevents a team from scaling generic metrics that never represented the failures users encounter.

Each transition removes a different kind of ambiguity.

- **Traces** show what the application actually did.
- **Failure examples** identify concrete runs that users or operators consider unacceptable.
- **Failure taxonomy** groups examples that share an actionable cause.
- **Requirements** state the behavior the system must exhibit instead.
- **Evaluators** turn those requirements into repeatable judgments.

If I skip directly from traces to evaluators, I risk encoding whatever is easiest to score rather than what is important to get right.

### Begin With the Failure, Not the Metric

Suppose I am building a customer-support agent that can issue refunds. A user writes:

> I was charged twice for order 8472. Please refund the duplicate charge.

The agent retrieves the order and the current refund policy. It correctly identifies two charges. It then calls `issue_refund` using the order total rather than the amount of the duplicate charge. The payment provider accepts the call, and the agent confidently tells the user that the duplicate charge has been refunded.

At a glance, several parts of the run look good:

- The route to the billing workflow was correct.
- Retrieval found the correct order and policy.
- The selected tool was appropriate.
- The final response was clear and relevant.
- The task appeared to complete successfully.

The trace reveals the actual failure: the `amount` argument was wrong. A generic answer-quality grader may pass the response because the language is excellent. A tool-selection grader may pass because `issue_refund` was the correct tool. Even a task-completion metric may pass if it checks only that the provider returned success.

The failure example gives me a sharper question: **What amount was the agent authorized to refund, and what evidence should establish that amount?**

That question is already closer to a requirement than "Which eval metric should I use?"

### Define What Success Looks Like First

Before writing any grader, I want a plain-language answer to:

**What does success look like for this situation?**

For the duplicate-charge case, success is not merely a polite response or a successful API call. A domain expert might define it as follows:

1. The agent identifies two settled charges for the same order.
2. It selects exactly one duplicate charge, not the legitimate purchase.
3. It verifies that the duplicate is eligible under the current refund policy.
4. It refunds exactly the duplicated amount to the original payment method.
5. It does not execute a second refund if the workflow is retried.
6. It claims success only after the payment provider confirms the transaction.
7. It gives the user the correct amount and refund reference without exposing sensitive payment data.

This definition contains several independent contracts: evidence, policy, argument correctness, idempotency, delivery, and communication. No single generic quality score represents all of them.

It is helpful to separate requirements by the kind of guarantee they express:

| Requirement type | Question | Refund example |
| --- | --- | --- |
| **Outcome** | What state must be true when the run ends? | One duplicate charge is refunded |
| **Trajectory** | Which steps or constraints must the run respect? | Verify eligibility before calling the refund tool |
| **Prohibition** | What must never happen? | Never refund the legitimate charge or refund twice |
| **Communication** | What must the user be told? | Report the confirmed amount and reference accurately |
| **Operational** | Within which resource limits must success occur? | Complete within the latency and cost budget |
| **Uncertainty** | What should happen when evidence is insufficient? | Ask for clarification or escalate instead of guessing |

Writing these requirements before the evaluators prevents a common inversion. The grader should implement the definition of success. The definition of success should not be reverse-engineered from whatever a grading library happens to provide.

### “Good” Is a Stakeholder Concept

The engineer who built the agent often knows the execution graph better than anyone else. That does not mean the engineer has the best definition of good behavior.

Different people see different contracts:

- A **user** knows whether the result actually resolved the problem and whether the explanation was useful.
- A **domain expert** knows the policy, exceptions, terminology, and consequences hidden behind an apparently simple request.
- A **support or operations team** knows which failures recur, which ones are recoverable, and which ones create expensive manual work.
- A **security, legal, or safety stakeholder** knows which actions require approval, auditability, or hard prohibitions.
- An **engineer** knows which observations are available, where component boundaries exist, and which checks can be made deterministic.

For the refund agent, an engineer might initially define success as `refund_api.status == "succeeded"`. A payments specialist will immediately see the missing questions: Was the correct charge selected? Was the amount correct? Was the charge eligible? Was the operation idempotent? Did the provider later reverse or reject it? A user may add another requirement: Did I receive a clear confirmation that lets me recognize the credit on my statement?

None of these perspectives is sufficient alone. The requirement should be negotiated where product intent, domain truth, user value, and technical observability meet.

This is also why traces are useful in requirement conversations. Asking a stakeholder "What is a good support agent?" invites vague answers such as helpful, safe, and accurate. Showing three concrete traces and asking which behavior first became unacceptable produces much sharper rules.

### Turn Requirements Into Evaluators

Only after the behavior is specified do I choose how to evaluate it. For the refund example, the mapping might look like this:

| Requirement | Observable evidence | Evaluator |
| --- | --- | --- |
| Duplicate charge identified | retrieved transaction IDs, timestamps, and amounts | deterministic comparison against account state |
| Correct refund amount | `issue_refund.amount` and duplicate charge amount | exact numeric assertion |
| Eligibility checked first | policy version and parent-child span order | trajectory rule |
| No duplicate execution | idempotency key and provider transaction history | state-transition assertion |
| Success claimed only after confirmation | provider status and final response | cross-span consistency check |
| Clear user explanation | final response and communication rubric | human-reviewed model grader or rubric scorer |
| Cost and latency within budget | trace totals | threshold checks |

The evaluator follows from the evidence.

If success is a numeric invariant, I prefer a deterministic assertion. If it is a schema contract, I use a parser or validator. If it is a state transition, I inspect the before-and-after state. If it concerns semantic quality such as clarity or completeness, a rubric-based human or model grader may be appropriate. If it concerns user value, production outcomes or explicit user feedback may be stronger evidence than any synthetic judge.

This mix is important. **LLM-as-a-judge** is useful for behavior that genuinely requires semantic judgment. It should not replace a direct assertion that two amounts are equal, that a forbidden tool was never called, or that an artifact exists at the promised location. Using a probabilistic grader for a deterministic contract adds uncertainty without adding information.

### Write the Rubric From Real Boundaries

When a model grader is appropriate, the failure examples should shape its rubric. "Rate the response quality from 1 to 5" leaves the grader to invent the meaning of quality. A stronger rubric names the dimensions, evidence, and failure boundaries that stakeholders already agreed on.

For a refund confirmation, the rubric might ask whether the response:

- states the provider-confirmed refund amount correctly,
- distinguishes the duplicated charge from the legitimate one,
- avoids claiming a settlement date that the trace does not support,
- gives the user a useful next step when the provider status is pending,
- avoids exposing full payment identifiers.

The rubric should include passing examples, failing examples, and difficult boundary cases. It should also be tested against expert labels. If the grader disagrees with domain experts on the cases that matter most, tuning its prompt until the score distribution looks tidy does not make it valid.

I care particularly about **false passes** and **false failures**. A false failure creates noise and slows iteration. A false pass certifies behavior that violates the requirement. For high-severity safety or financial constraints, the second error may be far more expensive. Evaluator quality is therefore itself an evaluation problem.

### Preserve the Link From Evaluator to Failure

Every evaluator should be traceable back to a requirement, and every requirement should be traceable back to a user need, policy, risk, or observed failure. I want to be able to ask:

- Which failure class is this evaluator meant to catch?
- Which stakeholder defined the unacceptable boundary?
- Which trace fields provide the evidence?
- Which dataset cases exercise the boundary?
- What action should I take when the evaluator fails?

If I cannot answer those questions, the metric may be decorative. A score can move without telling me whether the product became better or worse.

This traceability also prevents metric accumulation. I do not need every metric shown in a framework's documentation. I need the smallest set that covers the important requirements and localizes the failures I intend to prevent. New evaluators should earn their place by catching a meaningful failure class or protecting a product contract that existing evaluators miss.

### Traces First Does Not Mean Production Only

I do not need to wait for users to discover every failure. Traces can come from prototypes, dogfooding, adversarial exercises, simulations, shadow traffic, or sessions run with domain experts. The principle is not "ship first, evaluate later." The principle is **observe concrete behavior before abstracting it into a grader**.

Early in development, a small set of carefully reviewed traces is often more valuable than a large suite of generic metrics. Those traces expose the application's real decision points and give stakeholders something specific to critique. As the system reaches production, real failures expand the dataset and correct the assumptions embedded in the original requirements.

The result is an eval suite built from evidence rather than fashion. It does not begin with ten popular LLM metrics. It begins with the system's actual trajectories, the failures people care about, and a precise answer to the question **"What would success have looked like here?"**

Only then do I write the grader.

## 6. The Evaluation Stack: Use the Cheapest Judge That Works

Once the requirement and its evidence are clear, the next question is not **Which model should judge this?** It is **What is the cheapest evaluator that can make this decision reliably?**

Agent evaluations usually draw from three layers:

1. **Code-based evaluation:** deterministic checks over outputs, traces, and environment state.
2. **LLM-as-a-judge:** model-based judgment for semantic properties that are difficult to reduce to fixed rules.
3. **Human evaluation:** expert judgment for ambiguous, novel, high-risk, or poorly calibrated cases.

These layers differ in cost, latency, scalability, and the kinds of evidence they can understand. They should not be read as a maturity ladder. Human evaluation is not automatically the best grader for every property, and an LLM judge is not a more advanced version of a unit test. The useful ordering is an **escalation ladder**: begin with the most direct and reproducible test, then pay for more judgment only when the requirement demands it.

| Layer | Best suited for | Main advantage | Main limitation |
| --- | --- | --- | --- |
| **Code** | exact contracts, state, structure, tool use, cost, latency | fast, cheap, reproducible | cannot infer open-ended meaning well |
| **LLM judge** | semantic correctness, groundedness, completeness, tone | scalable judgment over natural language | probabilistic, biased, bounded by its context and knowledge |
| **Human** | novel edge cases, domain expertise, high-stakes decisions, calibration | can apply context, expertise, and product judgment | expensive, slow, and not perfectly consistent |

[Anthropic describes agent evals](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) using the same three grader families: code-based, model-based, and human. The important design choice is not to select one family for the entire application. It is to attach the cheapest competent judge to each contract.

The diagram below zooms in on the transition from model judgment to human review. A test case supplies the input, output, retrieval context, and other evidence to an LLM scorer. The scorer returns a score and, optionally, a reason. A threshold handles clear cases automatically. Borderline cases move to a human, whose decision can resolve the case and later improve the rubric or threshold.

<figure class="article-figure article-figure-plain">
  <img src="/images/illustrations/llm-evaluation-metric-human-escalation.png" width="1473" height="716" alt="An LLM test case is scored by an LLM judge, checked against a pass threshold, and escalated to human evaluation for edge cases.">
</figure>

The image is not a complete execution order. Code checks may run before the model judge, after it, or beside it. A failed schema validation does not need to wait for a model score. A forbidden production write should fail the run even if both the LLM judge and a human reviewer like the final answer. The stack combines independent evidence; it is not a pipeline in which each later layer overrides the earlier one.

### Layer 1: Code-Based Evals

Code-based evals include more than unit tests. They include exact and fuzzy string matching, regular expressions, schema validation, static analysis, assertions over tool calls, database state checks, ordering constraints in a trace, and thresholds for cost or latency. The [OpenAI grader API](https://developers.openai.com/api/reference/resources/graders) reflects this range with string checks, text-similarity graders, Python graders, model graders, and compositions of multiple graders.

I use code whenever success can be expressed as an invariant over observable evidence:

- Did the response parse as the required JSON schema?
- Did the agent call `verify_identity` before `issue_refund`?
- Does `issue_refund.amount` equal the duplicated charge amount?
- Was a forbidden tool ever called?
- Does the promised artifact exist at the returned path?
- Did the run remain below the token, latency, and cost budgets?
- Does the output contain a required identifier or a prohibited pattern?

These checks are attractive because their behavior is legible. The same evidence produces the same result. When a code grader fails, I can inspect the assertion and know exactly which contract was violated. I can run it on every pull request and every trace at a cost close to zero.

Even a crude check can expose a real failure. If a support agent must call `lookup_refund_policy` before stating an eligibility window, a trace assertion can fail any unsupported policy claim. A general correctness judge given only the conversation may accept a plausible but obsolete 30-day answer because it cannot see the tenant's actual 14-day policy. The simple check catches the missing-evidence contract; the richer judge answers a different question from weaker evidence.

Code-based evals become weak when I ask them to infer meaning indirectly. A regex can confirm that a citation-shaped string exists; it cannot establish that the cited source supports the claim. Exact match can verify a known account ID; it cannot decide whether an explanation is sufficiently clear for a confused customer. String similarity can reward lexical overlap while missing a contradiction. The boundary is simple: **use code for properties that are already explicit in data, state, or structure; do not force semantic judgment into a brittle proxy.**

### Layer 2: LLM-as-a-Judge

An LLM judge is useful when valid outputs can vary widely but the acceptance criteria can still be described in language. It can read a response, trace, reference answer, or retrieval context and apply a rubric for properties such as:

- whether a summary preserves the important facts,
- whether claims are supported by the supplied evidence,
- whether an explanation is coherent and complete,
- whether the response follows a nuanced instruction,
- whether the tone is appropriate for the situation,
- whether a tool trajectory appears wasteful or confused when no crisp rule captures the pattern.

This layer occupies the large space between exact assertions and manual review. It is far cheaper and faster than asking a person to inspect every run, and it can return a structured score plus a reason that helps with debugging. For open-ended applications, that makes evaluation at useful scale possible.

But **LLM-as-a-Judge is not the “advanced” replacement for code evals.** It is a different layer for a different kind of contract.

A model judge introduces another nondeterministic system into the harness. Its decision can change with the model version, prompt, response order, rubric, reference, and sampling configuration. Research has documented [position bias](https://aclanthology.org/2025.ijcnlp-long.18/) and substantial variation across models, datasets, properties, and required expertise, reinforcing that judges must be [validated against human judgments](https://aclanthology.org/2025.acl-short.20/) for the task where they will be used. More fundamentally, a judge cannot grade reliably against knowledge it cannot see.

The remedy is not simply to choose a larger judge. I need to design its evidence contract as carefully as I designed the agent's:

1. **Give it the relevant evidence.** If correctness depends on a policy, retrieved document, database record, or reference answer, include that material. Do not ask the judge to reconstruct private or current truth from parametric memory.
2. **Narrow the rubric.** “Rate quality from 1 to 5” asks the model to invent the standard. A rubric should name the criterion, the evidence to inspect, the pass boundary, and the behavior to take under uncertainty.
3. **Prefer separate judgments.** Correctness, groundedness, tone, and completeness can fail independently. One overall score hides disagreement among them.
4. **Test the judge on labeled edge cases.** Measure false passes and false failures against domain-expert decisions, especially near the production threshold.
5. **Monitor judge drift.** A model or prompt update changes the evaluator itself. The judge needs versioning and regression cases just like the task it grades.

The LLM's optional reason in the diagram is useful as an audit artifact, not as proof that the score is correct. A polished explanation can rationalize a bad judgment as fluently as an agent can rationalize a bad action. I should validate the decision against labeled cases, not trust it because the rationale sounds thoughtful.

### Layer 3: Human Evaluation

Human evaluation is the most expensive layer, so it should be spent where human judgment changes the answer: when the requirement needs expertise absent from the model, the boundary is disputed, the case is novel, or the consequence of a false pass is too high. Humans are especially important for:

- defining what good behavior means before automation,
- labeling a representative calibration set,
- adjudicating disagreements between graders,
- reviewing low-confidence and near-threshold cases,
- inspecting rare, high-severity failures,
- evaluating domain-specific correctness and subtle user harm,
- auditing whether the automated stack has drifted away from product intent.

Human evaluation is not an oracle. Reviewers disagree, tire, and interpret vague rubrics differently. They still need the source evidence, examples at the boundary, and an adjudication path. Their unique value is the ability to notice that the question itself is wrong: a source is obsolete, the rubric omits an exception, requirements conflict, or a new failure class exists. Human review therefore improves the layers below it rather than merely overriding them.

### Escalate Uncertainty, Not Every Case

The threshold in the diagram is useful only if it creates three regions rather than pretending every score is equally certain:

- **Clear pass:** automated evidence strongly supports success.
- **Clear fail:** one or more important contracts are violated.
- **Review region:** the score is near the boundary, graders disagree, evidence is incomplete, or the expected harm requires human confirmation.

This makes human effort follow information value. Reviewing the thousandth obvious schema failure teaches me little; reviewing a novel disagreement among code checks, model graders, and user feedback may expose a missing requirement. Repeated crisp decisions should become code checks, repeated semantic distinctions should refine the rubric, and persistent disagreement should remain visible as evidence that the requirement is not yet automatable.

A practical routing rule is:

**Use code when the contract is explicit. Use an LLM when the contract is semantic. Use a human when the contract or the truth is still uncertain.**

Mature suites are therefore hybrids. A refund response may pass tone, fail the exact amount assertion, and be escalated because the policy is ambiguous. The cheapest judge that works is usually the one with the shortest path from evidence to decision: code for explicit truth, a model for semantic contracts, and a qualified human when neither has enough knowledge.

The question is not **Which judge is best?** It is **Which judge can decide this contract from the evidence available, at the reliability and cost the consequence requires?**

## 7. Correctness Is Not Faithfulness

Consider a research agent that receives a set of retrieved documents and produces an answer that accurately summarizes them. Every claim in the answer can be traced to a sentence in the supplied context. The citations are relevant. Nothing was invented.

The answer is **faithful**.

But what if the retrieved documents are stale, incomplete, or wrong?

The agent may have represented its evidence perfectly while still giving the user a false answer. Faithfulness tells me whether the output stays within the provided context. Correctness tells me whether the output is true with respect to the world, an authoritative source of truth, or a validated reference answer.

**Faithfulness(output, context) ≠ Correctness(output, world)**

The distinction looks obvious when written this way, but it is routinely erased by evaluation prompts. A grader is asked, "Is this answer correct according to the context?" That question can test support from the context. It cannot establish that the context itself is correct. Calling the resulting score **correctness** does not give the evaluator access to reality.

Research on retrieval-augmented question answering has treated the two dimensions separately: correctness asks whether the response satisfies the user's information need, while faithfulness asks whether the response is based on the supplied knowledge. [Evaluations of retriever-augmented instruction-following models](https://arxiv.org/abs/2307.16877) found that systems can perform competitively on correctness while still struggling to stay faithful to the provided knowledge. One property does not imply the other.

### Two Different Evidence Relationships

For the operational definition I use here, a claim is faithful when the supplied context supports it. A faithfulness evaluator therefore needs three things:

- the agent's output,
- the context that was available to the agent,
- a rule for deciding whether each output claim follows from that context.

It does not need to know whether the context describes the world accurately. If a retrieved document says a product launched in March and the agent says it launched in March, the claim is faithful to that document. The faithfulness contract has been satisfied.

A correctness evaluator needs a different evidence set. It must compare the output with something that is allowed to establish truth for the task: a gold answer, an authoritative database, a current API result, a verified knowledge base, or a qualified human judgment. For long-form answers, a single reference string is often insufficient. [FActScore](https://aclanthology.org/2023.emnlp-main.741/) addresses this by decomposing a generation into atomic claims and measuring how many are supported by a reliable knowledge source.

The mechanics can look nearly identical: decompose the answer into claims, retrieve evidence, and verify each claim. The difference is where the evidence comes from. Work on faithfulness and factuality in retrieval-augmented systems makes this distinction explicit: [faithfulness checks claims against the passages supplied to the model, while factuality retrieves evidence from a broader knowledge base](https://aclanthology.org/anthology-files/pdf/climatenlp/2025.climatenlp-1.17.pdf).

This produces four possible outcomes:

| | **Correct in the world** | **Incorrect in the world** |
| --- | --- | --- |
| **Faithful to context** | The context is accurate and the answer uses it correctly | The answer accurately repeats stale, incomplete, or false context |
| **Unfaithful to context** | The answer reaches the truth from memory or another source that was not provided | The answer is unsupported and wrong |

Only the top-left cell is the ideal RAG behavior. The other three cells describe different failures and require different fixes.

If the answer is faithful but incorrect, changing the generation prompt may do nothing. The failure is likely in retrieval, source quality, freshness, or the authority assigned to a document. If the answer is correct but unfaithful, the user happened to receive the right result, but the system cannot show that it came from the evidence it was supposed to use. That may be acceptable for a casual question and unacceptable for a legal, financial, medical, or internal-policy workflow.

Correctness and faithfulness are therefore not competing scores. They protect different boundaries.

### The Divergence Appears in Real Evaluations

This is not merely a conceptual edge case. Published RAG evaluations produce materially different results depending on whether claims are checked against the context given to the model or against a broader source of truth.

In an evaluation of ClimateGPT, researchers measured claim support in two ways. They checked generated claims against the supplied RAG reference as a measure of faithfulness, then against a broader climate knowledge base as a measure of factuality. With retrieval enabled, the original ClimateGPT achieved only **30%** claim support against the supplied reference but **61%** against the knowledge base. The Faithful+ variant raised those scores to **57%** and **69%**, respectively. The same outputs therefore received very different judgments because the evaluators asked different questions of different evidence sets. [The paper reports both measurements side by side](https://aclanthology.org/anthology-files/pdf/climatenlp/2025.climatenlp-1.17.pdf).

The reverse failure—following context more closely while becoming less accurate—also appears empirically. Under fact-level conflicts, FaithfulRAG methods reduced errors from ignoring context but increased **incorrect-match errors**, in which the model learned the wrong thing from misleading evidence. [The authors describe this as contextual overfitting](https://aclanthology.org/2025.acl-long.1062/): better obedience to retrieved evidence is not the same as better judgment about whether that evidence should be trusted.

These results expose a third contract that a single answer score would hide: **context adequacy**. Before asking whether the model used the context faithfully, I should ask whether the context was sufficient, current, and authoritative enough to answer the question at all.

For a RAG or agent system, I can separate the responsibilities:

| Component | Evaluation question |
| --- | --- |
| **Source and retrieval** | Did the system provide evidence that was relevant, sufficient, authoritative, and fresh enough? |
| **Generation** | Did the output remain faithful to the evidence actually provided? |
| **End-to-end outcome** | Was the resulting answer correct with respect to the task's source of truth? |

The agent can pass the generation contract while the application fails end to end. That is exactly why the first broken contract matters more than the final symptom.

### Current Correctness Requires Current Ground Truth

FreshQA was created because static QA benchmarks cannot evaluate knowledge that changes. It pairs the candidate response with an explicit current answer and date, so the judge is not asked to manufacture ground truth from memory. The benchmark itself must be refreshed because correctness can change while the question remains identical. [FreshLLMs publishes its cases together with the evaluator prompt](https://aclanthology.org/2024.findings-acl.813/).

References help but do not make a judge infallible. Without them, judges tend to over-credit incorrect responses; with them, a judge may still favor its parametric belief over the supplied answer. Studies therefore recommend [reference-aware calibration](https://arxiv.org/abs/2607.12885) while documenting the remaining risk of [over-reliance on parametric knowledge](https://arxiv.org/abs/2601.07506).

Together, these results support a narrower and more defensible claim than the invented scorecard: a correctness judgment is only as meaningful as the ground truth supplied to the evaluator and the evaluator's demonstrated ability to use it. If a task depends on live internal state, the test needs a timestamped snapshot, access to the authoritative tool, or a reference captured at execution time. Without one of those, the honest result may be **not evaluable from available evidence**, not **fail**.

### The Evaluator Has an Information Boundary Too

It is easy to focus on whether the agent had enough context and forget that the evaluator is another information-processing system with its own boundary.

An evaluator can judge only relationships that are visible in its inputs. Given output and context, it can test groundedness. Given output and a verified reference, it can test agreement with that reference. Given output and live environment state, it can test current correctness. Given only output and a rubric, it can assess qualities such as clarity or style, but any factual judgment depends on the evaluator's uncertain parametric knowledge.

This yields a practical design rule:

**Do not ask a judge to evaluate correctness unless it can see the source of truth that defines correctness for the task.**

The source of truth changes by product:

- For a math problem, it may be an independently verified solution or executable checker.
- For a support policy, it may be the effective policy version for that customer and date.
- For a database question, it may be a snapshot or query result from the relevant transaction boundary.
- For current research, it may be authoritative sources with publication dates and explicit coverage requirements.
- For an open-ended expert task, it may require domain-expert review rather than a generic model judge.

Even a supplied citation deserves care. A citation can support a claim while failing to be the reason the model produced it. Research on RAG attribution calls this **post-rationalization**: the model may generate from prior knowledge and attach a superficially compatible citation afterward. Experiments have found substantial rates of such unfaithful attribution, reinforcing that [citation support and genuine reliance on evidence are separate questions](https://arxiv.org/abs/2412.18004).

For most application evals, I cannot observe the model's internal causal process directly. I can still test the operational contract available to me: whether every material claim is supported by the context supplied, whether required evidence was retrieved before the claim was made, and whether perturbing or removing that evidence changes the behavior as expected. The important point is to state exactly which form of faithfulness the evaluator measures rather than using the word as a general synonym for truth.

### Choose the Evaluation Question Before Tuning the Evaluator

When a correctness and faithfulness score disagree, my first response should not be to rewrite the judge prompt. I should ask four questions:

1. **What relationship is this metric intended to test?** Output to context, output to reference, or output to current world state?
2. **What evidence did the agent have?** Retrieved passages, tool results, database state, or only model memory?
3. **What evidence did the evaluator have?** Was it at least as current and authoritative as the evidence available to the agent?
4. **Which component owns the failure?** Retrieval, source freshness, generation, the reference set, or the evaluator itself?

Only after those questions are answered does evaluator tuning make sense. Otherwise I may optimize agreement with a judge that is answering the wrong question.

Faithfulness is valuable because it makes the agent accountable to the evidence it received. Correctness is valuable because users ultimately care whether the answer is true. Neither metric subsumes the other, and neither can be interpreted without its evidence boundary.

**A faithful answer can still be wrong. A correct answer can still be unsupported. And a judge without the relevant truth can score only its own ignorance.**

## 8. From a Trace to a Release Gate: A Minimal Eval Harness

The previous sections establish the pieces: tasks, datasets, traces, requirements, and graders. To make them operational, I need one more component: a harness that can recreate a situation, run the application, collect the evidence, grade the result, and compare it with a baseline.

An **evaluation harness** is the infrastructure around the eval. It provisions the test environment, executes trials, records versions and traces, invokes graders, aggregates results, and decides where those results go next. The harness does not define what good means. The task and its graders do that. Its job is to make the judgment repeatable.

### Start With One Executable Case

The duplicate-charge scenario from Section 5 is an illustrative fixture, not a claimed production incident. A minimized case might look like this:

<pre class="eval-spec" aria-label="Example evaluation case in YAML">id: refund_duplicate_charge
input:
  user_message: "I was charged twice for order 8472. Refund the duplicate."
initial_state:
  order_id: 8472
  settled_charges:
    - { id: ch_1, amount: 49.00 }
    - { id: ch_2, amount: 49.00 }
  refunds: []
  policy_version: refunds-2026-08-01
requirements:
  expected_outcome: exactly_one_duplicate_charge_refunded
  required_evidence:
    - current_order_state
    - current_refund_policy
  prohibited_actions:
    - refund_more_than_49.00
    - refund_more_than_once
graders:
  - correct_final_state
  - policy_checked_before_refund
  - no_forbidden_action
  - confirmation_matches_provider_state
metadata:
  slice: duplicate-charge
  severity: critical
  source: illustrative-fixture
</pre>

This row contains more than a prompt and a reference answer. It contains the state from which the agent must begin, the outcome that must become true, evidence the agent is required to consult, actions it must never take, and metadata that lets me report results for the relevant product slice.

I should also be able to execute a **reference solution** against the same fixture and pass every grader. If a known-good implementation cannot pass, the task, environment, or grader is broken. A frontier agent scoring zero across many attempts is not automatically evidence of low capability; it may be evidence of an impossible or ambiguous test.

The dataset also needs the negative counterpart. If I test only that the agent refunds duplicate charges, I may optimize it into refunding any repeated-looking payment. A balanced suite includes cases where two charges are legitimate, the evidence is ambiguous, the policy forbids an automatic refund, and the correct action is to ask, abstain, or escalate. Anthropic recommends both reference solutions and balanced problem sets for this reason: the eval must test when a behavior should occur and when it should not. [Its field guide describes these checks in the path from an initial task set to a stable harness](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents).

### A Task Is Not a Trial

The dataset row is a **task**. Each execution of that task is a **trial**. Because the agent is nondeterministic, one trial is one sample from its behavior, not a final verdict on its reliability.

Every trial should begin from an isolated environment:

- restore the same database fixture and policy version,
- remove files, caches, messages, and credentials left by previous trials,
- pin the application, prompt, model, tool schema, and grader versions,
- set explicit time, token, cost, retry, and action limits,
- assign a fresh trace ID and idempotency namespace.

Without isolation, trials can contaminate one another. An earlier run may leave a refunded charge, a cached answer, a generated file, or a warm retrieval result that makes the next run artificially easy. Shared resource exhaustion can create the opposite effect and make several trials fail for the same infrastructure reason. Those observations are correlated; counting them as independent agent failures produces a misleading score.

The harness should therefore return at least three top-level statuses:

| Status | Meaning | Release treatment |
| --- | --- | --- |
| **Pass** | The trial ran correctly and satisfied the required contracts | Include in product metrics |
| **Agent failure** | The trial ran correctly and the application violated a contract | Include as a failed product result |
| **Harness error** | The fixture, dependency, grader, timeout, or runner made the result invalid | Exclude from product score and investigate separately |

Silently converting harness errors into agent failures punishes the application for a broken test. Silently retrying them until they pass hides unreliable infrastructure. The harness itself needs error rates, logs, ownership, and regression tests.

### Grade the Outcome, the Trajectory, and the Operation

After a trial, I want three evidence bundles:

1. **Outcome:** Does the provider state contain exactly one refund for `49.00` on the duplicated charge?
2. **Trajectory:** Did the agent consult the effective policy, select the correct charge, obtain any required approval, and avoid prohibited actions?
3. **Operation:** How many turns, tool calls, retries, tokens, dollars, and seconds did the trial consume?

The outcome should dominate when multiple valid paths exist. I should not require one exact tool sequence merely because it is the sequence I imagined. Strict trajectory assertions belong where order is itself a product or safety contract: identity must be verified before disclosure, approval must precede a high-value refund, and a production write must never occur during a freeze. For everything else, a flexible path plus a correct final state is often better than a brittle golden trajectory.

The cheapest competent grader still applies. State and numeric invariants use code. Semantic explanation quality may use a calibrated model grader. Missing or conflicting evidence should produce `unknown` or `needs_review` rather than force an invented pass/fail judgment. Human reviewers adjudicate the boundary cases and periodically verify that the automated graders still represent product judgment.

### Measure Reliability, Not a Memorable Run

Suppose the refund task passes 15 of 20 isolated trials. The empirical success rate is 75%, but the product meaning depends on how the agent will be used.

**pass@k** asks whether at least one of `k` attempts succeeds. Under an independence assumption, a system with per-trial success probability `p` has probability `1 - (1 - p)^k` of at least one success. This is useful when retries are allowed and one successful solution is enough.

**pass^k** asks whether all `k` attempts succeed. Its corresponding probability is `p^k`. At `p = 0.75`, the probability of three consecutive successes is only about 42%. This is often the more relevant view for customer-facing or safety-sensitive agents, where users need the behavior to be consistently correct rather than occasionally recoverable. Anthropic uses these two measures to distinguish capability from consistency in nondeterministic agent evals. [The distinction becomes substantial as the number of trials grows](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents).

No result should be reported as a naked percentage. I want the numerator, denominator, number of trials per task, harness-error count, and results by meaningful slice. A 95% aggregate can hide a 40% pass rate for multilingual requests or a single catastrophic authorization failure. When comparing a candidate with production, I run both on the same cases and fixtures, report uncertainty around the difference, and inspect the trials where their verdicts disagree. Small score movements on small datasets are evidence to investigate, not automatic proof of improvement.

### Separate Capability From Regression

A **capability suite** asks what the agent can do now. It should contain difficult but valid tasks and may begin with a low pass rate. It provides room to improve.

A **regression suite** asks whether behavior the product already depends on still works. Known incidents, contractual behaviors, and critical safety cases belong here. Its expected pass rate should be close to 100%, with hard gates for requirements that cannot be traded against helpfulness or tone.

An overall release rule can then remain deterministic even when some component graders are probabilistic:

<div class="release-gate" role="group" aria-label="Example release gate">
  <section class="release-gate-rule release-gate-block">
    <strong>Block the candidate if</strong>
    <ul>
      <li>any critical safety, authorization, or state-integrity case fails;</li>
      <li>the regression suite drops beyond the agreed tolerance;</li>
      <li>a protected traffic slice regresses materially;</li>
      <li>latency, cost, retry, or action budgets are exceeded;</li>
      <li>grader disagreement falls inside the human-review region.</li>
    </ul>
  </section>
  <section class="release-gate-rule release-gate-allow">
    <strong>Allow a controlled release if</strong>
    <ul>
      <li>all hard gates pass;</li>
      <li>capability gains survive repeated trials;</li>
      <li>no important slice or operational metric regresses;</li>
      <li>harness errors remain below their own reliability threshold.</li>
    </ul>
  </section>
</div>

OpenAI's evaluation guidance calls for this kind of continuous process: evaluate early, run scoped tests on changes, log behavior, calibrate automation with human feedback, and grow the eval set as new nondeterministic cases appear. [Continuous evaluation turns the suite into a release practice rather than a periodic report](https://developers.openai.com/api/docs/guides/evaluation-best-practices).

### Evals Measure Controls; Guardrails Enforce Them

The Replit incident exposes a boundary that an eval article should not blur. An eval can detect a production write, estimate how often the agent attempts it, preserve the incident as a regression case, and verify that a new architecture behaves better. The eval does not itself revoke database credentials or stop a destructive query already in flight.

<figure class="article-figure article-figure-plain">
  <img src="/images/illustrations/guardrails-evals-action-layer.webp" width="1768" height="686" loading="lazy" decoding="async" alt="An agent pipeline with pre-model and post-model guardrails, traces feeding evals, an action layer, and a feedback loop for dynamic adaptation.">
</figure>

**Guardrails** are runtime controls. They include policy checks, input sanitization, output validation, redaction, least-privilege permissions, sandboxing, approval requirements, transaction boundaries, and limits on high-risk tools. Their job is to prevent, constrain, or interrupt an action before unacceptable harm occurs.

**Evaluators** judge behavior. Many run offline or asynchronously because model-based judgment is too slow, expensive, or uncertain for the critical path. A small evaluator can become an inline guardrail only when its latency and false-positive/false-negative trade-offs are acceptable for production. [Hamel Husain and Shreya Shankar make this distinction explicit](https://hamel.dev/blog/posts/evals-faq/whats-the-difference-between-guardrails-evaluators.html): use guardrails for immediate intervention and evaluators for monitoring and improvement.

The practical rule is:

> Evals tell me whether a control works. They are not a substitute for the control.

For a production database, the reliable fix is not a stronger sentence in the system prompt. It is an application architecture in which the agent cannot mutate production without a narrow capability, an explicit approval, and a recoverable transaction. The eval proves that those controls remain effective across the cases that matter.

### Connect Offline Tests to Production

The harness has two sources of work. **Offline evals** run curated tasks with fixtures and known evidence before deployment. **Online evals** inspect sampled production traces for safety, coherence, drift, user friction, cost, and unfamiliar behavior when authoritative ground truth may not be available.

The scopes also differ. A **span** can test one tool decision. A **trace** can test a complete agent turn and its state changes. A **session** or **thread** can test whether the user's need was resolved across multiple turns, handoffs, corrections, and memory updates. Single-turn passes do not guarantee a successful conversation. For multi-turn systems, production prefixes can be replayed while allowing the candidate to generate the remaining turn or branch, which preserves realistic context without requiring one brittle scripted dialogue. [LangChain describes this run, trace, and thread separation together with the offline/online loop](https://www.langchain.com/resources/agent-evals).

<figure class="article-figure article-figure-plain">
  <img src="/images/illustrations/arize-ax-evaluation-harness.webp" width="1672" height="941" loading="lazy" decoding="async" alt="The Arize AX evaluation harness organized into evaluation inputs, evaluation execution, and evaluation actions.">
</figure>

The [Arize diagram](https://arize.com/blog/what-is-an-evaluation-harness/) is product-specific, but the architecture is general. An eval system needs a way to select evidence, execute the appropriate graders, persist results beside the source trace, and route those results into an action: human review, an experiment, an alert, a regression case, or a release gate.

The smallest useful operating loop is therefore:

1. Capture representative and high-information traces.
2. Review failures with the person authorized to define good behavior.
3. Minimize each important failure into an isolated, executable task.
4. Run repeated trials against the production baseline and candidate.
5. Grade outcomes, required trajectory contracts, and operational budgets.
6. Apply transparent release gates and deploy runtime controls for high-impact failures.
7. Monitor sampled production behavior and promote new failures back into the suite.

This is the point at which an eval stops being a score in a notebook. It becomes part of the engineering system that decides what can ship, explains why it failed, and remembers what the product has already learned.

## 9. Never Build a “God” Evaluator

A tempting evaluator prompt asks one model to inspect everything at once:

> Judge this response for accuracy, faithfulness, completeness, tone, safety, formatting, and actionability. Return pass or fail.

The result is compact, inexpensive, and nearly useless when it fails.

If the judge returns **fail**, I do not know whether the answer was factually wrong, unsupported by context, unsafe, badly formatted, rude, or simply not useful. If it returns **pass**, I do not know whether every dimension passed or whether excellent tone and formatting compensated for a serious factual error. The single label destroys the information I need for debugging.

This is the **god evaluator** anti-pattern: one evaluator attempts to represent every desirable property of the application and compresses them into one verdict.

The problem is deeper than explainability. Each dimension has a different evidence contract and often needs a different type of grader. Accuracy may require a verified reference or live system state. Faithfulness requires the context supplied to the agent. Safety may require the entire trajectory. Format may need only a deterministic schema validator. Tone and actionability require semantic judgment from the final response. One prompt cannot make missing evidence appear merely by listing more criteria.

Calibration also becomes incoherent. A safety evaluator should be tuned to minimize dangerous false passes. A tone evaluator can tolerate more disagreement. A format evaluator should be deterministic. A correctness evaluator for expert knowledge must be validated against domain experts. Combining all of them gives me one threshold with no clear meaning and one error rate that hides several different error distributions.

The better design is one evaluator per failure dimension:

| Evaluator | Question it answers | Evidence it needs |
| --- | --- | --- |
| **accuracy_eval** | Is the answer correct? | output and authoritative ground truth |
| **faithfulness_eval** | Is every material claim supported? | output and supplied context |
| **actionability_eval** | Can the user act on the response or complete the task? | user goal and final response |
| **safety_eval** | Did the run violate a policy or cross a prohibited boundary? | policy, output, actions, and trace |
| **format_eval** | Does the output satisfy the required interface? | output and schema |
| **tool_use_eval** | Did the agent select and use tools correctly? | expected constraints and tool-call trace |

Now a run can produce a diagnostic result:

**Accuracy: pass. Faithfulness: pass. Actionability: fail. Safety: pass. Format: pass.**

That result tells me where to investigate. It also lets me build a targeted dataset for actionability, change the relevant rubric, and measure whether the fix improves that dimension without moving the others.

An overall release decision can still exist, but it should be a transparent policy over the component results rather than another model opinion. Safety, authorization, or correctness may be hard gates. Tone may be monitored as a graded signal. Format may be enforced before an output reaches the user. The aggregation rule belongs to the product requirement; it should not be improvised inside the judge's hidden reasoning.

Separate evaluators also make the stack from Section 6 practical. `format_eval` may be code. `faithfulness_eval` may be an LLM supplied with retrieval context. `accuracy_eval` may combine a reference check with expert review. The dimensions can share infrastructure without sharing a verdict.

**One evaluator should answer one evaluation question.** That is what makes calibration possible, failures localizable, and improvements attributable to the part of the system that actually changed.

## 10. Who Evaluates the Evaluator?

Up to this point, evaluation has looked like the layer that tells me whether the agent is good. I run the application, pass its behavior to a grader, and receive a score. The score becomes the evidence I use to change prompts, compare models, block releases, and claim improvement.

But the moment that grader is another language model, the argument turns back on itself.

**I have used AI to judge AI. Why should I trust the judge?**

An LLM judge can misunderstand the rubric, miss a subtle failure, invent a reason for the wrong verdict, or apply the threshold differently after a model update. A plausible score is still a model output. Treating it as ground truth simply moves the original reliability problem one layer higher.

The evaluator must therefore become an evaluated system too.

<figure class="article-figure article-figure-plain">
  <img src="/images/illustrations/llm-judge-human-meta-evaluation.png" width="703" height="763" alt="The same output is evaluated by an LLM judge and a human, their verdicts are compared, and the results are organized into true positive, false positive, false negative, and true negative outcomes.">
</figure>

The mental model is simple. Give the same examples to the LLM judge and to qualified human reviewers. Treat the adjudicated human labels as the operational ground truth for the rubric. Then compare the two sets of verdicts.

For the definitions below, **positive means that the evaluator flags a failure**:

| Outcome | LLM judge | Human label | Meaning |
| --- | --- | --- | --- |
| **True positive (TP)** | Fail | Fail | The evaluator caught a real failure |
| **False positive (FP)** | Fail | Pass | The evaluator raised a false alarm |
| **False negative (FN)** | Pass | Fail | The evaluator missed a real failure |
| **True negative (TN)** | Pass | Pass | The evaluator correctly accepted a good run |

The naming matters less than the consequences. A false positive creates noise. Engineers investigate a run that was acceptable, dashboards look worse than the product, and good changes may be blocked. A false negative is a failure that escapes detection. The eval says the system is safe to ship when a human reviewer would have rejected it.

For many agent applications, false negatives are the more dangerous error. A faithfulness judge that misses an unsupported claim, a safety judge that passes a prohibited action, or a tool-use judge that overlooks a production write can create confidence precisely where caution was required.

### Precision: Can I Trust an Alert?

**Precision = TP / (TP + FP)**

Precision asks: **Of all the runs the evaluator flagged, how many were real failures?**

High precision means a failed eval is usually worth investigating. Low precision means the judge generates many false alarms. The team may begin ignoring failures, manually rerunning cases until they pass, or weakening the threshold merely to make the dashboard usable.

Precision is especially important when every alert creates expensive human review. If a domain expert must spend thirty minutes adjudicating each flagged medical or legal response, a noisy evaluator consumes scarce expertise without proportionate value.

### Recall: How Many Failures Did I Catch?

**Recall = TP / (TP + FN)**

Recall asks: **Of all the real failures in the human-labeled set, how many did the evaluator catch?**

High recall means few failures escape. Low recall means the evaluator looks calm because it does not notice enough. For safety, authorization, privacy, and other high-severity contracts, recall is often the first priority: a false alarm can be reviewed, while a false pass may reach production.

Precision and recall pull attention toward different risks. Tightening a threshold may catch more failures and improve recall while also flagging more acceptable runs and reducing precision. Loosening it may make each alert more credible while allowing more failures through. There is no universally correct balance. The consequence of each error should determine the threshold.

Consider a human-labeled set of 100 runs containing 20 real failures. The judge catches 15 of them, misses 5, and incorrectly flags 10 good runs. Its overall accuracy is 85%, which sounds strong. But its precision is only 60%, and its recall is 75%. One in four real failures escapes, and two in five alerts are noise. The confusion matrix tells me much more than the attractive 85% headline.

Airbnb offers a useful operational target: calibrate a virtual judge against expert labels until agreement reaches the high-80s to 90s, using measures such as Cohen's kappa or Krippendorff's alpha where appropriate. That is a **starting gate, not a certificate**. Agreement can look high on an imbalanced set where almost everything passes, so I still need the confusion matrix, per-class errors, and held-out cases. The target is valuable because it makes trust conditional on measured human alignment; the error analysis determines whether that alignment is safe enough for the intended use.

### Meta-Evaluate Each Evaluator

Section 9 separated the god evaluator into dimensions such as accuracy, faithfulness, actionability, safety, and format. Each model-based evaluator now needs its own labeled test set.

A practical meta-evaluation loop is:

1. **Build a calibration set.** Include ordinary passes, known failures, difficult boundary cases, and examples from the production slice where the evaluator will run.
2. **Collect human labels.** Give reviewers the same rubric and the evidence required by that rubric. Use domain experts where correctness depends on specialized knowledge.
3. **Adjudicate disagreements.** If qualified humans interpret the boundary differently, the rubric or requirement may still be underspecified.
4. **Run the evaluator blind.** Freeze its prompt, model version, evidence, and threshold before comparing its verdicts with the human labels.
5. **Measure its errors.** Inspect the confusion matrix, Precision, Recall, and—most importantly—the actual false-positive and false-negative cases.
6. **Tune on calibration data, then test on held-out data.** Otherwise I can overfit the evaluator to the examples used to repair it.
7. **Repeat after changes.** A new judge model, rubric, prompt, data distribution, or product behavior can invalidate the previous calibration.

This process is **meta-evaluation**: an evaluation of the evaluator. A 2026 analysis of agreement metrics for LLM judges recommends reporting the confusion matrix alongside metrics such as Precision and Recall, and explicitly documenting how ties, invalid outputs, and abstentions are handled. Those choices change what the reported number means. [The paper provides a reporting checklist for judge–human validation](https://arxiv.org/abs/2606.00093).

Human labels are not metaphysical truth. Reviewers can make mistakes, disagree, or lack the evidence required for the decision. The goal is an operational ground truth: labels produced by the people authorized to define the product boundary, using a clear rubric and an adjudication process. If humans cannot label the cases consistently, training a model judge to imitate them will not repair the ambiguity.

This is also how automated judges should be positioned in practice. For GDPval, OpenAI defines pairwise human-expert preference as the grading standard and describes the LLM-based judge as a way to obtain a rough estimate of model performance. [The expert process remains the standard against which automation is interpreted](https://evals.openai.com/gdpval/grading).

The human layer does not need to review every production run forever. Humans define the rubric, create the labeled calibration set, adjudicate difficult disagreements, audit drift, and periodically test whether the automated judge still represents their decisions. Automation scales the judgment only after that relationship has been measured.

This is the turning point in the evaluation stack. The LLM judge is not an authority outside the system. It is a classifier with inputs, labels, thresholds, blind spots, and measurable error.

**An eval suite is not trustworthy because it contains an evaluator. It becomes trustworthy when the evaluator's own failures are visible, measured, and bounded.**

## 11. Golden Datasets: Where Production Failures Go to Become Tests

**Today's production incident should become tomorrow's test case.**

A production failure is expensive data. A real user found a path through the system that the requirements, examples, graders, and pre-release testing did not cover. The run contains evidence about the user's language, the environment, the available tools, and the sequence of decisions that produced the failure.

If I fix the prompt and close the incident, I have repaired one moment. If I preserve the case as a reproducible test, I have added the failure to the system's memory.

<figure class="article-figure article-figure-plain">
  <img src="/images/illustrations/production-failure-golden-dataset-regression-test.png" width="1449" height="219" alt="A production failure moves through human review and labeling into a golden dataset, where it becomes a regression test.">
</figure>

This is the engineering role of a **golden dataset**: a curated, versioned collection of cases whose expected behavior has been reviewed closely enough to support repeatable decisions. It is not “golden” because every answer is timeless or because one exact string is the only acceptable output. It is golden because the examples carry explicit product judgment and provenance.

[OpenAI describes a golden set as a living, authoritative reference for expert judgment about what good looks like](https://openai.com/index/evals-drive-next-chapter-of-ai/), then recommends feeding reviewed production logs and costly or ambiguous cases back into that set. Anthropic gives similarly concrete advice: start with 20–50 tasks drawn from real failures, inspect the bug tracker and support queue, and convert user-reported failures into test cases. The practical message is the same: [the dataset should grow from observed behavior and expert review, not from prompt generation alone](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents).

### A Golden Dataset Is Institutional Memory

A folder of random prompts is a collection of inputs. A golden dataset records what the organization has learned about the behavior it needs.

Over time, a useful set encodes:

- **Domain judgment:** distinctions that only product owners, operators, or subject-matter experts can reliably make.
- **Past production failures:** incidents, support escalations, bad tool calls, misleading answers, and near misses that must not recur.
- **Edge and boundary cases:** rare combinations, ambiguous requests, conflicting instructions, missing information, and threshold behavior.
- **Real user language:** abbreviations, misspellings, indirect requests, local terminology, adversarial phrasing, and the messy context absent from synthetic prompts.
- **High-severity scenarios:** cases that may be uncommon but carry disproportionate safety, financial, privacy, or trust consequences.
- **Negative examples:** situations in which the agent should abstain, ask a question, refuse, avoid a tool, or leave state unchanged.
- **Environment state:** retrieved documents, tool responses, permissions, database fixtures, timestamps, and other conditions required to reproduce the run.
- **Acceptable variation:** the contracts an answer must satisfy without pretending that every good answer must match one reference sentence.
- **Failure ownership:** the failure dimension, the first broken contract, severity, affected user segment, and the component expected to change.
- **Provenance:** where the case came from, who reviewed it, which policy or requirement defined the label, and which version of that truth was used.

That last item matters. A case derived from a pricing policy, database snapshot, or current fact can become wrong when its source changes. Without a timestamp and source version, a regression failure may mean that the agent regressed—or merely that the test fossilized an old world.

The label should also preserve the dimensions from Section 9. A case is more useful when it says **faithfulness failed because claim 3 lacked support** or **tool authorization failed before the final answer** than when it stores only a generic **fail**. The purpose is not just to catch a regression. It is to make the regression localizable.

### Turn an Incident into a Reproducible Case

The diagram compresses a small but important pipeline:

1. **Capture the run.** Preserve the input, output, trace, tool results, retrieved context, relevant state, and product version before the evidence disappears.
2. **Review it with a human.** Confirm that the behavior was actually wrong, identify the first broken contract, and determine the desired behavior. A user complaint is a signal, not yet a label.
3. **Minimize and sanitize the case.** Remove irrelevant noise, redact sensitive data, and retain the smallest environment that still reproduces the failure.
4. **Attach labels and graders.** Record the expected outcome, acceptable alternatives, failure dimensions, severity, and the cheapest evaluator that can test each contract.
5. **Add it to a versioned dataset.** Give the case an owner and provenance so future changes can be reviewed rather than silently overwriting history.
6. **Run it as a regression test.** Execute the case against every meaningful prompt, model, retrieval, tool, policy, and orchestration change.

Not every production trace belongs in the golden set verbatim. Duplicate failures should be clustered. Sensitive data should be removed or replaced with a fixture. A long conversation may need to be reduced without deleting the earlier event that caused the later failure. And if reviewers cannot agree on the expected result, the case belongs in an adjudication queue until the contract is clear enough to test.

This curation is what turns an anecdote into an engineering artifact.

### Golden Does Not Mean Representative

A regression set and a random production sample answer different questions.

The golden set asks: **Do the behaviors we explicitly care about still work, including failures we have already paid to discover?** It will often oversample rare, difficult, and severe cases on purpose. Its aggregate pass rate therefore should not be presented as the expected success rate for production traffic.

A sampled production set asks: **How is the system performing on the distribution users are generating now?** It reveals common behavior, changing language, new workflows, and distribution drift. Random sampling can estimate frequency; targeted golden cases protect known contracts.

I need both. The golden set prevents known failures from returning. Production sampling discovers failures the set does not yet know to contain. Anthropic explicitly positions automated evals, monitoring, user feedback, transcript review, and human studies as complementary layers: production reveals unanticipated behavior, while automated tests make those lessons reproducible before the next release.

There is another reason to keep separate slices. If I repeatedly tune a prompt against the same visible golden cases, I can overfit to the suite while failing on semantically equivalent inputs. A healthy evaluation system keeps a stable regression set, a fresh holdout set for honest comparison, and production-derived challenge cases that continue to expand the boundary.

### The Experiment Flywheel

Once the dataset exists, experiments stop being collections of memorable demos. Every candidate change can run against the same cases and the same failure dimensions.

A prompt change may fix a refusal case while reducing actionability elsewhere. A new retriever may improve current correctness while lowering source quality. A stronger model may solve more tasks but use an expensive tool unnecessarily. Because the cases and graders remain stable, the experiment can show not only whether the aggregate moved, but which contracts improved and which regressed.

The winning candidate then moves toward production. Monitoring and user feedback expose new failures. Humans review the consequential or ambiguous cases. Those labels enter the dataset, and the expanded dataset shapes the next experiment.

**Production → review → dataset → regression test → experiment → production.**

This is a flywheel because each cycle increases the organization's ability to specify and test its own product. OpenAI describes the result as a context-specific dataset that compounds over time and is difficult to copy; its internal data agent offers a concrete implementation, using curated question–answer pairs with manually authored golden SQL as continuously running unit tests for regressions. [The evals let the team iterate while protecting analytical behavior it already knows must remain correct](https://openai.com/index/inside-our-in-house-data-agent/).

The dataset does not merely measure the agent. It accumulates the domain decisions, operational scars, and product taste that define what the agent is for.

**A failure that is fixed but not preserved as a test is only temporarily understood.**

## 12. Case Study: Airbnb's Eval-Driven Development Loop

Airbnb's account is useful because it connects the pieces in this article into one operating loop. The company reports using LLM-powered features across review highlights, customer support, and communication for guests and hosts. Product teams define their own criteria and workflows, while a central infrastructure team supplies common tooling and shares evaluation practices across domains. That division is important: **evaluation infrastructure can be centralized, but the definition of good behavior remains product-specific.**

Airbnb explicitly describes its end-to-end walkthrough as a **fictionalized and simplified version of a real use case**, not as a controlled experiment or a universal production recipe. The scenario is a travel-support assistant that answers questions about platform policies. Its value is therefore methodological: it shows how a team can move from raw behavior to trustworthy monitoring without pretending that one generic score measures the product.

<figure class="article-figure article-figure-plain">
  <img src="/images/illustrations/airbnb-eval-driven-development-workflow.png" width="1525" height="450" loading="lazy" decoding="async" alt="Airbnb's eval-driven development loop: explore, turn failures into evals, calibrate, improve the system, scale, monitor production, and feed discoveries back into exploration.">
  <figcaption>Airbnb's eval-driven development workflow, redrawn by the author from the process described in its engineering case study.</figcaption>
</figure>

### Explore and Turn Failures Into Evals

The walkthrough begins with 100 prototype inputs and manual review of every output. The review finds four distinct failure classes: 15 unsupported policy details, 8 answers that are correct but too verbose, 5 refusals of valid questions, and 3 malformed JSON responses. These counts are illustrative findings inside Airbnb's simplified walkthrough, not benchmark rates for its products.

The next move is not to average those failures into a single quality score. The team attaches a suitable evaluator to each contract:

| Observed failure | Contract | Evaluator |
| --- | --- | --- |
| Malformed JSON | The response must satisfy the output schema | Programmatic schema check |
| Excessive length | The response must remain within a defined bound | Programmatic length check |
| Unsupported policy detail | Claims must follow from the supplied policy evidence | Faithfulness virtual judge |
| Unnecessary verbosity | The answer must be concise without losing required information | Conciseness virtual judge |
| Over-refusal | Valid, answerable requests should not be rejected | Separate evaluator or expert review |

This is failure-driven metric design in concrete form. Code handles explicit structure. Narrow model graders handle semantic boundaries. Subject-matter experts provide the labels that make those boundaries operational.

### Calibrate the Evaluator, Then Improve the Product

In Airbnb's example, a product or domain expert labels 60 cases, including bad outputs, to form a golden set. The initial faithfulness judge agrees with those labels 78% of the time. Disagreement analysis shows a specific defect: the judge treats accurate paraphrases as unsupported. After the rubric and few-shot examples are refined, agreement rises to 88%.

The score increase is not the most important part. The disagreement revealed *how the evaluator was wrong*, and the labeled examples made the repair testable. Airbnb's broader guidance is to begin with tens rather than thousands of expert-labeled rows, include negative examples, stop when experts cannot agree on the label, and prefer roughly 3–5 well-calibrated virtual judges over 20–30 noisy metrics. One evaluator should own one correctness dimension, matching the anti–god-evaluator principle from Section 9.

Only after the judge becomes credible does the team use it to improve the actual system. In the walkthrough, the product change is to retrieval, and faithfulness failures fall. Airbnb also recommends changing one experimental variable at a time: hold the model fixed while varying the prompt, then hold the prompt fixed while varying the model, and evaluate serving changes separately. That discipline makes attribution possible. A stable judge narrows the candidates; samples from strong candidates can then expose new evaluator weaknesses. The product and its evaluators improve together, but remain separately versioned systems with separate failure modes.

### Scale Offline, Then Mirror the Evals in Production

The final step expands the illustrative evaluation from the small golden set to 5,000 examples. The walkthrough then samples 5% of de-identified live traffic each day, applies programmatic checks and virtual judges, sends flagged outputs to human review, and holds a weekly product review. New production failure modes become new evals, which drive the next system change.

Those figures are Airbnb's walkthrough parameters, not defaults every team should copy. Sampling should follow traffic volume, risk, privacy constraints, evaluator cost, and the rarity of the failures being sought. Airbnb also notes that live data is continuously sampled with privacy-preserving techniques, robustly de-identified before human review, and purpose-limited to safety and quality assurance. Production evaluation is a data-governance system as much as a scoring system.

The general loop is the durable lesson:

<p class="concept-equation">Explore → Encode Failures → Calibrate → Improve → Scale → Monitor → Explore</p>

This closes a gap that offline-only evaluation leaves open. A pre-release suite protects the behaviors already known. Production monitoring discovers changes in language, traffic, policies, and failure modes that the suite cannot predict. Human review decides which discoveries represent real product failures. The resulting cases return to the dataset, so evaluation becomes a development practice rather than a one-time launch check.

The [Airbnb engineering case study](https://airbnb.tech/ai-ml/eval-driven-development-lessons-from-evaluating-genai-at-scale/) does not supply a universal formula for evals; it supplies a concrete organizational pattern. Start close to the data, make product judgment explicit, automate only calibrated distinctions, improve the system under controlled comparisons, and keep learning after deployment.

## 13. Conclusion

An agent does not “work” merely because it produced a convincing answer, completed a few demonstrations, or achieved a high aggregate score. It works only to the extent that its important behaviors have been specified, observed, and tested across the situations in which failure matters.

A defensible eval system begins with the application rather than the model, and with concrete failures rather than convenient metrics. It records the trajectory, identifies the first broken contract, preserves the case in a reproducible dataset, and attaches the cheapest evaluator capable of judging that contract reliably. It repeats trials to measure consistency, separates agent failures from harness failures, and compares candidate changes with a stable baseline. When the evaluator itself is probabilistic, its errors are measured against human judgment rather than accepted on authority.

No single score can certify an agent. The evidence is a collection of bounded claims: which system was tested, on which situations, according to which requirements, using which sources of truth, across how many trials, and with what known evaluator error. Release gates turn those claims into engineering decisions. Guardrails, permissions, approvals, and recoverable state transitions enforce the boundaries that cannot be left to model behavior.

The objective is not to eliminate nondeterminism. It is to make its consequences visible and manageable.

**An agent earns trust when its failures can be found, explained, reproduced, contained, and prevented from quietly returning.**
