---
title: "Evals from First Principles: How to Measure, Debug, and Improve AI Agents"
date: 2026-08-10
draft: false
---

## 1. Introduction

In July 2025, SaaStr founder Jason Lemkin was nine days into building an application with Replit Agent when its batch processing stopped working. He asked the agent what had happened. The answer eventually exposed something much worse than a broken job: during an explicit **code freeze**, the agent had deleted the application's production database. The deleted data included records for 1,206 executives and more than 1,196 companies. Lemkin [documented the incident publicly](https://x.com/jasonlk/status/1946069562723897802) as it unfolded.

The agent had not merely missed an implied preference. Lemkin had repeatedly instructed it not to make changes without permission. In the account it produced after the deletion, the agent acknowledged that it had ignored those instructions and destroyed live data anyway. It had enough context to restate the boundary after crossing it, but not enough control to respect the boundary before acting. [Contemporary reporting preserved the exchange](https://www.tomshardware.com/tech-industry/artificial-intelligence/ai-coding-platform-goes-rogue-during-code-freeze-and-deletes-entire-company-database-replit-ceo-apologizes-after-ai-engine-says-it-made-a-catastrophic-error-in-judgment-and-destroyed-all-production-data), including the agent's own description of the sequence.

That description is not a **root-cause analysis**. A model explaining its previous behavior is still generating text, not exposing a reliable record of its internal causes. The confession sounds precise and self-aware, but it does not tell me which instruction lost priority, why the destructive action remained available, what context the agent saw at that step, or whether another run would make the same decision.

Replit CEO Amjad Masad later confirmed that an agent in development had deleted production data and called the outcome unacceptable. Replit responded with product changes, including automatic separation between development and production databases, staging environments, and restore capabilities. The response matters because it moved beyond asking the model to be more careful. It changed the system around the model so that the same class of failure would become harder to repeat. [Masad described those measures publicly](https://x.com/amasad/status/1946986468586721478).

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
  <img src="/images/illustrations/agent-application-boundary.png" alt="An input flows into an agent and then to an output, while the agent interacts with model calls, retrieval, tools, skills, and state.">
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
  <img src="/images/illustrations/eval-task-dataset-scorer.png" alt="An eval branches into Task, Dataset, and Scorer, representing the system being run, the situations being tested, and the definition of success.">
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

Coverage determines the scope of every conclusion. A task that scores 95 percent on fifty easy examples is not "95 percent good." It achieved that score on that particular mixture of situations. Change the mixture, and the meaning of the number changes.

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
  <img src="/images/illustrations/agent-router-downstream-workflows.png" alt="A query enters a router that can choose semantic search over documents, text-to-SQL over databases, or REST API calls to external services.">
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

Consider a research agent asked to compare three competitors and save the finished report to a shared workspace. The agent searches the right sources, retrieves relevant documents, resolves conflicting claims, and generates a strong report. It then calls the file-writing tool with the correct content and a plausible destination. The operating system rejects the write because the service account lacks permission for that directory. No report appears in the workspace.

From the user's perspective, the task failed. If the only recorded output is an empty artifact or a generic message such as "I could not complete the request," I might conclude that the model failed to research or write the report. I might change the prompt, replace the model, add more documents, or rewrite the synthesis scorer. None of those changes touches the actual defect.

The research was correct. The synthesis was correct. The failure occurred in the **delivery layer**.

This distinction is easy to miss because the final output compresses an entire execution into one symptom. **The response was wrong** is an observation about the boundary of the application. It is not a root cause.

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

If frequency is an estimated probability and severity is an estimated cost per occurrence, their product approximates **expected harm**. If I use ordinal scales such as 1 to 5, the result is only a ranking heuristic, not a precise economic quantity. Either way, the formula forces two separate questions: how likely is this failure, and how bad is it when it happens?

Suppose a report agent produces an extra blank line in 15 percent of runs. The defect is common, visible, and low severity. A different agent writes to the wrong customer's workspace in 0.1 percent of runs. That defect is rare, but its privacy and trust consequences are severe. Fixing the formatting issue first because its count is larger would optimize the dashboard rather than the product's risk.

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

Observability does not replace evaluation. A trace can show me exactly what happened without telling me whether that behavior was acceptable. Evaluation does not replace observability either. A failing score can tell me that a contract was violated without showing me why. I need the judgment of an eval and the causal evidence of a trace.

Only then can I move from **the response was wrong** to a claim I can act on: the router selected semantic search instead of billing, the tool received the wrong account ID, the model ignored retrieved evidence, or the report was correct but never reached the user.

That is the real prerequisite for a useful eval system. Before I measure failures, I must make them visible. Before I prevent them, I must make them reproducible.
