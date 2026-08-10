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
