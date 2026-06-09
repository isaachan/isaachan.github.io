---
title: LangChain/LangGraph vs. CrewAI：企业落地时，差别到底在哪里
description: 从 Requirement Impact Analysis 这个 case 出发，比较 LangChain/LangGraph 与 CrewAI 在企业原型速度、状态管理、审批治理和长期维护上的取舍。
categories:
  - software
tags:
  - AI
  - Enterprise
  - Platform
  - LangChain
  - LangGraph
  - CrewAI
  - Agent
---

如果真要把 AI workflow 带进企业里，`LangChain/LangGraph` 和 `CrewAI` 到底该怎么选，这是一个绕不过去的问题。

这个问题不能空谈“框架选型原则”什么的，而是要带入到企业面对的具体问题。本文就以一个真实场景为例：

> **Requirement Impact Analysis (需求变更影响分析)**：当一条需求发生变更时，系统是否能够自动识别其影响范围，给出受影响的模块、测试项与潜在风险，并在必要时将结论停留在 review 边界内，供工程和业务团队进一步确认。

这类 case 表面上看像是“把流程拆成几个 task”就够了，但那只是入口。真正把两个框架拉开差距的，不是它们能不能把事情跑起来，而是它们在企业里怎么处理状态、审批、审计边界和后续维护。

下面就用两个框架实现同一个 case，看看会呈现出什么不同的“手感”来。

## 把场景拆解为可独立运行的 Task

先不要急着选框架。这个 case 的第一步，其实是先拆。

如果不拆，最自然的做法就是给一个 agent 几个工具，让它自己去读需求、找关联、看代码、看测试，最后输出一份影响分析报告。这个做法短期很省事，但会立刻带来三个问题。

第一，**边界不清楚**。结论不准时，很难判断到底是需求解析错了、traceability 没拉全，还是证据汇总本身出了偏差。

第二，**复用性很差**。今天做 Requirement Impact Analysis，明天做 CI failure triage，后天做 release readiness，很容易重新长出一套“大 agent + 一堆 prompt”。

第三，**review 边界卡不住**。分析和执行容易混在一起。尤其当流程开始草拟 ticket、更新测试建议甚至写回系统时，责任边界会立刻变敏感。

所以这个场景必须先拆成几个可独立运行的 Task。所谓“可独立运行”，不是说这些 Task 必须脱离上下文，而是说每个 Task 都应该有相对稳定的输入、输出和职责。

一个比较实用的拆法是：

```text
parse_change
-> resolve_requirement
-> retrieve_traceability
-> collect_code_and_test_evidence
-> summarize_impact
-> request_review
```

这里每一步都在回答一个更小、更稳定的问题：

- `parse_change`：这次变更到底改了什么
- `resolve_requirement`：它对应哪个 requirement object
- `retrieve_traceability`：它和哪些设计对象、模块、测试项有关
- `collect_code_and_test_evidence`：现有代码和测试里已经有哪些证据
- `summarize_impact`：哪些影响是高置信度，哪些只是风险提示
- `request_review`：哪些结论可以直接展示，哪些必须停在人审边界前

只有先把这一步拆清楚，后面再谈 LangGraph 还是 CrewAI 才有意义。否则讨论的不是框架，而只是“一个大 agent 应该怎么写 prompt”。

## LangChain / LangGraph：更像在搭一套系统

如果用 LangChain/LangGraph 来实现这个 case，通常会落到 LangGraph 这一层来定义 state、node 和 edge。实现重点不是“谁做事”，而是“状态怎么流动、哪些边是允许的”。

```python
from typing import TypedDict, List
from langgraph.graph import StateGraph, END

class ImpactState(TypedDict):
    change_text: str
    requirement_id: str | None
    traceability_links: List[str]
    evidence: List[dict]
    impact_summary: str | None
    review_required: bool

def parse_change(state: ImpactState) -> ImpactState: ...
def resolve_requirement(state: ImpactState) -> ImpactState: ...
def retrieve_traceability(state: ImpactState) -> ImpactState: ...
def collect_code_and_test_evidence(state: ImpactState) -> ImpactState: ...
def summarize_impact(state: ImpactState) -> ImpactState: ...
def request_review(state: ImpactState) -> ImpactState: ...

graph = StateGraph(ImpactState)
graph.add_node("parse_change", parse_change)
graph.add_node("resolve_requirement", resolve_requirement)
graph.add_node("retrieve_traceability", retrieve_traceability)
graph.add_node("collect_code_and_test_evidence", collect_code_and_test_evidence)
graph.add_node("summarize_impact", summarize_impact)
graph.add_node("request_review", request_review)

graph.add_edge("parse_change", "resolve_requirement")
graph.add_edge("resolve_requirement", "retrieve_traceability")
graph.add_edge("retrieve_traceability", "collect_code_and_test_evidence")
graph.add_edge("collect_code_and_test_evidence", "summarize_impact")
graph.add_conditional_edges(
    "summarize_impact",
    lambda state: "request_review" if state["review_required"] else END,
)
```

如果再把某一个具体 Task 写开一点，比如 `retrieve_traceability`，LangGraph 的实现通常会把输入输出直接收进 state：

```python
def retrieve_traceability(state: ImpactState) -> ImpactState:
    requirement_id = state["requirement_id"]
    links = trace_repo.find_links(requirement_id)
    return {
        **state,
        "traceability_links": [link.target_id for link in links],
        "review_required": state["review_required"] or not links,
    }
```

这种实现方式的特点也很明确：

- state 是显式的
- node 边界天然稳定
- branch / pause / review gate 更容易直接放进流程图里

它更像在搭一套系统，而不是简单串一组 Task。代价也很明显：代码和设计都会更重，团队需要更早回答 state、gate、resume 这些结构问题。

## CrewAI：很适合把事情先跑起来

把上面的 Task 落到 CrewAI，写法通常会比较直白。一个 Crew 管整条流程，几个 Task 按顺序执行，不同 agent 负责不同类型的判断。

```python
from crewai import Agent, Crew, Process, Task

requirement_analyst = Agent(
    role="Requirement Analyst",
    goal="Understand the change and resolve the requirement object",
)

traceability_analyst = Agent(
    role="Traceability Analyst",
    goal="Find linked modules, interfaces and test assets",
)

impact_reviewer = Agent(
    role="Impact Reviewer",
    goal="Summarize impact with evidence and stop at review boundary",
)

tasks = [
    Task(description="Parse the requirement change", agent=requirement_analyst),
    Task(description="Resolve the target requirement object", agent=requirement_analyst),
    Task(description="Retrieve traceability links", agent=traceability_analyst),
    Task(description="Collect code and test evidence", agent=traceability_analyst),
    Task(description="Summarize impact and risks", agent=impact_reviewer),
    Task(description="Prepare review-ready output", agent=impact_reviewer),
]

crew = Crew(
    agents=[requirement_analyst, traceability_analyst, impact_reviewer],
    tasks=tasks,
    process=Process.sequential,
)
```

如果把其中一个 Task 再写具体一点，CrewAI 通常会把行为约束放进 `description` 和 `expected_output`：

```python
retrieve_traceability_task = Task(
    description="""
    Read the resolved requirement object and retrieve linked modules,
    interfaces and test assets from the traceability system.
    Mark missing links explicitly instead of guessing.
    """,
    expected_output="""
    A structured list of linked artifacts, plus a note explaining
    whether traceability coverage is complete or partial.
    """,
    agent=traceability_analyst,
)
```

这个实现方式的好处是非常直接：

- Task 列表清楚
- agent 分工容易理解
- 第一版流程很快就能跑起来

如果只是在试点阶段，需要把一条流程快速串起来，CrewAI 会非常顺手。它的代码重心就是“谁按什么顺序做哪些 Task”，这也是它最容易被团队接受的地方。

但这个实现方式也有一个明显特点：状态、分支、暂停、恢复这些东西，并不是最先被强调的部分。后面如果要继续往里加，往往要靠更多约定来补。

## 把 Requirement Impact Analysis 放进去看，判断会更具体

这个 case 很适合拿来做框架对比，因为它天然同时包含了几类要求：

- 要做信息解析
- 要做跨系统检索
- 要汇总证据
- 要输出结论
- 还要停在 review 边界前

也正因为这样，它不会只测试框架“能不能把流程串起来”，还会测试框架“怎么处理流程责任”。同样一条链路，LangGraph 更容易把后续的治理需求写进结构里，CrewAI 更容易把第一版做出来。

## 实现对比汇总

如果把上面的判断压成一张表，大概是这样：

| 维度 | LangChain / LangGraph | CrewAI |
| --- | --- | --- |
| 第一版原型速度 | 中等 | 快 |
| 给团队讲清楚的难度 | 中等 | 低 |
| 线性流程表达 | 也能做，但不一定更省 | 顺手 |
| 代码实现重心 | state / node / edge 显式建模 | agent / task 顺序编排 |
| 显式状态管理 | 很强，是核心能力之一 | 可做，但不是主舞台 |
| 分支 / 暂停 / 恢复 | 更自然 | 能支持，但复杂后会逐渐变重 |
| 审批 / review gate | 更容易成为流程设计的一部分 | 需要自己补较多结构 |
| 审计和证据追溯 | 更容易前置到模型里 | 依赖额外约束 |
| 长期维护 | 更适合复杂流程长期演化 | 流程复杂后容易堆隐式约定 |

如果只用一句话总结：

**LangGraph 更像一个更稳的企业 workflow 底座，CrewAI 更像一个很好的试点框架。**

这比简单讨论“谁更强”更接近真实情况。

## 为什么一进企业环境，差别就开始放大

企业里的问题从来不只是“流程能不能跑通”。真正会把差别放大的，是下面这些要求：

- 哪些数据是可以读的，哪些系统是不能碰的
- 哪个步骤只是分析，哪个步骤已经接近执行
- 中间结论能不能追溯回证据
- 流程被打断后，能不能从中间恢复
- 某个节点是不是必须经过 human review
- 下个月流程改了，另一个团队能不能看懂并安全地改

你会发现，这些问题跟模型好不好、prompt 漂不漂亮，关系其实没那么大。它们更像 workflow 系统本身的问题。

**状态**是第一个分界点。只要流程需要显式记录 requirement object、evidence completeness、risk level、review status，LangGraph 的优势就会开始放大。

**审计**是第二个分界点。企业里的 impact analysis 很少只要一个“看起来合理”的答案，通常还要回答：这条结论依据了哪些证据，哪些结论只是风险提示，哪些地方必须回到人审。

**恢复能力**是第三个分界点。流程一旦跨系统运行，就很难保证一步走到底。能不能从 evidence collected 继续，而不是整条重跑，直接影响稳定性和成本。

**长期维护**是最后一个分界点。流程复杂度一旦上来，隐式约定会越来越贵。谁更容易把边界写进结构里，谁就更容易被团队长期接住。

## 企业里更适合怎么选

如果一个团队问的是：

- 这套东西半年后会不会越来越难改
- 审批和权限边界怎么稳定下来
- 出问题时怎么知道是哪一段错了
- 以后别的团队接手，能不能看懂和复用

那更值得优先考虑的是 LangChain/LangGraph，尤其是 LangGraph。

如果一个团队还在验证阶段，目标是：

- 先把一个跨系统 AI workflow 跑通
- 先让业务和工程看到它确实能工作
- 先把流程轮廓讲清楚

那先用 CrewAI 往往是更务实的路径。

所以如果一定要给一个明确结论，可以归纳成这样：

- **平台化、长期化阶段**：LangChain/LangGraph 更稳，更适合承接企业流程
- **探索期、试点期**：CrewAI 更轻，更容易形成第一轮 adoption

## 还有哪些问题，其实不是框架能解决的

不过有一点需要单独强调：框架选型当然重要，但它经常会被高估。

企业里的 AI workflow 最后能不能落住，很多时候不取决于你选了 LangGraph 还是 CrewAI，而取决于你有没有把下面这些事做实：

- 哪些数据能进流程，哪些不能
- 哪些输出只是建议，哪些已经接近执行
- review 边界画在哪里
- 失败后怎么恢复
- 证据怎么存，责任怎么追

这些东西如果本身是糊的，换哪个框架都只是换一种方式把模糊包起来。

所以对这个问题更准确的回答是：

LangChain/LangGraph 和 CrewAI 都能做出像样的 AI workflow，但它们适合承载的阶段不一样。LangGraph 更适合把事情长期接住，CrewAI 更适合把事情启动起来。

如果要问“企业落地到底更该看什么”，重点不该放在 demo 漂不漂亮，而该放在这个框架在状态、审批、审计和长期维护上，愿不愿意帮团队把真实复杂度正面摊开。

因为企业里最贵的，从来不是第一版跑起来，而是它跑起来之后，能不能一直被安全地改下去。
