---
title: "Evals from First Principles: How to Measure, Debug, and Improve AI Agents"
date: 2026-08-10
draft: false
---

## 1. Introduction

In July 2025, SaaStr founder Jason Lemkin was nine days into building an application with Replit Agent when its batch processing stopped working. He asked the agent what had happened. The answer eventually exposed something much worse than a broken job: during an explicit code freeze, the agent had deleted the application's production database. The deleted data included records for 1,206 executives and more than 1,196 companies. Lemkin [documented the incident publicly](https://x.com/jasonlk/status/1946069562723897802) as it unfolded.

The agent had not merely missed an implied preference. Lemkin had repeatedly instructed it not to make changes without permission. In the account it produced after the deletion, the agent acknowledged that it had ignored those instructions and destroyed live data anyway. It had enough context to restate the boundary after crossing it, but not enough control to respect the boundary before acting. [Contemporary reporting preserved the exchange](https://www.tomshardware.com/tech-industry/artificial-intelligence/ai-coding-platform-goes-rogue-during-code-freeze-and-deletes-entire-company-database-replit-ceo-apologizes-after-ai-engine-says-it-made-a-catastrophic-error-in-judgment-and-destroyed-all-production-data), including the agent's own description of the sequence.

That description is not a root-cause analysis. A model explaining its previous behavior is still generating text, not exposing a reliable record of its internal causes. The confession sounds precise and self-aware, but it does not tell me which instruction lost priority, why the destructive action remained available, what context the agent saw at that step, or whether another run would make the same decision.

Replit CEO Amjad Masad later confirmed that an agent in development had deleted production data and called the outcome unacceptable. Replit responded with product changes, including automatic separation between development and production databases, staging environments, and restore capabilities. The response matters because it moved beyond asking the model to be more careful. It changed the system around the model so that the same class of failure would become harder to repeat. [Masad described those measures publicly](https://x.com/amasad/status/1946986468586721478).

The database deletion is dramatic, but it is only one point in a much larger failure surface. An agent can choose the wrong tool, or choose the right tool with the wrong arguments. It can retrieve excellent evidence and synthesize it badly. It can produce the correct answer at an absurd cost. A relevant skill may never trigger; an irrelevant one may trigger instead. The agent may ignore a tool result, trust stale memory over fresh evidence, stop one step too early, retry forever, violate a policy midway through an otherwise valid trajectory, or reach a good final answer through a process that should never be allowed in production.

Rerunning the same task may not reproduce the same mistake. Another trajectory might respect the code freeze. It might inspect the database without modifying it, ask for permission, stop when it encounters uncertainty, or fail in an entirely different way. The behavior I need to understand is not a single output. It is a distribution of possible trajectories, including the rare ones that are costly, unsafe, or simply strange.

I would never ship a database migration because I tried it a few times and it looked fine. I would want tests for the expected path, the boundary conditions, and the failures I had already encountered. I would want logs that tell me which branch ran, inputs that let me reproduce the bug, and regression tests that prevent it from quietly returning. Deterministic software can still surprise me, but software engineering has spent decades building practices that turn those surprises into inspectable problems.

With LLM applications, that standard often disappears. I type a few prompts into a chat window, read the responses, change a sentence in the system prompt, and try again. If the outputs feel better, I call the change an improvement. If five examples pass, I start to believe the system works. This is vibe checking, and for many LLM applications it is still the entire test suite.

Vibe checking is useful while exploring an idea. It is fast, intuitive, and often the first way I notice that something is wrong. But it cannot tell me how often the agent violates a constraint, which step caused the violation, whether a prompt change fixed the underlying failure or merely moved it, or what else regressed while I was watching my favorite examples. A convincing demo is evidence that a system *can* work. It is not evidence that the system works reliably.

An AI agent should be treated as a nondeterministic software system. The goal of evals is not merely to assign it a score, but to turn failures into observable, reproducible, and eventually preventable behavior.

That changes the question. I am no longer asking only whether the final answer looks good. I am asking what the agent saw, which decisions it made, which tools it called, what those calls cost, where the trajectory diverged, and whether the same class of failure will be caught the next time it appears.

**If an agent can look competent across a long session and then erase production data in seconds, what evidence would justify saying that it “works”?**
