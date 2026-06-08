---
title: Paseo：在手机上指挥 AI 写代码是什么体验
description: 介绍 Paseo 这个开源的多供应商 AI coding agent 编排工具，从实际使用场景出发，说明它解决了什么问题、怎么工作的、以及还有哪些不足。
categories:
  - software
tags:
  - AI
  - Agent
  - Paseo
  - Claude Code
  - Codex
  - DevTools
  - OpenSource
---

去年底我开始同时用多个 AI coding agent。Claude Code 负责复杂的架构设计和 review，Codex 跑大批量的实现工作，偶尔还会让 Pi 试试不同的思路。它们各有各的长处，但管理起来很痛苦。

每个 agent 都跑在自己的终端窗口里。我 Mac 上常年开着四五个 terminal tab，每个里面可能还有一个 agent 在跑。切来切去，经常忘了哪个 agent 在干什么。更麻烦的是，我一离开电脑就完全不知道进度了——有时候 agent 跑着跑着遇到 permission 请求卡住了，我就白白等了半小时。

我需要的是一个能统一管理这些 agent 的地方。而且最好能从手机上操作。

试了几个方案之后，我开始用 [Paseo](https://paseo.sh)。

## Paseo 是什么

简单说，Paseo 是一个 self-hosted 的 agent 编排 daemon。它在你的机器上跑一个服务，然后你可以从 Mac 桌面 app、网页、或者手机上连接到它，统一管理所有 coding agent。

它支持的 agent 包括 Claude Code、Codex、OpenCode、Copilot、Pi，总共 30 多个。每个 agent 用各自的 native harness 运行——你的 subscription、skills、config、MCP server 都原样保留，Paseo 不做包装和修改。

核心理念一句话：**agent 在你自己的机器上跑，用你自己的开发环境，但你可以在任何地方操控它们。**

## 我实际怎么用

一个典型的场景是这样的。

我在 Mac 上开着 Paseo daemon。早上到公司，打开桌面 app，给 Claude Code 发一个任务："review 一下 feat/dashboard 分支的改动，重点看 auth 逻辑。"

Claude Code 开始跑的时候，我切到另一个 panel，同时给 Codex 派了一个实现任务："把 returns API 的错误处理改成统一的 error response 格式。"

两个 agent 并排跑着，我可以随时看它们的输出。Paseo 的 split panel 设计让这个体验很自然——左边是 agent 对话，右边可以是 terminal、diff、或者 browser preview。

然后我去开会了。

开会的时候掏出手机，打开 Paseo app，看到 Claude Code 那边卡在了一个 permission 请求上——它想写文件，需要我确认。我在手机上点了 approve，它继续跑。开完会回来，两边都跑完了，diff 已经在面板里等着 review。

<video src="/images/paseo.mp4" controls muted loop playsinline style="width:100%;max-width:400px;border-radius:12px;display:block;margin:1.5rem auto;"></video>

这个流程以前是不可想象的。以前我只能坐在电脑前等，或者离开之后焦虑地不知道发生了什么。

## 几个让我觉得"对了"的设计

用了一段时间，有几个细节让我觉得这个工具的设计者是真正自己用过这些东西的。

### Worktree 和 dev server

Paseo 对并行开发的支持做得很好。每个 agent 可以在自己的 git worktree 里工作，互不干扰。而且当你让 agent 跑 `npm run dev` 的时候，Paseo 会根据分支名自动分配一个 URL——`web.fix-auth.my-app.localhost`——不会出现"端口被占了"的问题。

这解决了多 agent 场景下最常见的摩擦。

### 不只是 UI，CLI 也很完整

我一开始以为这种"好看界面"的工具，CLI 会比较弱。但 Paseo 的 CLI 几乎可以做到 UI 能做的一切：

```bash
paseo run "implement user authentication"
paseo run --provider codex --worktree feature-x "implement feature X"
paseo ls          # 列出所有 running agent
paseo attach abc123  # 实时查看某个 agent 的输出
paseo send abc123 "also add tests"  # 给运行中的 agent 追加任务
```

还有 loop 和 schedule 功能——你可以设置定时任务，让 agent 定期检查某些东西。这在实际工作中很有用，比如每天早上自动跑一遍 lint 和 test。

### 手机端是真正的 native，不是 webview 凑的

这一点必须单独说。Paseo 的手机 app 是我用过的开发者工具类 app 里体验最好的之一。它有完整的功能 parity——不只是"查看"agent 状态，而是可以在手机上批准权限、发送新任务、查看 diff、甚至操作 terminal。

我有几次周末在外面，突然想起来有个小改动要做，掏出手机给 Claude Code 发了个 prompt，几分钟就搞定了。这在以前需要找电脑、连 VPN、开 terminal。

### E2E 加密 relay

远程访问有两种方式：直连（局域网或 Tailscale），或者通过 Paseo 的 end-to-end 加密 relay。relay 的好处是不需要自己搭 VPN 或暴露端口，而且代码不会经过 Paseo 的服务器——relay 只传输加密后的 agent 交互数据。

## 哪些地方还不够

诚实地说，有几个地方让我觉得还在 early stage。

**社区和生态还在早期。** 虽然支持 30+ agent，但一些比较小众的工具整合还不太稳定。GitHub 上的文档和示例也不够多，遇到问题主要靠试。

**权限系统的粒度可以更细。** 目前手机端的权限审批是好用的，但我想看到更细的规则——比如"来自某个 repo 的文件操作不需要审批"，或者"工作时间内的某些操作自动放行"。这应该是 roadmap 上的东西。

**语音交互目前还是偏 experimental。** 本地语音 stack 这个方向很好，但要真正在通勤或走路时"说代码"，体验还不够顺滑。准确率本身不是问题，问题在于代码相关的口语表达和自然语言之间还有很大的差距——你说"把那个 auth middleware refactor 一下"，它很难精确知道"那个"是指什么。

**价格是免费的，但可持续性是一个问号。** Paseo 目前完全免费开源，没有明确的商业模式。对于想在生产环境依赖它的团队来说，这是一个需要留意的点。不过好在你需要的是 agent 的 subscription（Claude Code、Codex 等），Paseo 本身只是一个编排层，即使哪天它变了，你的 agent 和配置都在本地。

## 和类似工具的对比

市面上有几个类似方向的产品，但 Paseo 的定位不太一样：

- **VS Code / Cursor 等 IDE 内 agent**：它们把 agent 嵌入编辑器，体验很顺，但你的 agent 被绑在了那个编辑器里。Paseo 的 agent 是独立的进程——你可以在终端里继续用 vim，让 agent 在后台跑。

- **各种 agent 平台（Devin、Factory 等）**：它们在云端跑 agent，有自己的沙箱。好处是不占本地资源，坏处是你的开发环境、配置、工具链全在本地，云端 agent 没法直接用。Paseo 让 agent 用你的真实环境。

- **直接开多个 terminal**：成本最低，但缺乏统一视图和远程操控能力。Paseo 解决的就是这个"管理"问题。

## 最后

Paseo 解决的是一个我真实遇到的问题：当你开始认真用多个 AI coding agent 的时候，怎么管理它们。它不是要替代任何一个 agent，而是给它们一个统一的操作面板。

我现在的工作流已经和 Paseo 绑得比较紧了。桌面 + 手机的切换体验让我不用一直守在电脑前，这点对生活质量的改善超过了我对它的预期。

下一步我比较期待的是更完善的 permissions 策略系统和更好的 agent 间协作——比如一个 agent 跑完后自动把结果交给另一个 agent review，这在现在的 CLI loop 里可以做，但还不够开箱即用。

如果你也在同时用多个 AI coding agent，或者想试试在手机上管理 agent，值得花半小时装一个试试。
