---
title: Refined Primitives List for 3 Core Use Cases
description: 在 AI platform adoption 之后，整理 Requirement Impact Analysis、CI Failure Triage、Release Evidence Summarization 的 workflow、task、context、tool、policy 全量清单。
categories:
  - software
tags:
  - AI
  - Platform
  - Workflow
  - Task
---

上次我写 AI platform adoption 时，更多在讲思路和边界。那之后我们一直在细化实现细节，讨论也从“抽象对不对”变成“这套东西给团队用时是否清楚、可复用、可治理”。

所以这篇不再谈原则，直接给一份可落地的 full list，围绕三个 use case：Requirement Impact Analysis、CI Failure Triage、Release Evidence Summarization。

## 1. Workflow

### 1.1 Workflow: Requirement Impact Analysis

```text
workflow_id: requirement_impact_analysis_workflow
```

步骤：

1. `resolve_entity`  
   输入需求 ID / Jira ID / CR ID。  
   解析项目、版本、需求对象。
2. `parse_requirement_change`  
   提取需求变更点、功能范围、约束条件。
3. `trace_requirement_links`  
   检索关联需求、上游需求、下游需求、测试用例、缺陷。
4. `identify_impacted_components`  
   识别受影响组件、服务、ECU、代码仓库、owner。
5. `retrieve_evidence`  
   拉取代码变更、历史缺陷、测试结果、发布历史。
6. `assess_change_impact`  
   判断影响范围、风险等级、测试建议。
7. `generate_structured_report`  
   生成影响分析报告。
8. `draft_ticket_or_report`  
   生成 draft ticket / draft impact report。
9. `request_human_review`  
   等待工程师确认。

输出结构：

```text
- Summary
- Requirement change interpretation
- Impacted components
- Related requirements
- Related tests
- Related defects
- Risk assessment
- Recommended actions
- Evidence links
- Draft ticket/report
```

### 1.2 Workflow: CI Failure Triage

```text
workflow_id: ci_failure_triage_workflow
```

步骤：

1. `resolve_entity`  
   解析 pipeline ID、project、branch、commit。
2. `parse_ci_failure`  
   解析失败阶段、失败测试、关键错误日志。
3. `find_related_commits`  
   查找最近提交、merge request、代码 owner。
4. `match_known_failures`  
   查找历史相似失败、已知缺陷、flaky test 记录。
5. `retrieve_evidence`  
   拉取 CI log、测试历史、环境元数据、Jira 缺陷。
6. `diagnose_failure_cause`  
   生成可能原因和置信度。
7. `suggest_owner`  
   建议 owner / team。
8. `generate_structured_report`  
   生成分诊摘要。
9. `draft_ticket_or_report`  
   可选：生成 draft Jira ticket。
10. `request_human_review`  
    用户确认是否提交 ticket / 分派 owner。

输出结构：

```text
- Failure summary
- Failed stage/test
- Key log excerpts
- Related commits
- Similar historical failures
- Known defects
- Suspected root cause
- Confidence level
- Suggested owner
- Recommended next action
- Evidence links
```

### 1.3 Workflow: Release Evidence Summarization

```text
workflow_id: release_evidence_summarization_workflow
```

步骤：

1. `resolve_entity`  
   解析 project、version、release candidate。
2. `collect_release_evidence`  
   收集需求、Jira、测试、缺陷、风险、waiver、release note。
3. `check_release_gaps`  
   检查未完成需求、失败测试、阻塞缺陷、缺失证据。
4. `summarize_evidence`  
   汇总核心发布证据。
5. `identify_risks`  
   识别发布风险、质量风险、流程风险。
6. `summarize_release_readiness`  
   判断发布就绪度。
7. `generate_release_recommendation`  
   生成发布建议。
8. `generate_structured_report`  
   生成 release readiness report。
9. `draft_ticket_or_report`  
   生成发布总结草稿。
10. `request_human_review`  
    进入发布评审。

输出结构：

```text
- Release summary
- Scope
- Requirement status
- Test status
- Defect status
- Known risks
- Waivers/exceptions
- Missing evidence
- Readiness judgment
- Recommendation
- Evidence links
```

## 2. Task

| id | name | description（用途） | used by |
| --- | --- | --- | --- |
| resolve_entity | 实体解析 | 把用户输入中的项目名、需求 ID、pipeline ID、版本号解析成系统实体 | all |
| retrieve_evidence | 证据检索 | 从上下文包和工具中取回相关证据 | all |
| summarize_evidence | 证据总结 | 把多来源证据整理成结构化摘要 | all |
| identify_risks | 风险识别 | 从证据中识别风险、缺口、阻塞项 | all |
| suggest_owner | Owner 建议 | 根据组件、提交、历史记录建议负责人 | Requirement Impact Analysis, CI Failure Triage |
| generate_structured_report | 生成结构化报告 | 按模板输出可审阅报告 | all |
| draft_ticket_or_report | 生成草稿产物 | 创建 Jira 草稿、发布报告草稿或评审草稿 | all |
| request_human_review | 请求人工复核 | 把结果交给人确认，不自动生效 | all |
| parse_requirement_change | 解析需求变更 | 提取需求目标、变更点、影响对象 | Requirement Impact Analysis |
| trace_requirement_links | 追踪需求链路 | 找关联需求、架构组件、测试用例、缺陷 | Requirement Impact Analysis |
| identify_impacted_components | 识别受影响组件 | 推断受影响系统 / ECU / 服务 / 模块 | Requirement Impact Analysis |
| assess_change_impact | 评估变更影响 | 生成影响范围和风险说明 | Requirement Impact Analysis |
| parse_ci_failure | 解析 CI 失败 | 提取失败阶段、失败测试、错误日志、环境信息 | CI Failure Triage |
| find_related_commits | 查找相关提交 | 找到近期可能相关的 commit / merge request | CI Failure Triage |
| match_known_failures | 匹配历史失败 | 查找类似失败、已知缺陷、重复问题 | CI Failure Triage |
| diagnose_failure_cause | 诊断失败原因 | 生成可能 root cause 和置信度 | CI Failure Triage |
| collect_release_evidence | 收集发布证据 | 聚合需求、测试、缺陷、风险、waiver | Release Evidence Summarization |
| check_release_gaps | 检查发布缺口 | 找未完成需求、失败测试、阻塞缺陷 | Release Evidence Summarization |
| summarize_release_readiness | 总结发布就绪度 | 生成 ready / not ready / conditional ready 判断 | Release Evidence Summarization |
| generate_release_recommendation | 生成发布建议 | 给出发布、延期、带风险发布建议 | Release Evidence Summarization |

## 3. Context Package

![Entities summary](/images/entities-summary.svg)

### 3.1 Context Package: Traceability

```text
context_package_id: traceability_context
```

作用：支持需求影响分析，也支持发布证据总结。

Including data:

- requirement items
- requirement links
- architecture components
- component ownership
- test case links
- Jira issue links
- release mapping

Entities:

- requirement_id
- project_id
- component_id
- test_case_id
- issue_id
- release_version
- owner

### 3.2 Context Package: Code Change

```text
context_package_id: code_change_context
```

作用：支持需求影响分析和 CI 失败分诊。

Including data:

- repositories
- commits
- merge requests
- changed files
- code owners
- branch information
- recent change history

Entities:

- repo_id
- branch
- commit_hash
- merge_request_id
- file_path
- component_id
- owner

### 3.3 Context Package: CI/Test

```text
context_package_id: ci_test_context
```

作用：支持 CI 失败分诊和发布证据总结。

Including data:

- pipeline runs
- build status
- test results
- failed test logs
- flaky test history
- test environment metadata
- test coverage summary

Entities:

- pipeline_id
- build_id
- test_run_id
- test_case_id
- commit_hash
- environment_id
- status

### 3.4 Context Package: Defect/Risk

```text
context_package_id: defect_risk_context
```

作用：三个 Capability 都需要。

Including data:

- Jira issues
- known defects
- bug history
- risk items
- blocking issues
- severity / priority
- issue status
- linked requirement / component / release

Entities:

- issue_id
- project_id
- component_id
- requirement_id
- release_version
- severity
- priority
- status
- owner

### 3.5 Context Package: Release

```text
context_package_id: release_context
```

作用：主要支持发布证据总结，也支持需求影响分析。

Including data:

- release plan
- release scope
- version mapping
- release candidate
- release notes
- waiver / exception records
- go / no-go decision history

Entities:

- project_id
- release_version
- release_candidate_id
- requirement_id
- issue_id
- waiver_id
- approval_status

## 4. Tool

| id | category | name | description |
| --- | --- | --- | --- |
| requirement_read | read-only | 需求系统读取工具 | 读取需求、变更请求、需求链路 |
| jira_read | read-only | Jira 读取工具 | 读取 issue、缺陷、风险、状态、owner |
| git_read | read-only | Git 读取工具 | 读取 commit、MR、changed files、branch |
| ci_read | read-only | CI 读取工具 | 读取 pipeline、build、stage、job 状态 |
| test_result_read | read-only | 测试结果读取工具 | 读取测试用例、测试结果、失败日志 |
| release_read | read-only | 发布系统读取工具 | 读取版本计划、release scope、RC 信息 |
| ownership_read | read-only | Owner 读取工具 | 查询组件、代码、项目 owner |
| artifact_read | read-only | 文档 / 报告读取工具 | 读取架构文档、release note、测试报告 |
| jira_draft_ticket | draft | Jira 草稿工具 | 生成但不提交 ticket |
| report_draft_create | draft | 报告草稿工具 | 生成 impact report / release summary draft |
| review_request_create | draft | 评审请求草稿工具 | 创建 review request，但需要人工确认 |
| jira_create_ticket | action-taking | Jira 创建工具 | 直接创建 Jira ticket（高风险，MVP 不建议启用） |
| assign_owner | action-taking | Owner 分派工具 | 自动分派 owner（组织影响大，MVP 不建议启用） |
| trigger_ci | action-taking | CI 触发工具 | 触发 CI（有资源成本和误操作风险） |
| update_release_status | action-taking | 发布状态更新工具 | 修改发布状态（高风险，必须审批） |
| update_requirement | action-taking | 需求更新工具 | 修改需求（高风险，必须审批） |

## 5. Policy

### 5.1 access_control_policy

```text
policy_id: access_control_policy
```

Description：用户只能访问自己有权限的数据。

Rules:

- 所有 context retrieval 必须经过权限检查
- 工具调用继承用户身份或服务账号 + 用户授权
- 不允许跨项目读取无权限数据
- 输出不得泄露无权限来源

Applied_to：all。

### 5.2 read_only_by_default_policy

```text
policy_id: read_only_by_default_policy
```

Description：默认所有工具只读，写操作只能生成草稿。

Rules:

- AI 不允许直接修改需求、代码、测试、发布状态
- AI 不允许直接分派 owner
- AI 只允许生成 draft ticket / draft report
- 草稿必须由人确认后提交

Applied_to：all。

### 5.3 source_citation_required_policy

```text
policy_id: source_citation_required_policy
```

Description：重要结论必须有证据来源。

Rules:

- 风险判断必须引用 Jira / test / requirement / commit / release evidence
- root cause 判断必须引用日志、提交、历史缺陷等证据
- 发布建议必须引用测试、缺陷、需求覆盖和 waiver 证据
- 没有证据时必须标记为 assumption / hypothesis

Applied_to：all。

### 5.4 human_review_required_policy

```text
policy_id: human_review_required_policy
```

Description：AI 输出不能自动成为正式工程结论。

Rules:

- impact analysis 必须人工确认
- CI root cause 只是 suggestion
- release readiness recommendation 必须人工审批
- draft ticket / draft report 不自动提交

Applied_to：all（尤其 Release Evidence Summarization）。

### 5.5 safety_criticality_policy

```text
policy_id: safety_criticality_policy
```

Description：安全相关内容提升控制级别。

Rules:

- 如果 requirement / issue / component 带 safety label，自动升级为 stricter review
- 涉及 ADAS、braking、steering、powertrain、cybersecurity 等组件时提升风险等级
- 输出必须标明 safety relevance
- 不允许 AI 给出未经审核的 safety conclusion

Applied_to：Requirement Impact Analysis、Release Evidence Summarization、部分 CI Failure Triage。

### 5.6 audit_logging_policy

```text
policy_id: audit_logging_policy
```

Description：记录 AI 执行过程并可追溯。

Rules:

- 记录 user
- 记录 capability
- 记录 input references
- 记录 context packages used
- 记录 tool calls
- 记录 model route
- 记录 generated output
- 记录 citations
- 记录 policy decisions
- 记录 review result

Applied_to：all。

这轮把细节继续收敛后，我自己的体感是：平台真正的“稳定性”来自这些清单的边界一致性，而不是某个单点模型能力。只要这五层 primitive 对齐，后续 capability 再扩展，复杂度也还能被团队接住。
