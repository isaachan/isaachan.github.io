---
title: AI Platform Primitives
description: A case story about how we defined reusable AI platform primitives across departments - task, context, tool, model, policy, evaluation, and workflow.
categories:
  - software
tags:
  - AI
  - Platform
  - Architecture
  - Context Engineering
  - Evaluation
  - Workflow
---

Recently, I had several rounds of discussions to define the enterprise AI engineering platform. The starting point looked familiar: every department wanted to adopt AI, but each team was still asking for the same wrappers, retrieval logic, access control, logging, and evaluation hooks. After a few meetings, it became clear that the platform was not just model access. The real platform value was the set of shared primitives that made the safe path the easy path.

At first, it was tempting to describe the platform as a unified inference API and stop there. That would have been neat on paper, but it would only move the duplication somewhere else. Teams would still rebuild prompt scaffolding, tool adapters, policy checks, and eval pipelines on their own. So we changed the framing: instead of asking what endpoint to expose, we asked what reusable abstractions should sit between application teams and raw models or enterprise systems.

## Overview

![AI Platform overview](/images/2026-04-11-ai-platform-primitives/overview.svg)

### 1. Task abstraction

We started with task abstraction because teams were already describing the same work in different words: 
- summarize, 
- classify, 
- explain, 
- draft, 
- review, 
- analyze impact, 
- retrieve and answer. 

The platform should turn those into standard task contracts instead of leaving every team to invent its own prompt shape.

That abstraction carries the `input type`, `output schema`, `review requirement`, `allowed toolset`, and default `routing policy`. It does not force the same wording everywhere. It just makes the task boundaries clear enough that teams can reuse them.

### 2. Context abstraction

Context engineering was the most duplicated layer, and also the easiest place to make subtle mistakes. So we treated context as a *reusable package* rather than a prompt fragment.

In our discussion, the obvious packages were 
- code context, 
- requirement context, 
- defect context, 
- release evidence, 
- traceability context

Each package needs to define where the `data` comes from, how `retrieval` works, how `freshness` is handled, which `permissions` apply, and how the final `context` is shaped for the model.

This is the part that usually gets rebuilt badly. Once a team has a half-working retrieval pipeline, it is hard to share. A context abstraction gives us one place to keep that logic honest.

### 3. Tool abstraction

The next layer was tool abstraction. No team should need to build direct adapters to Git, CI/CD, requirements systems, or test systems every time they want an AI workflow.

We described the tools as stable capabilities: 
- `get changed files`, 
- `get linked requirements`, 
- `fetch the latest failed tests`, 
- `create a draft work item`, 
- `read defect history`, 
- `append review evidence`

The important part is not the verb list itself. The important part is that each tool is typed, scoped, permission-aware, and auditable.

Once those rules are inside the platform, application teams can compose workflows without rebuilding their own security layer.

### 4. Model abstraction

Model abstraction was the easiest to explain and the easiest to overstate. Raw model access is useful, but it is not enough to make the platform reusable.

What teams actually need is a unified inference API, structured output support, routing across model tiers, fallback behavior, token and cost telemetry, and deployment awareness for cloud versus private models. That lets product teams build against platform semantics instead of vendor-specific SDK behavior.

### 5. Policy abstraction

Policy was the place where cross-department discussions became real. If every team interprets policy differently, the platform fragments immediately.

So we framed policy as an inherited runtime behavior, not a document. We talked about permissions for 
- read-only workflows, 
- draft-generation workflows, 
- action-taking workflows, 
- high-sensitivity data workflows, and 
- safety-relevant artifact workflows 
  
Each policy sets the data classes, tool classes, approval requirements, logging requirements, and rollout stage.

That way, application teams inherit controls instead of recreating them in every project.

### 6. Evaluation abstraction

Evaluation is usually the first thing teams say they will add later, which is another way of saying they may never add it. That is why we wanted it as a platform primitive.

The shared layer should include task-level eval harnesses, golden datasets or rubric sets, scoring pipelines, regression hooks, side-by-side model comparison, and online feedback metrics. If the platform makes evaluation easy, teams can improve without inventing a separate measurement framework each time.

### 7. Workflow abstraction

The last primitive was workflow abstraction. This is where the platform stops being a pile of building blocks and starts looking like a system teams can actually use.

We talked about reusable templates such as 
- `retrieve -> generate -> review`, 
- `analyze -> route -> approve`, 
- `summarize -> compare -> escalate`, and 
- `detect issue -> gather evidence -> draft action` 

These templates should include context assembly, tool sequencing, approval checkpoints, observability hooks, and failure handling.

That is the part that turns AI capability into something repeatable across departments.

## What we want teams to say

By the end of the discussion, we wanted an application team to be able to say something like:

```
 I need a requirement impact analysis task, using traceability context, with read-only plus draft-ticket tools, under a review-required policy, and evaluated with the shared artifact-analysis harness
```

If the platform can supply the connectors, retrieval logic, access control, logging, model routing, and approval boundaries underneath that request, then the team can stay focused on the product problem instead of rebuilding the same scaffolding.

## What I would avoid

The main thing I would avoid is a platform that only exposes raw model calls. That looks simple, but it pushes all the hard work back to the teams. I would also avoid a monolithic copilot that forces everyone into one user experience, and I would avoid letting every team define its own prompt and eval lifecycle.

The better split is simple: standardize the how, and let product teams specialize the why and where.

## Closing thought

That was the conclusion we kept coming back to in the discussions. The platform should standardize execution contracts, context assembly, policy inheritance, observability, and evaluation. It should leave room for teams to customize user experience and domain logic above that layer.

That is not a glamorous answer, but it is the one that scales without turning every department into its own platform team.
