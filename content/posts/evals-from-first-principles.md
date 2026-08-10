---
title: "Evals from First Principles: How to Measure, Debug, and Improve AI Agents"
date: 2026-08-10
draft: false
---

## 1. Introduction

At 9:12 on a Monday morning, a traveler asks an AI agent to move her evening flight to the next morning. She gives it three constraints: keep the seat in business class, do not change the destination, and do not confirm anything if the fare difference is more than $200.

The agent finds the correct reservation. It searches the correct route. It selects a flight that leaves at 8:10 the next morning, confirms that a business-class seat is available, and receives a fare difference of $1,842 from the booking system. Its reasoning trace even notes the user's $200 limit.

Then it calls the right tool with the wrong argument:

```text
change_flight(
    booking_id="Q7M4TZ",
    new_flight="VN210",
    confirm=true
)
```

The API returns `200 OK`. The reservation is updated. The agent replies, "Your flight has been successfully changed."

From the perspective of every individual component, much of this interaction looks healthy. Retrieval found the right record. The search tool returned a valid option. The booking tool executed exactly what it was asked to execute. The final answer was fluent and factually consistent with the new reservation. Yet the system failed at the only point that mattered: it took an expensive, irreversible action that the user had explicitly forbidden.

This scenario is constructed, but the failure pattern is ordinary. An agent can fail by choosing the wrong tool, or by choosing the right tool with the wrong arguments. It can retrieve excellent evidence and synthesize it badly. It can produce the correct answer at an absurd cost. A relevant skill may never trigger; an irrelevant one may trigger instead. The agent may ignore a tool result, trust stale memory over fresh evidence, stop one step too early, retry forever, violate a policy midway through an otherwise valid trajectory, or reach a good final answer through a process that should never be allowed in production.

Worse, rerunning the same request may not reproduce the same mistake. The next run might set `confirm=false`. Another might ask the user for approval. Another might select a different flight, call a different sequence of tools, or spend ten times as many tokens before arriving at the same answer. The behavior I need to understand is not a single output. It is a distribution of possible trajectories, including the rare ones that are costly, unsafe, or simply strange.

I would never ship a payment service because I tried it a few times and it looked fine. I would want tests for the expected path, the boundary conditions, and the failures I had already encountered. I would want logs that tell me which branch ran, inputs that let me reproduce the bug, and regression tests that prevent it from quietly returning. Deterministic software can still surprise me, but software engineering has spent decades building practices that turn those surprises into inspectable problems.

With LLM applications, that standard often disappears. I type a few prompts into a chat window, read the responses, change a sentence in the system prompt, and try again. If the outputs feel better, I call the change an improvement. If five examples pass, I start to believe the system works. This is vibe checking, and for many LLM applications it is still the entire test suite.

Vibe checking is useful while exploring an idea. It is fast, intuitive, and often the first way I notice that something is wrong. But it cannot tell me how often the agent violates a constraint, which step caused the violation, whether a prompt change fixed the underlying failure or merely moved it, or what else regressed while I was watching my favorite examples. A convincing demo is evidence that a system *can* work. It is not evidence that the system works reliably.

An AI agent should be treated as a nondeterministic software system. The goal of evals is not merely to assign it a score, but to turn failures into observable, reproducible, and eventually preventable behavior.

That changes the question. I am no longer asking only whether the final answer looks good. I am asking what the agent saw, which decisions it made, which tools it called, what those calls cost, where the trajectory diverged, and whether the same class of failure will be caught the next time it appears.

**If the same request can produce ten plausible trajectories, and only one of them silently costs the user $1,842, what would it actually mean to say that the agent “works”?**
