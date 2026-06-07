---
title: AI Platform Primitives, After Adoption Started
description: A follow-up case story on how the primitive model changed after feedback, why workflow and agent were merged into one orchestrator layer, and how three enterprise cases shaped the platform contract.
categories:
  - software
tags:
  - AI
  - Platform
  - Architecture
  - Workflow
  - Agent
  - Context Engineering
---

Since I published the earlier post about AI platform primitives, we did not just keep discussing it on slides. We started adoption. And once the platform started touching real cases, some of the nice-looking ideas from the first draft became a little too clean.

The feedback was consistent: the primitives were useful, but the original framing was too flat. In particular, `workflow` was not just "a fixed template", and `agent` was not a completely separate planet. The more concrete the cases became, the more we had to treat the primitives as layered, not peer-level nouns.

## What changed after the first round of feedback

The biggest change was this: we stopped describing the platform as a list of primitives and started describing it as a stack.

![AI platform primitives stack](/images/stack.svg)

The important shift is that the layers do different jobs.

- **Capability** is what the user understands.
- **Workflow / Agent** is what the platform executes.
- **Task** is the reusable unit inside the orchestration.
- **Context Package / Tool** is how the task gets evidence and acts on systems.

That sounds small, but it changed the design a lot. In the earlier version, we spent too much time trying to decide whether something was a workflow or an agent. After adoption started, we realized that distinction is useful, but only inside one layer. From the platform point of view, they both sit in the same orchestrator slot.

So we changed the wording a little:

- `workflow` is the more deterministic side of the orchestrator layer.
- `agent` is the more adaptive side of the same layer.
- both are published primitives,
- both need contracts,
- both need policy,
- both need audit,
- and both should not become free-form black boxes.

That helped us stop arguing about names and start discussing behavior.

## Three real cases that forced the model to become concrete

We did not start from abstract theory. We started from three cases that people actually wanted to use.

| Case | What it needed | Main primitives |
| --- | --- | --- |
| Requirement Impact Analysis | Understand a requirement change and find impacted components, tests, and risks | workflow, tasks, context packages, tools, evaluation |
| CI Failure Triage | Explain why a pipeline failed and suggest next actions | workflow, tasks, tools, policy, draft output |
| Release Readiness | Aggregate evidence and produce a release summary | orchestrator, context packages, tools, review gate |

The first case ended up driving most of the platform design.

## Requirement Impact Analysis changed the most

At first, requirement impact analysis sounded like a single task. It is not. Once we wrote the actual steps down, it became obvious that it is a small orchestration problem.

The user asks something like:

> this requirement changed, what does it impact?

That question hides a lot of work:

1. identify the requirement object,
2. resolve linked components,
3. pull traceability data,
4. gather code, test, and defect evidence,
5. compare the old and new requirement,
6. summarize impact,
7. draft follow-up work,
8. ask for review before anything is written back.

So the capability is "Requirement Impact Analysis", but the platform execution is a chain of tasks.

```text
Requirement Impact Analysis
  -> parse_requirement_change
  -> resolve_entity
  -> retrieve_evidence
  -> compare_artifacts
  -> generate_structured_report
  -> draft_ticket
  -> request_review
```

That chain is where the requirement impact discussion became practical. The question was no longer "what is the abstraction?" but "what does each step need to know, and what is it allowed to do?"

### The workflow

We ended up treating requirement impact analysis as a published workflow, not a single task.

It has a fixed enough shape to be reused, but it still needs to adapt to the input. If the requirement is vague, the workflow has to ask for clarification. If the requirement is specific, it can move straight into evidence collection. If the change is sensitive, it has to stop before any write action.

That is why the workflow layer matters. It is the place where the case becomes a process.

### The tasks

The most reusable tasks were:

- `resolve_entity`
- `retrieve_evidence`
- `compare_artifacts`
- `generate_structured_report`
- `draft_ticket`

We tried to make these tasks feel generic enough to reuse, but not so generic that they became meaningless.

For example, `retrieve_evidence` is not just "fetch something". For this case, it has to know whether the evidence comes from Jira, code, test results, or defect history. That means the task needs a clear input contract, but the internal retrieval plan can still vary.

### The tools

The tool layer is where the platform becomes real. For this case, the typical tools were:

- requirement_read
- Jira_read
- git_diff or code_read
- test_result_read
- defect_read
- artifact_read

I think this is where the layered model paid off the most. We did not want every task to embed its own API code. The task should ask for evidence. The tool layer should know how to reach Jira, pipelines, documents, or repos safely.

### The context packages

The requirement case also made context packages feel less like a prompt trick and more like a data contract.

We used packages such as:

- requirement context
- traceability context
- code change context
- test evidence context
- defect context

Each package has to answer a boring but important question: what data belongs here, and what does not?

If a package only returns a blob of text, it is too weak. If it returns everything, it becomes noisy. The useful middle ground is structured enough to support downstream tasks, but not so broad that the model has to dig through junk.

### Requirement impact analysis is really about evidence

This was the core requirement impact lesson: the output is not just a summary. The output is a defended summary.

That means the report needs to say:

- what changed,
- what was found impacted,
- what evidence supports that,
- what is still uncertain,
- and what human review is needed next.

So the evaluation criteria changed too. We were not only asking whether the answer "sounds right". We cared about citation coverage, traceability coverage, false negatives, and whether the report stayed reviewable.

In practice, the requirement case became the best forcing function for requirement impact analysis because it exposed every missing primitive at once.

## The workflow definition changed a little

One thing we changed explicitly was the definition of `workflow`.

Earlier, I leaned too hard on "workflow = fixed template". That turned out to be too narrow.

After adoption started, we found that workflows need to cover both:

- deterministic execution paths,
- and partially adaptive orchestration.

That is why we now group `workflow` and `agent` together as the published orchestrator layer.

The difference is still useful:

- a workflow is more rule-driven,
- an agent is more reasoning-driven,
- but both are orchestrators,
- both can call tasks,
- and both must stay inside the same policy and audit boundaries.

This sounds like a small terminology change, but it removed a lot of accidental complexity. We no longer needed two separate governance stories.

## The other two cases

The other two cases were simpler, but they confirmed the same pattern.

### CI Failure Triage

Here the user wants to know why a pipeline failed and what to do next. The shape is similar, but the evidence sources are different.

The workflow usually needs:

- failure log retrieval,
- commit and diff lookup,
- test result analysis,
- owner suggestion,
- draft incident or ticket output.

It is a good case for the platform because it uses the same layered contract, but it proves that a capability should not be hardwired to one data source.

### Release Readiness

Release readiness is where the review boundary became visible.

The platform can collect evidence, summarize gaps, and draft a release note, but it should not cross into write actions without a deliberate approval step. That case reminded us that policy is not an afterthought. It is part of the primitive design.

## What we still have not solved

We still have one hard edge: nested orchestration.

Right now, I prefer keeping published workflow/agent composition simple. Internal executors can be as complex as needed, but the published graph should stay understandable. That keeps policy inheritance, auditing, and evaluation from turning into a maze.

So the current direction is:

- keep published primitives layered,
- keep workflow and agent in one orchestrator layer,
- keep tasks reusable,
- keep tools and context packages controlled,
- and keep the complex reasoning inside the implementation boundary when possible.

That is less elegant than the first draft, but it survives real adoption better.

## Where is skill

At some point we asked a different question: where does `skill` fit in this picture?

The short answer is that `skill` is a much broader concept, and it sits at a different abstraction level from the platform primitives above. A skill can bundle behavior, instructions, orchestration, and even implementation detail. So if we keep treating it as the same kind of primitive as workflow, agent, or task, the model gets blurry again.

The cleaner split is:

- **Published skill**: a reusable primitive that can be registered, governed, versioned, and reused across cases, similar to workflow, agent, or task.
- **Private skill**: an internal implementation asset that stays inside one team or one capability and does not enter platform governance.

That distinction mattered because not every skill should become a platform object. Some skills are really just local know-how wrapped for one use case. Others are stable enough to be shared, evaluated, and audited like the other published primitives.

So in practice, we treated skill as a broader umbrella, then drew a line inside it:

```text
Skill
  -> Published skill
  -> Private skill
```

The published skill belongs to the same governed world as workflow, agent, and task. The private skill does not. It can still be useful, but it stays out of the shared registry and out of the platform contract.

## Closing

So the story since the last post is not "the design was wrong". It is more like this: we had a good first model, then real adoption made the rough edges visible, and the platform got more honest.

The result is a little less neat, but much more useful. The primitives are layered. `workflow` and `agent` now live in the same published orchestrator layer. And the three cases, especially requirement impact analysis, forced us to define the contract around evidence, policy, and review instead of around nice words.

That feels closer to something a real platform team can ship.
